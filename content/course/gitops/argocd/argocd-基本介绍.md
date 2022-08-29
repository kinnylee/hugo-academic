---
title: argocd源码分析-基本介绍
author: kinnylee
date: 2021-03-21
toc: true
categories:
  - 云原生
tags:
  - 源码分析
  - argocd
thumbnail: 'images/argocd.png'



---

# Argocd源码分析-基本介绍

## 概述

argocd是一个基于k8s的声明式的、GitOps工具

## 为什么应该使用argocd

- 应用程序定义、配置、环境应该是声明式的和版本控制的
- 应用程序发布和生命周期管理应该是自动化的、可审计的和易于理解的

## 架构

![官方架构图](https://argoproj.github.io/argo-cd/assets/argocd_architecture.png)

argocd有一个客户端组件和多个服务端组件：

控制器：argocd-application-controller，以 StatefulSet 的方式部署

客户端组件：argocd

后台服务端包括以下几个组件：

- argocd-server：后台服务，提供http服务给ui，提供grpc服务给cli
- argocd-repo-server：仓库服务
- argocd-dex-server：集成第三方sso
- argocd-redis：缓存服务redis

### argocd-server

argocd-server 是 argocd 中的 API Server，提供 API 给 web ui、命令行cli 和其他 ci\cd 系统调用，主要职责如下：

- 应用管理和状态上报
- 调用应用的操作（比如：同步、回滚、其他用户定义的动作）
- 仓库、集群凭据管理
- 代理外部系统提供的认证和授权
- RBAC增强
- 监听处理 git webhook 事件

### argocd-repo-server

repo-server是一个内部服务，维护一个持有应用程序部署文件的git仓库本地缓存，主要职责是根据用户提供的以下输入信息，生成和返回k8s manifests文件：

- git仓库 URL 地址
- git版本信息（commit、tag、branch）
- 文件在git仓库中的相对路径
- 特殊的模板设置：参数、ksonnet环境、helm的values.yaml文件等

### application-controller

application-controller是一个k8s的控制器，持续监听运行的应用程序，并且对比当前状态和期望状态。主要职责是执行用户定义的用于声明周期的各种事件

## 安装

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 配置ingress

执行以下命令

 ```bash
kubectl apply -f ingress.yaml -n argocd
 ```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.glodon.com
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
```

### 登录账号

- 用户名：admin
- 密码：默认是argocd-server的pod的名称

可以通过以下命令获取：

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

## 声明式安装

Argo cd 提供两个自定义CRD：Application 和 AppProject，用于描述应用信息和项目信息。

其他的资源也可以通过声明式的方式存储，使用kubectl apply 进行安装，最终存储在k8s集群的etcd中。

所有的资源都被安装在 argocd 这个命名空间中，这也意味这项目名和应用名不能重复。

### Application

Application代表一个部署的应用实例，包括两个关键的字段：

- source：指定了存储在git中的应用的期望状态
- destination：指定了目标集群和命名空间

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test
  namespace: argocd
spec:
  destination:
    namespace: test
    server: https://kubernetes.default.svc
  project: default
  source:
    path: test
    repoURL: http://geek.glodon.com/scm/cloudsandbox/cloudsandbox-manifest.git
    targetRevision: dev
    # 以下两项是 helm 仓库的配置
    # repoURL: https://argoproj.github.io/argo-helm
    # chart: argo
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
```

### AppProject

AppProject代表一批应用的逻辑分组，定义了以下字段：

- sourceRepos：在该项目内的应用可以从哪些远程仓库拉取manifest信息
- destinations：项目内的应用可以部署的目标集群和命名空间
- roles：项目内的应用可以访问的资源信息

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - namespace: '*'
    server: '*'
  sourceRepos:
  - '*'

```

### 仓库信息

> 有些仓库（比如gitlab）需要特别指定url地址的后缀`.git`，否则服务的将发送301重定向到带`.git`后缀的地址去，argocd不支持这种重定向，因此你必须修改url配置，显示的加上后缀。

仓库凭据存储在secret中，仓库信息存储在`argocd-cm`这个configmap中，每个仓库地址必须有一个url字段，其他字段根据使用的仓库类型决定：

- https类型的git仓库
  - usernameSecret
  - passwordSecret
- ssh类型的git仓库
  - sshPrivateKeySecret
- github仓库地址
  - githubAppPrivateKeySecret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
data:
  repositories: |
    - url: http://geek.xxx.com/scm/xxx/xxx-manifest.git
      type: git
      passwordSecret:
        key: password
        name: repo-1616912267
      usernameSecret:
        key: username
        name: repo-1616912267
    - url: git@github.com:argoproj/my-private-repository
      sshPrivateKeySecret:
        name: my-secret
        key: sshPrivateKey
    - url: https://ghe.example.com/argoproj/my-private-repository
      githubAppID: 1
      githubAppInstallationID: 2
      githubAppEnterpriseBaseUrl: https://ghe.example.com/api/v3
      githubAppPrivateKeySecret:
        name: my-secret
        key: githubAppPrivateKey
```

### 密码信息

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-1616912267
  namespace: argocd
type: Opaque
data:
  password: xxxxxxx
  username: xxxxxx
```

### 集群信息

集群凭证和仓库凭证一样存储在secret中，但是入口不需要存储在argocd-cm 这个configmap中，secret必须有`argocd.argoproj.io/secret-type:cluster`这个标签，secret数据必须包含以下字段：

-  name：集群名称
- server：集群api url地址
- namespaces：可选项，逗号分割。集群可访问namespace列表
- config：配置信息

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mycluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: mycluster.com
  server: https://mycluster.com
  config: |
    {
      "bearerToken": "<authentication token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64 encoded certificate>"
      }
    }
```

### Helm chart 仓库

非标准的 helm chart 仓库必须注册在`argocd-cm`这个configmap的`repositories`字段下，每个仓库必须有url、type、name字段，私有仓库必须配置访问凭证和https证书

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # v1.2 or earlier use `helm.repositories`
  helm.repositories: |
    - url: https://storage.googleapis.com/istio-prerelease/daily-build/master-latest-daily/charts
      name: istio.io
  # v1.3 or later use `repositories` with `type: helm`
  repositories: |
    - type: helm
      url: https://storage.googleapis.com/istio-prerelease/daily-build/master-latest-daily/charts
      name: istio.io
    - type: helm
      url: https://argoproj.github.io/argo-helm
      name: argo
      usernameSecret:
        name: my-secret
        key: username
      passwordSecret:
        name: my-secret
        key: password
      caSecret:
        name: my-secret
        key: ca
      certSecret:
        name: my-secret
        key: cert
      keySecret:
        name: my-secret
        key: key
```

### 资源排除、包含

在执行自动发现和同步时，可以配置某些资源被排除，这样argocd就能够自动感知到。排除的资源配置在`argocd-cm`这个configmap的`resource.exclusions `字段中，这个字段是一个数组类型，每个对象可以包括以下字段：

- apiGroup
- kinds
- cluster

如果这三个字段都被匹配到，这个资源将被忽略。

```yaml
apiVersion: v1
data:
  resource.exclusions: |
    - apiGroups:
      - "*"
      kinds:
      - "*"
      clusters:
      - https://192.168.0.20
kind: ConfigMap
```

除了配置排除，还可以使用`resource.include`配置包含项，默认会包含所有的资源。

```yaml
apiVersion: v1
data:
  resource.inclusions: |
    - apiGroups:
      - "*"
      kinds:
      - Deployment
      clusters:
      - https://192.168.0.20
kind: ConfigMap
```

## 高可用

argocd大部分是无状态，所有数据存储在k8s资源对象中，最终转换为etcd中数据存储。redis是唯一的作为一次性缓存可能会丢失数据，如果丢失将会被重新恢复而不会影响服务。

[ha manifest](https://github.com/argoproj/argo-cd/tree/master/manifests) 提供给用户用于高可用安装。

> 由于pod的亲和性限制，高可用安装至少需要三个节点

### 扩容

#### argocd-repo-server

argocd-repo-server负责克隆git仓库上最新的文件，并使用合适的工具生成manifest文件，相关配置如下：

- argocd-repo-server 通过 fork/exec 执行配置管理工具生成manifests。fork可能会由于内存不足而失败，--parallelismlimit 参数可以控制并发生成manifests的数量，避免内存不足导致oom
- argocd-repo-server为了确保生成manifests时的干净环境，导致一个git仓库对应多个Application时，可能导致git服务端的性能问题，参考后面的内容查看更多信息
- argocd-repo-server克隆仓库代码到/tmp目录，如果有很多仓库或者仓库有很多文件，可能导致磁盘空间不足，为了避免这个问题可以考虑使用pv
- git ls-remote 命令获取git仓库的版本（header、branch 或者tag），这个操作可能会非常频繁并且失败会重试，为了避免失败反复重复，可以设置环境变量`ARGOCD_GIT_ATTEMPTS_COUNT`
- 默认每三分钟同步一次应用的manifests信息，argocd假定应用的manifest改变只有通过仓库的manifest改变，因此设置的默认缓存有效期是24h，可以通过`--repo-cache-expiration duration`设置缓存有效时间
- fork/exec 配置管理工具，比如helm、kustomize，并且强制90s超时，可以通过设置`ARGOCD_EXEC_TIMEOUT`调整超时时间

#### argocd-application-controller

argocd-application-controller 包括两类功能：

- 使用 repo-server 生成manifests
- 通过 k8s api 获取集群的实际状态

相关设置如下：

- 每个controller副本使用2个独立的队列，分别处理应用信息的调谐控制、同步应用信息。每个队列的处理器数量，通过这两个参数配置：`--status-processors` (20 by default) and `--operation-processors` (10 by default) 。可以调整这个数量。1000个应用的配置参考： `--status-processors` and 25 for `--operation-processors`

#### argocd-server

argocd-server是无状态的，并且可能是最不容易出问题的组件，可以考虑最少设置3个副本。

#### argocd-dex-server、argocd-redis-server

- argocd-dex-server使用了一个内存数据库，两个或多个实例可能会导致数据不一致的情况
- argocd-redis-server预先配置了三个服务端是哨兵模式

## 灾备

可以使用`argocd-util`工具导入和导出所有的argocd数据。

确保`~/.kube/config`配置指向argocd集群

### 导出数据

```bash
argocd version | grep server
# ...
export VERSION=v1.0.1
docker run -v ~/.kube:/home/argocd/.kube --rm argoproj/argocd:$VERSION argocd-util export > backup.yaml
```

### 导入数据

```bash
argocd version | grep server
# ...
export VERSION=v1.0.1
docker run -i -v ~/.kube:/home/argocd/.kube --rm argoproj/argocd:$VERSION argocd-util import - < backup.yaml
```



## argocd客户端工具

## 源码分析

### 目录结构

```sh
➜  argo-cd git:(v1.8.7) ✗ tree . -L 1
.
├── CHANGELOG.md
├── Dockerfile
├── Dockerfile.dev
├── LICENSE
├── Makefile
├── OWNERS
├── Procfile
├── README.md
├── SECURITY_CONTACTS
├── USERS.md
├── VERSION
├── assets // 资源文件，包括svg文件、swagger.json文件
├── cmd // 命令行入口文件，包括argocd客户端、server、repo-server、controller、dex
├── common // 定义了常量和版本信息
├── controller // application-controller 代码，实现控制协调，监听 project、application 资源，并做逻辑处理
├── docs // 存放文档
├── examples // known_host 的示例，Grafana的配置文件？
├── go.mod
├── go.sum
├── hack // 脚本文件
├── manifests // 部署 argocd 的编排文件
├── mkdocs.yml
├── overrides
├── pkg // grpc 使用的 protobuf 文件，crd对应的k8s资源文件
├── reposerver // repo-server 服务对应的 grpc 相关代码
├── resource_customizations // 定义了一些yaml文件 和 lua 脚本文件
├── server // server 服务对应的 grpc 相关代码
├── sonar-project.properties
├── test // 测试相关数据
├── tools // 生成命令行工具说明文档的代码
├── ui // 前端页面
├── uid_entrypoint.sh
├── util // 工具类，包含了很多重要的实现逻辑，比如git下载代码、helm和kustomize部署应用等等
└── white-list
```

### argocd命令行工具

