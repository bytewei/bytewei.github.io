### 背景
> cicd配置文件写的越来越多，越来越臃肿了，今天想优化，将公共配置抽象出来，没想到在简化kustomize配置时遇到了ArgoCD App无法生成的问题，花了很长时间最终定位出原来是测试环境集群的kubeconfig没有配置导致，在此做个记录。

### CICD架构简介
> 通过gitlab-ci.yml配置文件定义ci、cd全链路，目前已经将共性的配置独立出去了（如：maven构建、sonar扫描、argocd app的创建等）测试、线上环境各一套k8s集群，ArgoCD部署在测试环境k8s集群中，该ArgoCD同时管理测试与线上k8s集群。


==问题== 

今天遇到的问题是：测试环境的ArgoCD app可以自动生成，线上环境的ArgoCD app不能生成，并且报了下面错误：

```
# 执行命令 kustomize build | kubectl apply -f -
# 报了下面错误
error: json: cannot unmarshal object into Go struct field Kustomization.patchesStrategicMerge of type patch.StrategicMerge
```

```
# 以及下面报错
error: unable to recognize "STDIN": no matches for kind "Application" in version "argoproj.io/v1alpha1"
```

最开始以为是ArgoCD的bug，在谷歌上搜索了问题，查到了一个类似的报错，不过那个bug是ArgoCD1.2版本的，我们用的是2.1版本的，该bug早已修复，所以排除了ArgoCD的报错；后来怀疑是kustomize漏洞，在电脑上模拟执行kustomize命令可以生成ArgoCD app文件，并且kustomize目前已经很成熟了不可能会有这种bug，排除了kustomize问题。

最后我把整个yaml文件逐行读了一遍，发现生成线上k8s的ArgoCD App的yaml用的是线上环境k8s的KUBECONFIG文件，然而ArgoCD是部署在测试环境的k8s集群上，这就导致kubectl连接不了测试环境k8s集群，也就生成不了线上k8s的ArgoCD的App。

yaml配置文件如下：

```
# java-deployment.yml
# template
# 下面带“#”的“echo $IN_KUBE_CONFIG ...”一行是之前没有加的，这会导致KUBECONFIG使用错误。

.deploy:
  image: ${DEV_REGISTRY_HOST}/basic/deploy-tools:v2.0
  variables:
    GIT_STRATEGY: none
    ENV_NAME: dev-xx # replace
    CD_GIT_PATH: dev # replace
    K8S_CLUSTER: https://xxx # replace: $INSIDE_K8S_CLUSTER or $CCE2
    APP_CONFIG: "xxx" # repalce
    CD_GIT_BRANCH: $ENV_NAME
    K8S_NS: ${BUSINESS_NAME}-${ENV_NAME}
    IN_HARBOR_SECRET: xxx-xx.yaml
    PROD_HARBOR_SECRET: xxx-xxx.yaml
    IN_KUBE_CONFIG: $kube_xxx
    PROD_KUBE_CONFIG: $kube_yyy
  tags:
    - k8s
  before_script:
    - git init
    - git config --global user.email "gitlab@git.k8s.local"
    - git config --global user.name "GitLab CI/CD"
  script:
    - git clone https://${CI_USERNAME}:${CI_PASSWORD}@$CD_REPO
    - cd $APP_NAME
    - git checkout -B $CD_GIT_BRANCH
    - sed -ri "s/Tag[^*]*/Tag    kube.${ENV_NAME}.${APP_NAME}/" base/fluent-bit-sidecar-config.yaml
    - sed -i "s@APP_CONFIG@${APP_CONFIG}@g" base/jvm_opts_patch.yaml
    - sed -i "s@NAMESERVERS@${DNS}@g" base/custom_env_patch.yaml
    - cd overlays/$CD_GIT_PATH
    - >
      if [ "$CD_GIT_PATH" == "pre" ] || [ "$CD_GIT_PATH" == "prod" ]; then
        kustomize edit set image registry=${PROD_REGISTRY_HOST}/$REGISTRY_GROUP/$REGISTRY_IMAGE
        kustomize edit set image registry-fluent=${PROD_REGISTRY_HOST}/public/efk/fluent/fluent-bit:latest
      else
        kustomize edit set image registry=${DEV_REGISTRY_HOST}/$REGISTRY_GROUP/$REGISTRY_IMAGE
        kustomize edit set image registry-fluent=${DEV_REGISTRY_HOST}/public/efk/fluent/fluent-bit:latest
      fi
    - git add .
    - git commit -am "[skip ci] image update tag:${CI_PIPELINE_ID}-${CI_COMMIT_SHORT_SHA}(pipeline-hash)" || /bin/true
    - git push origin $CD_GIT_BRANCH -f
  after_script:
    - echo "ARGO_PROJ:$ARGO_PROJ"
    - git clone https://${CI_USERNAME}:${CI_PASSWORD}@git.xx.cn/deployments/base.git
    - git checkout -B master
    - >
      if [ "$CD_GIT_PATH" == "pre" ] || [ "$CD_GIT_PATH" == "prod" ]; then
        echo $PROD_KUBE_CONFIG |base64 -d > $KUBECONFIG && export KUBECONFIG=$KUBECONFIG
        kubectl create namespace $K8S_NS || /bin/true
        kubectl apply -f base/java-base/$PROD_HARBOR_SECRET -n $K8S_NS
#         echo $IN_KUBE_CONFIG |base64 -d > $KUBECONFIG && export KUBECONFIG=$KUBECONFIG
      else
        echo $IN_KUBE_CONFIG |base64 -d > $KUBECONFIG && export KUBECONFIG=$KUBECONFIG
        kubectl create namespace $K8S_NS || /bin/true
        kubectl apply -f base/java-base/$IN_HARBOR_SECRET -n $K8S_NS
      fi
    - |
      cat <<EOF >./kustomization.yaml
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
        - base/java-base/argo-application.yaml
      patches:
      - patch: |
          - op: replace
            path: /metadata/name
            value: ${APP_NAME}-${ENV_NAME}
          - op: replace
            path: /metadata/labels/app
            value: ${APP_NAME}
          - op: replace
            path: /spec/project
            value: $ARGO_PROJ
          - op: replace
            path: /spec/source/repoURL
            value: https://$CD_REPO
          - op: replace
            path: /spec/source/targetRevision
            value: $CD_GIT_BRANCH
          - op: replace
            path: /spec/source/path
            value: overlays/$CD_GIT_PATH
          - op: replace
            path: /spec/destination/server
            value: $K8S_CLUSTER
          - op: replace
            path: /spec/destination/namespace
            value: $K8S_NS
        target:
          kind: Application
      EOF
    - kustomize build | kubectl apply -f -
    - >
      if [ "$CD_GIT_PATH" == "dev" ] || [ "$CD_GIT_PATH" == "testing" ]; then
        echo y |argocd login $ARGOCD_IP --username $ARGCD_USER --password $ARGOCD_PASSWD --insecure
        argocd app sync ${APP_NAME}-${ENV_NAME}
      fi

```

### 几点思考
==事情本身的思考==

1. 问题虽小却也花了不少时间在排查上，造成这个问题的原因是配置文件太乱，这是之前我为了赶进度给自己挖的坑。
2. 配置缺少注释，这也给定位问题带来了困难。


==配置文件的思考==

1. 配置文件较多，发版配置很多：开发与测试环境用1个配置，uat环境用1各配置，预发与生产环境用1个配置，各环境大部分配置相同，但恰好是那些少部分不同的配置给我今天的优化埋了坑。
2. 上面的yaml文件仍有地方需要优化，还可以将共性部分抽象成公共部分。
3. 即使用了kustomize工具管理配置，cicd全链路的配置仍然很多很繁琐，希望通过引入OAM（kubevela）平台降低配置。


==其他收获==

1. gitlab-ci.yml是支持比较复杂的shell脚本写法的，如：

```
- >
      if [ "$CD_GIT_PATH" == "dev" ] || [ "$CD_GIT_PATH" == "testing" ]; then
        echo y |argocd login $ARGOCD_IP --username $ARGCD_USER --password $ARGOCD_PASSWD --insecure
        argocd app sync ${APP_NAME}-${ENV_NAME}
      fi
```
2. 当执行ArgoCD sync（通过argocd cli）触发了下面错误时有可能是k8s node节点时间不对（今天遇到的下面问题就是服务器时间不对）：

```
time="2022-02-11T10:25:18Z" level=fatal msg="rpc error: code = Unauthenticated desc = invalid session: token is not valid yet; wait 4m30.513298563s"
```


