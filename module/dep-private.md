# 2.2 如此，用 dep 获取私有库

## 介绍

`dep`是一个依赖管理工具。它需要 1.9 或更新的`Golang`版本才能编译

`dep`已经能够在生产环节安全使用，但还在官方的试验阶段，也就是还不在`go tool`中。但我想是迟早的事 :=)

指南和参考资料，请参阅[文档](https://golang.github.io/dep/)

## 获取私有库

我们常用的`git`方式有两种，第一种是通过`ssh`，第二种是`https`

本文中我们以`gitlab.com`为案例，创建一个`private`的私有仓库

### 通过`ssh`方式

首先我们需要在本机上生成`ssh-key`，若没有生成过可右拐[传送门](https://segmentfault.com/a/1190000013450267)

得到需要使用的`ssh-key`后，我们打开我们的`gitlab.com`，复制粘贴入我们的`Settings` -> `SSH Keys`中

![image](https://sfault-image.b0.upaiyun.com/105/328/1053288135-5a96d458e9145)

添加成功后，我们直接在`Gopkg.toml`里配置好我们的参数

```
[[constraint]]
  branch = "master"
  name = "gitlab.com/eddycjy/test"
  source = "git@gitlab.com:EDDYCJY/test.git"
```

在拉取资源前，要注意假设你是第一次用该`ssh-key`拉取，需要先手动用`git clone`拉取一遍，同意`ECDSA key fingerprint`：

```
$ git clone git@gitlab.com:EDDYCJY/test.git
Cloning into 'test'...
The authenticity of host 'gitlab.com (52.167.219.168)' can't be established.
ECDSA key fingerprint is xxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no)? yes
...
```

接下来，我们在项目下**直接执行`dep ensure`就可以拉取**下来了！

#### 问题

1. 假设你是第一次，又没有执行上面那一步（`ECDSA key fingerprint`），会一直卡住

2. 正确的反馈应当是执行完命令后没有错误，但如果出现该错误提示，那说明该存储仓库没有被纳入`dep`中（例：`gitee`）

```
$ dep ensure

The following issues were found in Gopkg.toml:

unable to deduce repository and source type for "xxxx": unable to read metadata: go-import metadata not found

ProjectRoot name validation failed
```

### 通过`https`方式

我们直接在`Gopkg.toml`里配置好我们的参数

```
[[constraint]]
  branch = "master"
  name = "gitlab.com/eddycjy/test"
  source = "https://{username}:{password}@gitlab.com"
```

主要是修改`source`的配置项，`username`填写在`gitlab`的用户名，`password`为密码

最后回到项目下**执行`dep ensure`拉取资源**就可以了

## 最后

`dep`目前还是官方试验阶段，还可能存在变数，多加注意
