---
title: kustomize介绍
linktitle: kustomize介绍

summary: kubernetes源码分析，包括各个模块的分析

date: 2022-07-10

type: book

weight: 1
---

本文只是记录笔记，便于记忆。建议 [阅读原文](https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/kustomization/)

Kustomize 是一个独立的工具，用来通过 Kustomization 文件定制 k8s 对象。

从 1.14 版本开始，kubectl 也支持 kustomization 文件来管理 k8s 对象，要查看 kustomization 文件的目录中的资源，执行下面的命令

```bash
kubectl kustomize <dir>
```

要应用这些资源，执行以下命令

```bash
kubectl apply -k <dir>
```



## 概述

kustomzie 是一个用来定制 kubernetes 配置的工具，它提供以下功能特性来管理应用配置文件：

- 从其他来源生成资源
- 为资源设置贯穿性字段
- 组织和定制资源集合

### 从其他来源生成资源

configmap 和 secret 中保存一些数据信息，这些信息会被其他资源使用（比如pod）。而 configmap 或 secret 中的数据往往来源于集群外部，比如某个 .properties 文件或者 ssh 密钥文件，kustomize 提供 `configMapGenerator`和`secretGenerator`,可以基于文件或字面值来生成secret和configmap

#### configMapGenerator

```bash
cat <<EOF >application.properties
Foo=Bar
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

可以使用以下命令检查生成的configmap是否正确：

```bash
kubectl kustomize ./
```

ConfigMap 也可以基于字面的键值对来生成，只需要在 configMapGenerator 的 literals 列表中添加表项：

```sh
cat EOF<< ./kustomization.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - Foo=Bar
EOF
```

#### secretGenerator

基于文件或键值对生成secret，和前面生成configmap类似

```bash
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

基于键值对生成secret

```bash
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
EOF
```

#### generatorOptions

生成的configmap和secret都会包含内容哈希值后缀，这是为了确保内容发生时，所生成的是新的 configmap 或 secret，要禁止自动添加后缀，用户可以使用 generatorOptions

```bash
cat EOF<< ./kustomization.yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - Foo=Bar
generatorOptions:
	disableNameSuffixHash: true
	labels:
		type: generated
	annotations:
		note: generated
EOF
```



### 设置贯穿性字段

在项目中为所有 k8s 资源对象设置贯穿性字段是常见的操作，使用场景如下：

- 为所有资源设置相同的命名空间
- 为所有资源对象添加相同的前缀或后缀
- 为对象添加相同的标签集合
- 为对象添加相同的注解集合

```bash
cat <<EOF >./kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
commonLabels:
	app: bingo
commonAnnotations:
  oncallPager: 123456
resources:
- deployment.yaml
EOF
```

### 组织和定制资源

一种常见的做法是在项目中构造资源集合并将其放到同一个文件或目录中管理。kuztomization 提供基于不同文件来组织资源并向其应用补丁或其他定制的能力。

#### 组织

kustomize 支持组合不同的资源。kustomization.yaml 文件中的 resources 字段定义配置中要包含的资源列表

```bash
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

#### 定制

补丁文件可以用来对资源执行不同的定制，kustomize 通过 `patchesStrategicMerge`和 `patchesJson6902`支持不同的打补丁机制。除此之外，还提供`images`字段设置新的镜像来更改容器中使用的镜像

- patchesStrategicMerge：内容是文件路径列表，每个文件都可以对原文件打补丁。补丁文件中的名称必须与待打补丁的资源名称匹配。

  ```bash
  # 生成另一个补丁 set_memory.yaml
  cat <<EOF > set_memory.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-nginx
  spec:
    template:
      spec:
        containers:
        - name: my-nginx
          resources:
            limits:
              memory: 512Mi
  EOF
  
  cat <<EOF >./kustomization.yaml
  resources:
  - deployment.yaml
  patchesStrategicMerge:
  - increase_replicas.yaml
  - set_memory.yaml
  EOF
  ```

  

- patchesJson6902：并非所有的字段都支持策略性合并，为了支持对任意字段修改，通过pathesJson6902来应用json的补丁能力。为了找到正确的资源，需要指定资源的group、version、kind 和 name

  ```bash
  # 创建一个 JSON 补丁文件
  cat <<EOF > patch.yaml
  - op: replace
    path: /spec/replicas
    value: 3
  EOF
  
  # 创建一个 kustomization.yaml
  cat <<EOF >./kustomization.yaml
  resources:
  - deployment.yaml
  
  patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx
    path: patch.yaml
  EOF
  ```

- images：

  ```bash
  cat <<EOF > deployment.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-nginx
  spec:
    selector:
      matchLabels:
        run: my-nginx
    replicas: 2
    template:
      metadata:
        labels:
          run: my-nginx
      spec:
        containers:
        - name: my-nginx
          image: nginx
          ports:
          - containerPort: 80
  EOF
  
  cat <<EOF >./kustomization.yaml
  resources:
  - deployment.yaml
  images:
  - name: nginx
    newName: my.image.registry/nginx
    newTag: 1.4.0
  EOF
  ```

  

## 基准与覆盖

Kustomize 中有基准（bases）和覆盖（overlays）的概念区分。

- 基准：是包含`kustomization.yaml`文件的一个目录，可以是本地目录或远程目录，只要存在 `kustomization.yaml`文件即可。
- 覆盖：也是一个目录，包含将其他kustomization 目录当做 bases 类引用的 kusomization.yaml 文件

使用场景：开发环境、测试环境 等 有不同的配置属性

```bash
mkdir dev
cat <<EOF > dev/kustomization.yaml
bases:
- ../base
namePrefix: dev-
EOF

mkdir prod
cat <<EOF > prod/kustomization.yaml
bases:
- ../base
namePrefix: prod-
EOF
```



## 使用kustomize来应用、查看、删除对象

### 应用对象

```bash
kubectl apply -k ./
```

### 查看对象

```bash
kubectl get -k ./
# 或者
kubectl describe -k ./
```

### 比较对象

```bash
kubectl diff -k ./
```

### 删除对象

```bash
kubectl delete -k ./
```

## 功能特性列表

| 字段                  | 类型                                                         | 解释                                                         |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| namespace             | string                                                       | 为所有资源添加名字空间                                       |
| namePrefix            | string                                                       | 此字段的值将被添加到所有资源名称前面                         |
| nameSuffix            | string                                                       | 此字段的值将被添加到所有资源名称后面                         |
| commonLabels          | map[string]string                                            | 要添加到所有资源和选择算符的标签                             |
| commonAnnotations     | map[string]string                                            | 要添加到所有资源的注解                                       |
| resources             | []string                                                     | 列表中的每个条目都必须能够解析为现有的资源配置文件           |
| configmapGenerator    | [][ConfigMapArgs](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/kustomization.go#L99) | 列表中的每个条目都会生成一个 ConfigMap                       |
| secretGenerator       | [][SecretArgs](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/kustomization.go#L106) | 列表中的每个条目都会生成一个 Secret                          |
| generatorOptions      | [GeneratorOptions](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/kustomization.go#L109) | 更改所有 ConfigMap 和 Secret 生成器的行为                    |
| bases                 | []string                                                     | 列表中每个条目都应能解析为一个包含 kustomization.yaml 文件的目录 |
| patchesStrategicMerge | []string                                                     | 列表中每个条目都能解析为某 Kubernetes 对象的策略性合并补丁   |
| patchesJson6902       | [][Json6902](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/patchjson6902.go#L8) | 列表中每个条目都能解析为一个 Kubernetes 对象和一个 JSON 补丁 |
| vars                  | [][Var](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/var.go#L31) | 每个条目用来从某资源的字段来析取文字                         |
| images                | [][Image](https://github.com/kubernetes-sigs/kustomize/tree/master/api/types/image.go#L23) | 每个条目都用来更改镜像的名称、标记与/或摘要，不必生成补丁    |
| configurations        | []string                                                     | 列表中每个条目都应能解析为一个包含 [Kustomize 转换器配置](https://github.com/kubernetes-sigs/kustomize/tree/master/examples/transformerconfigs) 的文件 |
| crds                  | []string                                                     | 列表中每个条目都赢能够解析为 Kubernetes 类别的 OpenAPI 定义文件 |