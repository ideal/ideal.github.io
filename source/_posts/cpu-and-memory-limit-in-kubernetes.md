---
title: kubernetes中的容器cpu和内存限制
date: 2019-07-01 15:28:48
tags:
    - kubernetes
---

K8S部署容器时可以[指定CPU和内存的资源限制](https://kubernetes.io/zh/docs/tasks/administer-cluster/cpu-memory-limit/)，其真正实现也是通过CGroups做到的。我们来看个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources:
      limits:
        cpu: 500m
        memory: 400Mi
      requests:
        cpu: 200m
        memory: 200Mi
```

<!-- more -->

limits表示指定资源的上限，requests表示创建容器时初始的资源请求。cpu的单位m表示1/1000核，比如这里的500m，其实表示0.5核心。如果不带单位，比如1或者1.0，则表示分配1核cpu。内存的单位可以是Mi、Gi，分别表示MB和GB。这里的设置就表示初始时cpu 0.2核，内存200M；上限cpu 0.5核，内存400M。

首先来看cpu。

```bash
$ kubectl get pod nginx -o=jsonpath='{.spec.containers[0].resources}'
map[limits:map[cpu:500m memory:400Mi] requests:map[cpu:200m memory:200Mi]]
```

可以看到确实设置了初始和上限。

```bash
$ docker ps|grep nginx|grep -v pause|cut -d' ' -f1
7ac3aef786af

$ docker inspect 7ac3aef786af --format '{{.HostConfig.CpuShares}} {{.HostConfig.CpuQuota}} {{.HostConfig.CpuPeriod}}'
204 50000 100000
```

我们看到这里返回CpuShares是204，而不是200。这是因为K8S以1000为单位，而Docker以1024为单位。0.2\*1024 =204.8。

```bash
$ ps -ef|grep nginx
docker    5522  5279  0 08:34 pts/0    00:00:00 grep nginx
root     23814 23796  0 08:01 ?        00:00:00 nginx: master process nginx -g daemon off;
101      23833 23814  0 08:01 ?        00:00:00 nginx: worker process

$ cat /proc/23814/cgroup |grep cpuacct
8:cpu,cpuacct:/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971

$ cat cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971/cpu.shares
204
```

最终我们在`/sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971/cpu.shares`找到了这个初始值。

另外，HostConfig.CpuPeriod对应cpu.cfs_period_us，HostConfig.CpuQuota对应cpu.cfs_quota_us。

```bash
$ cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971/cpu.cfs_period_us
100000

$ cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971/cpu.cfs_quota_us
50000
```

这两个需要这样来换算。首先将单个核心cpu的带宽定义为0.1秒，也就是100000微秒。我们前面指定了上限500m，那么就是0.5\*100000 = 5000微秒。这里的cpu.cfs_period_us也就是100000微秒。

再来看内存的部分。

```bash
$ docker inspect 7ac3aef786af --format '{{.HostConfig.Memory}}'
419430400
```

也就是我们设置的内存上限。

```bash
$ cat /proc/23814/cgroup |grep memory
6:memory:/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971

$ cat /sys/fs/cgroup/memory/kubepods/burstable/pod6500174f-9bd6-11e9-9708-08002716449c/7ac3aef786af10a26528a0f805933ebfc5749223a2ed44f3e7fe49f20aa9b971/memory.limit_in_bytes
419430400
```

可以看到和CGroups中对应的memory.limit_in_bytes是相同的，也就是内存限制通过CGroups实现的方式。

