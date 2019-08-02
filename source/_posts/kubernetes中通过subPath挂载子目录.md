---
title: kubernetes中通过subPath挂载子目录
date: 2019-08-02 14:14:20
tags:
    - kubernetes
---

kubernetes支持很多种[Volume](https://kubernetes.io/docs/concepts/storage/volumes/)，比如NFS，甚至AWS的EBS，阿里云的OSS等。某些时候，我们可能需要针对不同的容器，挂载一个存储卷的不同子目录，以达到控制权限等目的。比如将用户数据放在一个NFS文件系统上，不同的用户区分不同的子目录。如果每个用户的容器，都直接挂载了整个NFS，同时又不能封禁容器的外网访问，那么这样是不安全的，如果被[反弹Shell](https://www.cnblogs.com/ssooking/p/5900664.html)之后，会存在安全隐患。我们可以通过subPath来解决这个问题。以hostPath类型的Volume为例，假设我们需要挂载/srv/aaa/bbb这个目录到/mnt/aaa/bbb，以及/srv/ccc/ddd目录到/mnt/ccc/ddd，而/srv下面的其他目录和文件，希望在容器中不可见，那么这样就可以做到：

<!-- more -->

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /mnt/aaa/bbb
      name: volume-test
      subPath: aaa/bbb
    - mountPath: /mnt/ccc/ddd
      name: volume-test
      subPath: ccc/ddd
  volumes:
    - name: volume-test
      hostPath:
        path: /srv
```

启动容器之后，我们通过`kubectl exec -it nginx /bin/bash`连进去查看是否达到我们想要的效果。

![/mnt in container](/images/mnt-in-container.png)

而在节点上的/srv是这样的：

![/srv in node](/images/srv-in-node.png)

确实实现了只挂载特定子目录。
