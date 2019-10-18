---
title: access multiple clusters with kuberneter context
date: 2019-10-18 16:02:59
tags:
	- kubernetes
---

有时候往往需要和多个Kubernetes集群交互，我之前一直是每个集群一个config文件，然后通过`cp ~/.kube/config.xxxx ~/.kube/config`这样来切换🤦‍♂️。

其实通过Kubernetes的[Context](https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)，就不用这么麻烦了。不过按照文档操作有点繁琐，其实自己编辑`~/.kube/config`就行了，不过记得先备份一下。就是把其他集群的cluster、user、context分别添加一份到`~/.kube/config`，注意改下名字之类。

之后就可以这样`kubectl config use-context minikube`来切换context了。执行`kubectl config current-context`可以显示当前的context。但是这个命令还是太长了，不友好。可以安装[kubectx](https://github.com/ahmetb/kubectx)，支持交互式操作来选择并切换context和namespace。在macOS只要执行`brew install kubectx`就能安装了。

下面是执行`kubectx`的效果：

![kubectx](/images/kubectx.png)

同样，执行`kubens`可以列出当前context指定集群的可用namespace，然后通过方向键选择。之后执行`kubectl get pod`就不用指定`--context=xxx -n=yyy`来操作了。

更进一步，由于kubectl支持插件机制，我们只要执行：

```shell
ln -s /usr/local/bin/kubectx /usr/local/bin/kubectl-ctx
ln -s /usr/local/bin/kubens /usr/local/bin/kubectl-ns
```

然后只要运行`kubectl ctx`和`kubectl ns`就行了，不需要再敲kubectx和kubens。

