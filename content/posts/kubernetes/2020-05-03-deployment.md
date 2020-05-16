---
title: "在 Kubernetes 中部署应用程序"
date: 2020-05-03T11:05:00+08:00
toc: true
tags: 
  - kubernetes
---

在完成了本地 Kubernetes 的快速搭建（基于 Docker）后，我们已经可以正式的使用它了。对于我们平时最常见的需求，那就是往 Kubernetes 里部署应用程序，如果你没有看过 Kubernetes 相关的知识，这时候你可能会六神无主，但问题不大，我们就可以使用最经典的 Nginx 来小试身手。


## 创建 Deployment

创建 nginx-deployment.yaml 文件：

```yaml
apiVersion: apps/v1
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

```shell
$ kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created
```

## 查看运行状态

查看 Pod 运行情况：

```shell
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-9fbc65d67-9j68x   1/1     Running   0          1m
nginx-deployment-9fbc65d67-nwbhj   1/1     Running   0          1m
```

查看 Deployment 部署情况：

```shell
$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           29m
```

我们也可以通过 describe 命令进行查看

```shell
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

```shell
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
```

应用 nginx-service.yaml 文件：

```shell
$ kubectl apply -f nginx-service.yaml
```

查看应用的运行情况：

```shell
$ kubectl get services -o wide
```

但这时候是无法访问到 Nginx 的，我们可以通过 Kubernetes 的 NodePort 的方式对外提供访问：


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

然后再进行访问：

```shell
$ curl http://127.0.0.1:30001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...


```

至此我们已经打通了和 Nginx 之间的访问。

## 部署 Go 程序

在部署环境中常常需要将应用程序部署上去，然后对外进行提供服务，我们模拟一个 Go 程序：

```go
func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.String(http.StatusOK, "pong")
    })
    err := r.Run(":9001")
    if err != nil {
        log.Fatalf("r.Run err: %v", err)
    }
}
```

### 编写和编译 Dockerfile

在项目根目录创建 Dockerfile 文件，进行编写：

```shell
FROM golang:latest

ENV GOPROXY https://goproxy.cn,direct
WORKDIR $GOPATH/src/github.com/eddycjy/awesome-project
COPY . $GOPATH/src/github.com/eddycjy/awesome-project
RUN go build .

EXPOSE 8000
ENTRYPOINT ["./awesome-project"]
```

编译并打标签：

```shell
$ docker build -t eddycjy/awesome-project:v0.0.1 .
...
Successfully built b53cef4d2967
Successfully tagged eddycjy/awesome-project:v0.0.1
```

验证打包进 Docker 中的程序是否正常运行：

```shell
$ docker run -p 10001:9001 awesome-project
[GIN-debug] GET    /ping                     --> main.main.func1 (3 handlers)
[GIN-debug] Listening and serving HTTP on :9001
[GIN] 2020/05/03 - 01:51:40 | 200 |        16.9µs |      172.17.0.1 | GET      "/ping"
```

### 上传到 Dockerhub

登陆并推送镜像到 Dockerhub：

```shell
$ docker login  
$ docker push eddycjy/awesome-project:v0.0.1      
The push refers to repository [docker.io/eddycjy/awesome-project]
8192ac09ffeb: Pushed 
9eb17b90d619: Pushed 
b04d698ea69d: Pushed 
31561785c3fc: Mounted from library/golang 
4486631650dc: Mounted from library/golang 
5e28718a7d23: Mounted from library/golang 
ea1227feeccb: Mounted from library/golang 
9cae1895156d: Mounted from library/golang 
52dba9daa22c: Mounted from library/golang 
78c1b9419976: Mounted from library/golang 
v0.0.1: digest: sha256:a1ef61e899db75eb2171652356be15559f1991b94a971306fb79ceccea8dd515 size: 2422
```

这时候你在 hub.docker.com 上就能看的你刚刚所上传的镜像内容：

![image](https://image.eddycjy.com/605dca784a33f0466c655d4418818154.jpg)

## 编写 Kubernetes 配置

接下来我们需要针对刚刚所打包的 Go 程序创建 Deployment，编写 go-deployment.yaml 文件：

```shell
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: awesome-project
  labels:
    app: awesome-project
spec:
  replicas: 2
  selector:
    matchLabels:
      app: awesome-project
  template:
    metadata:
      labels:
        app: awesome-project
    spec:
      containers:
      - name: awesome-project
        image: eddycjy/awesome-project:v0.0.1
```

创建 Service，编写 go-service.yaml：

```
apiVersion: v1
kind: Service
metadata:
  name: awesome-project-svc
  labels:
    app: awesome-project
spec:
  ports:
  - port: 9001
  type: ClusterIP
  selector:
    app: awesome-project
```

## 部署 Ingress

### Ingress Controller

我们采用 Docker for Mac 特定提供的 Ingress Controller 部署脚本：

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/cloud/deploy.yaml
```

其会所有命名空间监视 Ingress 对象，并配置 RBAC 权限，否则你有可能会遇到 403 Forbidden 的问题。

### Nginx Ingress

在完成了 Ingress Controller 等相关部署后，我们可以正式的部署属于自己业务的 Nginx Ingress 对象：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - host: website-ingress.local
    http:
      paths:
      - backend:
          serviceName: awesome-project-svc
          servicePort: 9001
```

`kubectl apply -f` 应用刚刚所编写的配置文件，然后查看运行情况：

```
$ kubectl get ingresses.
NAME           HOSTS                   ADDRESS     PORTS   AGE
test-ingress   website-ingress.local   localhost   80      8h
```

如何发现 ADDRESS 为空，则存在问题，需要进行排查（可能性有很多）。在确定 ADDRESS 属性正常后，我们需要打开 `/etc/hosts` 并配置 HOST `127.0.0.1 awesome-project.local` ，并进行验证：

```shell
$ curl http://awesome-project.local/ping
pong
```

至此，我们完成了一个简单的 Go 程序的部署和外部调用。

## 小结

在本章中，我们通过部署 Nginx、Ingress、Go 程序的方式，直接实践了 Kubernetes 的基本流程，达到了将自己的简单程序部署在 Kubernetes 的一个小目标，接下来在后续的章节中我们将进一步针对文中所使用到的相关属性和内容进行详细说明。

毕竟在实践过后，就要去了解为什么，这样子才能做到融会贯通。
