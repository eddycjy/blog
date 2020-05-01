---
title: "在 Kubernetes 中部署应用程序"
date: 2020-05-01T17:08:41+08:00
toc: true
draft: true
tags: 
  - kubernetes
---

在完成了本地 Kubernetes 的快速搭建（基于 Docker）后，我们已经可以正式的使用它了。对于我们平时最常见的需求，那就是往 Kubernetes 里部署应用程序，如果你没有看过 Kubernetes 相关的知识，这时候你可能会六神无主，但问题不大，我们就可以使用最经典的 Nginx 来小试身手。


## 创建 Deployment

创建 nginx-deployment.yaml 文件：

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.18.0
```

应用 nginx-deployment.yaml 文件：

```
$ kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created
```

## 查看运行状态

查看 Pod 运行情况：

```
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-9fbc65d67-9j68x   1/1     Running   0          1m
nginx-deployment-9fbc65d67-nwbhj   1/1     Running   0          1m
```

查看 Deployment 部署情况：

```
$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           29m
```

我们也可以通过 describe 命令进行查看

```
$ kubectl describe pod nginx-deployment-9fbc65d67-9j68x
Name:           nginx-deployment-9fbc65d67-9j68x
Namespace:      default
Priority:       0
Node:           docker-desktop/192.168.65.3
Start Time:     Fri, 01 May 2020 17:36:12 +0800
Labels:         app=nginx
                pod-template-hash=9fbc65d67
...
Events:
  Type    Reason     Age   From                     Message
  ----    ------     ----  ----                     -------
  Normal  Scheduled  45m   default-scheduler        Successfully assigned default/nginx-deployment-9fbc65d67-9j68x to docker-desktop
  Normal  Pulling    45m   kubelet, docker-desktop  Pulling image "nginx:1.18.0"
  Normal  Pulled     44m   kubelet, docker-desktop  Successfully pulled image "nginx:1.18.0"
  Normal  Created    44m   kubelet, docker-desktop  Created container nginx
  Normal  Started    44m   kubelet, docker-desktop  Started container nginx
```

## 查看 Dashboard

在应用了 Nginx 的 Deployment 后，我们可以查看上一章节中我们所搭建的 Dashboard：

![image](https://image.eddycjy.com/8e08a407d9333759e99a5f09f18e6e8c.jpg)

可能你在想，我只是执行了一条命令，怎么就把 Nginx 跑起来了，这时候你可以去查看容器组中的事件，就能够看到这个容器在运行时做涉及到的事件：

![image](https://image.eddycjy.com/8d96979504a96c5d5b60fad3eeb35060.jpg)

## 部署 Nginx

### 创建 Nginx Service

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 80
    nodePort: 30001
    targetPort: 80
  type: NodePort
```

应用 nginx-service.yaml 文件：

```
$ kubectl apply -f nginx-service.yaml
```

查看应用的运行情况：

```
$ kubectl get services -o wide
```

### 验证 Nginx 访问

```
$ curl 127.0.0.1:30001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
...
```


### Nginx 配置加载

这个时候你又会遇到一个新的问题，那就是 Nginx 跑起来了，但是在实际应用场景中，Nginx 可是配了一大堆的路由的，那我们在 Kubernetes 中要如何处理呢。

### ConfigMap


## 部署 Go 程序