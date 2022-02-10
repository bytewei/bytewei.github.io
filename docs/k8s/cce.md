
### 1. 华为云机房不支持某些硬盘格式问题。
> 如：上海二区不支持SATA了，需要在yaml文件中指定使用其他格式


```
annotations:
        everest.io/disk-volume-type: SSD
```

同时指定storageclass：

```
storageClassName: csi-disk
```



### 2. 块存储挂盘问题（这其实不是问题，也不是CCE的问题，在此记录）。
> 给mysql5.7挂载云硬盘（块存储）时，该存储会被格式化并在存储目录中添加“lost+found”目录，mysql5.7会认为该存储有数据而终止初始化，导致mysql无法部署成功，解决办法是：在spec.containers中添加忽略“lost+found”的参数：
 
```
spec:
      containers:
      - args:
        - --ignore-db-dir=lost+found
```
