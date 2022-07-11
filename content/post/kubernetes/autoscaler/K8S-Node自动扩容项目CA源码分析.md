---
title: K8S-Node自动扩容项目CA源码分析
subtitle: ""

# Summary for listings and search engines
summary: 上一篇文章介绍了 k8s 自动扩缩容的三种方式：HPA、VPA、CA，以及各自的使用场景和架构，本文针对 kubernetes 扩容项目 autoscaler 子项目 clusterautoscaler 做源码级别分析。

# Link this post with a project
projects: []

# Date published
date: '2020-12-13T00:00:00Z'

# Date updated
lastmod: '2020-12-13T00:00:00Z'

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
  focal_point: ''
  placement: 2
  preview_only: false

authors:
  - admin

tags:
  - 自动扩容
  - 源码分析

categories:
  - Kubernetes

---
`{{< toc >}}`

```go
func (m *AwsManager) Refresh() error {
	...
	// 调用 forceRefresh
	return m.forceRefresh()
}
```

## 一、概述

上一篇文章介绍了 k8s 自动扩缩容的三种方式：HPA、VPA、CA，以及各自的使用场景和架构。本文针对 CA 做源码分析。

### 1.1 CA架构回顾

[参考](https://www.cxybb.com/article/hello2mao/80418625)

CA由一下几个模块组成：

- autoscaler：核心模块，负责整体扩缩容功能
- Estimator：负责评估计算扩容节点
- Simulator：负责模拟调度，计算缩容节点
- Cloud Provider：与云上 IaaS 层交互，负责增删改查节点。云厂商需要实现相关接口。

![CA](https://img-blog.csdn.net/20180726192202249?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvMm1hbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 1.2 仓库代码结构

[源码地址](https://github.com/kubernetes/autoscaler/cluster-autoscaler)

CA 代码在 k8s 官方的 autoscaler 仓库下，该仓库存放自动扩缩容相关组件，包括前文介绍的 VPA、今天的主角CA、还有一个VPA修改pod资源的插件 Addon Resizer。使用的版本是 cluster-autoscaler-release-1.21，目录结构如下

```bash
.
├── CONTRIBUTING.md
├── LICENSE
├── OWNERS
├── README.md
├── SECURITY_CONTACTS
├── addon-resizer       # addon-resizer 代码
├── builder
├── charts
├── cluster-autoscaler  # CA 代码
├── code-of-conduct.md
├── hack
└── vertical-pod-autoscaler # VPA 代码
```

### 1.3 CA 代码结构

```bash
.
├── FAQ.md															# FAQ，里面有很多关于 CA 原理和使用的说明
├── cloudprovider												# cloud provider模块，包括接口和各个云厂商的实现
│   ├── alicloud
│   ├── aws
│   ├── azure
│   ├── baiducloud
│   ├── builder
│   ├── cloud_provider.go								# cloud provider 接口，要对接自己的云，需要实现该接口操作 IaaS 资源
│   ├── gce
│   ├── huaweicloud
├── core                                # CA 核心模块
│   ├── autoscaler.go										# 定义 Autoscaler 接口
│   ├── equivalence_groups.go						# 资源不足的 pod 按扩容类型分类的处理逻辑
│   ├── filter_out_schedulable.go
│   ├── scale_down.go     							# 缩容
│   ├── scale_up.go											# 扩容
│   ├── static_autoscaler.go					  # Autoscaler 的实现类
│   └── utils
├── estimator														# Estimator 模块，评估扩容节点
│   ├── binpacking_estimator.go					# Estimator 实现类，实现首次适应背包算法（装箱算法）
│   └── estimator.go										# 定义 Estimator 接口
├── expander				
│   ├── expander.go											# expander 定义了选择多个符合条件的 NodeGroup 的策略接口
│   ├── factory													# 根据传入的不同策略名称，创建对应的实现类
│   ├── mostpods												# mostpods 策略：调度最多的 pod
│   ├── price														# price 策略：价格最低
│   ├── priority												# priority 策略：根据 NodeGroup 的优先级选择
│   ├── random													# random 策略：随机选择符合条件的 NodeGroup 中的一个
│   └── waste														# waste 策略：资源利用率最高
├── go.mod
├── go.sum															
├── main.go															# main 方法
├── metrics															# 指标采集
├── processors
│   ├── callbacks
│   ├── customresources
│   ├── nodegroupconfig
│   ├── nodegroups
│   ├── nodegroupset
│   ├── nodeinfos
│   ├── nodes
│   ├── pods
│   ├── processors.go
│   └── status
├── proposals													# 提案，设计文档信息
│   ├── balance_similar.md
│   ├── circumvent-tag-limit-aws.md
│   ├── clusterstate.md
│   ├── images
│   ├── kubemark_integration.md
│   ├── metrics.md
│   ├── min_at_zero_gcp.md
│   ├── node_autoprovisioning.md
│   ├── plugable-provider-grpc.md
│   ├── pricing.md
├── simulator														# 模拟调度模块，主要用于缩容
│   ├── basic_cluster_snapshot.go
│   ├── cluster.go
│   ├── cluster_snapshot.go
│   ├── delegating_shared_lister.go
│   ├── delta_cluster_snapshot.go
│   ├── drain.go
│   ├── nodes.go
│   ├── nodes_test.go
│   ├── predicate_error.go
│   ├── predicates_checker_interface.go
│   ├── scheduler_based_predicates_checker.go
│   ├── test_predicates_checker.go
│   ├── test_utils.go
│   ├── tracker.go
```



## 二、CloudProvider 模块

### 2.1 CloudProvider 接口

包含配置信息、与云厂商交互的方法。核心方法有：

- Name()：提供唯一的名称
- Refresh()：刷新云厂商资源信息
- NodeGroups()：获取所有的节点组
- NodeGroupForNode(...)：获取某个节点所属的节点组

```go
type CloudProvider interface {
  // cloud provider 名称
	Name() string
	// 返回 cloud privider 配置的所有 node group
	NodeGroups() []NodeGroup
	// 返回给定 Node 节点所属的 NodeGroup
  // 如果节点不应该被 autoscaler 处理，应该返回 nil
	NodeGroupForNode(*apiv1.Node) (NodeGroup, error)
  // 可选实现，价格模型
	Pricing() (PricingModel, errors.AutoscalerError)
  // 可选实现，获取 cloud provider 支持的所有机器型号
	GetAvailableMachineTypes() ([]string, error)
  // 基于定义的 node，构建理论的 node group
  // 阻塞方法，直到 node group 创建出来
  // 可选实现
	NewNodeGroup(machineType string, labels map[string]string, systemLabels map[string]string,
		taints []apiv1.Taint, extraResources map[string]resource.Quantity) (NodeGroup, error)
	// 返回结构化资源限额
  GetResourceLimiter() (*ResourceLimiter, error)
	// 返回添加到 Node 上的 GPU 资源标签
	GPULabel() string
	// 返回所有可用的 GPU 类型
	GetAvailableGPUTypes() map[string]struct{}
	// 清理工作，比如：go 协程
	Cleanup() error
	// 在每次主循环之前调用，并且用于动态更新 cloud provider 状态
  // 尤其是由 NodeGroups 改变，导致返回一个 node group 列表
	Refresh() error
}
```

### 2.2 NodeGroup 节点组

NodeGroup提供相关接口，操作具有相同容量和标签的一组节点。核心方法有：

- MaxSize()：节点组允许的最大扩容数量
- MinSize()：节点组允许的最小缩容数量
- TargetSize()：节点组当前数量
- IncreaseSize(delta int)：新增 delta 个节点的方法
- DecreaseTargetSize(delta int)：减少节点的方法
- DeleteNodes(...)：删除实例的方法
- TemplateNodeInfo()：

包含配置信息和方法控制

```go
type NodeGroup interface {
  // 返回 node group 的最大数量
	MaxSize() int
	// 返回 node group 的最小数量
	MinSize() int
  // 必须实现该方法。
  // 返回当前目标数量，必须实现该方法
  // 有可能 k8s 节点数量和这个值不相等，但是一旦一切稳定（node完成启动和注册、节点彻底删除）就应该等于 Size()
	TargetSize() (int, error)
  // 必须实现该方法。
  // 增加 node group 的数量，为了删除节点你需要显示指定名称并调用 DeleteNode 方法
  // 该方法会阻塞知道 node group 数量更新
	IncreaseSize(delta int) error
  // 必须实现该方法。
  // 从 node group 中删除节点。
  // 如果失败或者 node 不属于 node group 将会报错。
  // 该方法会阻塞知道 node group 数量更新
	DeleteNodes([]*apiv1.Node) error
  // 从 Node group 中减少目标数量
  // 该方法不允许删除任何节点，仅仅用于减少没有完全填充的新节点
  // 参数 delta 必须是负数，假定 cloud provider 在调整目标数量时，将不会删除存在的节点
	DecreaseTargetSize(delta int) error
	// 返回 node group 唯一标识
	Id() string
	// 返回调试信息
	Debug() string
  // 返回所有属于 node group 的节点列表
  // Instance 对象必须包含 id 字段，其他字段值可选
  // 列表也包含不会成为 k8s 的那些节点
	Nodes() ([]Instance, error)

	// 可选实现
  // 返回包含空 node 新的的调度结构体，将被用于扩容仿真，以预测一个新的扩容节点是什么样的
  // 返回的 NodeInfo 信息包含完整的 Node 对象信N，包括：标签、容量、能分配的 pod 信息（类似kube-proxy）
	TemplateNodeInfo() (*schedulerframework.NodeInfo, error)
  // 必须实现。返回 node group 是否存在
	Exist() bool
	// 可选实现。创建 node group
	Create() (NodeGroup, error)
  // 可选实现，删除 node group
	Delete() error
  // 是否支持自动供应
	Autoprovisioned() bool
  // 可选实现，返回配置参数
	GetOptions(defaults config.NodeGroupAutoscalingOptions) (*config.NodeGroupAutoscalingOptions, error)
}
```

### 2.3 CloudProvider 实现的厂商

大部分云厂商都实现了该接口，[参考](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#faqdocumentation)

- 国外的：亚马逊AWS、谷歌GCE、微软Azure
- 国内的：阿里云、华为云、腾讯云、百度云
- 其他：......

### 2.4 AWS 接口实现

以 aws 为例分析实现实现逻辑，代码结构如下

```bash
.
├── auto_scaling.go
├── auto_scaling_groups.go	 # 获取 asg 信息，保存在 asgCache 缓存中
├── aws_cloud_provider.go		 # awsCloudProvider 实现 CloudProvider 接口里的方法
├── aws_manager.go					 # 根据账号密码，构建 awsManager 对象来操作 aws 资源
├── aws_util.go							 # 获取机型、可用区等信息
├── ec2.go
├── ec2_instance_types			
│   └── gen.go
├── ec2_instance_types.go		 # 默认机型列表
└── examples
    ├── cluster-autoscaler-autodiscover.yaml	# 自动发现模式部署 ca
    ├── cluster-autoscaler-multi-asg.yaml			# 多 asg 模式部署 ca
    ├── cluster-autoscaler-one-asg.yaml				# 单 asg 模式部署 ca
    └── cluster-autoscaler-run-on-control-plane.yaml
```

#### 2.4.1 Name

返回 provider 的名称 aws

```go
// AwsProviderName = "aws"
func (aws *awsCloudProvider) Name() string {
   return cloudprovider.AwsProviderName
}
```

#### 2.4.2 Refresh

调用链：CloudProvider -> AwsManager -> asgCache

refresh 的功能是获取 aws 中的 asg，以及模板、实例等信息保存到缓存中

```go
func (aws *awsCloudProvider) Refresh() error {
	// 调用 awsManager 的 Refresh 方法
  return aws.awsManager.Refresh()
}

func (m *AwsManager) Refresh() error {
	...
  // 调用 forceRefresh
	return m.forceRefresh()
}

func (m *AwsManager) forceRefresh() error {
  // 调用 asgCache 的 regenerate
	if err := m.asgCache.regenerate(); err != nil {
		...
	}
	...
	return nil
}

func (m *asgCache) regenerate() error {
	...
	newInstanceToAsgCache := make(map[AwsInstanceRef]*asg)
	newAsgToInstancesCache := make(map[AwsRef][]AwsInstanceRef)

	// Build list of knowns ASG names
  // 获取所有的 asg name
	refreshNames, err := m.buildAsgNames()

  // 根据 asg name 获取 asg 对象
  // 调用 aws-sdk-go 中 AutoScaling 的 DescribeAutoScalingGroupsPages
	groups, err := m.service.getAutoscalingGroupsByNames(refreshNames)
	// 填充 asg 启动配置的实例模板
	err = m.service.populateLaunchConfigurationInstanceTypeCache(groups)
	if err != nil {
		klog.Warningf("Failed to fully populate all launchConfigurations: %v", err)
	}

	// If currently any ASG has more Desired than running Instances, introduce placeholders
	// for the instances to come up. This is required to track Desired instances that
	// will never come up, like with Spot Request that can't be fulfilled
	groups = m.createPlaceholdersForDesiredNonStartedInstances(groups)

	// Register or update ASGs
	exists := make(map[AwsRef]bool)
	for _, group := range groups {
		asg, err := m.buildAsgFromAWS(group)
		if err != nil {
			return err
		}
		exists[asg.AwsRef] = true
		// 注册 asg
		asg = m.register(asg)

		newAsgToInstancesCache[asg.AwsRef] = make([]AwsInstanceRef, len(group.Instances))

    // 将 group 中所有的实例信息保存到缓存中
		for i, instance := range group.Instances {
      // 根据 group 的 instance 信息构造 instance
			ref := m.buildInstanceRefFromAWS(instance)
			newInstanceToAsgCache[ref] = asg
			newAsgToInstancesCache[asg.AwsRef][i] = ref
		}
	}

  // 注销不存在的 asg
	for _, asg := range m.registeredAsgs {
		if !exists[asg.AwsRef] && !m.explicitlyConfigured[asg.AwsRef] {
			m.unregister(asg)
		}
	}

  // 生成 asg -> instance 的缓存
	m.asgToInstances = newAsgToInstancesCache
  // 生成 instance -> asg 的缓存
	m.instanceToAsg = newInstanceToAsgCache
	return nil
}
```

#### 2.4.3 NodeGroups

调用 awsManager 获取所有的 asg

```go
func (aws *awsCloudProvider) NodeGroups() []cloudprovider.NodeGroup {
  // 调用 awsManager 获取所有的 asg 
  asgs := aws.awsManager.getAsgs()
   ngs := make([]cloudprovider.NodeGroup, len(asgs))
   for i, asg := range asgs {
      ngs[i] = &AwsNodeGroup{
         asg:        asg,
         awsManager: aws.awsManager,
      }
   }

   return ngs
}
```

#### 2.4.4 NodeGroupForNode

- 从 Node.Spec.ProviderID 中获取  id
- 调用 awsManager 获取 id 对应的 asg

```go
func (aws *awsCloudProvider) NodeGroupForNode(node *apiv1.Node) (cloudprovider.NodeGroup, error) {
   if len(node.Spec.ProviderID) == 0 {
      klog.Warningf("Node %v has no providerId", node.Name)
      return nil, nil
   }
   // 从 Node.Spec.ProviderID 中获取  id
   ref, err := AwsRefFromProviderId(node.Spec.ProviderID)
   if err != nil {
      return nil, err
   }
   // 获取 asg
   asg := aws.awsManager.GetAsgForInstance(*ref)

   if asg == nil {
      return nil, nil
   }

   return &AwsNodeGroup{
      asg:        asg,
      awsManager: aws.awsManager,
   }, nil
}
```

#### 2.4.5 IncreaseSize

调用链：CloudProvider -> AwsManager -> asgCache -> aws-sdk-go

通过传入操作 aws asg 的参数，调用 aws-sdk-go 的 asg 接口，实现新增节点的效果

```go
func (ng *AwsNodeGroup) IncreaseSize(delta int) error {
  // 增量不能小于 0
	if delta <= 0 {
		return fmt.Errorf("size increase must be positive")
	}
	size := ng.asg.curSize
  // 增加后不能超过最大值
	if size+delta > ng.asg.maxSize {
		return fmt.Errorf("size increase too large - desired:%d max:%d", size+delta, ng.asg.maxSize)
	}
  // 调用 awsManager 设置 size 为当前 size + delta
	return ng.awsManager.SetAsgSize(ng.asg, size+delta)
}

// SetAsgSize sets ASG size.
func (m *AwsManager) SetAsgSize(asg *asg, size int) error {
	return m.asgCache.SetAsgSize(asg, size)
}

// 加锁操作
func (m *asgCache) SetAsgSize(asg *asg, size int) error {
	m.mutex.Lock()
	defer m.mutex.Unlock()

	return m.setAsgSizeNoLock(asg, size)
}

func (m *asgCache) setAsgSizeNoLock(asg *asg, size int) error {
  // 拼接参数：name、size、honorCooldown
	params := &autoscaling.SetDesiredCapacityInput{
		AutoScalingGroupName: aws.String(asg.Name),
		DesiredCapacity:      aws.Int64(int64(size)),
		HonorCooldown:        aws.Bool(false),
	}
	klog.V(0).Infof("Setting asg %s size to %d", asg.Name, size)
  // 调用 aws-sdk-go 操作 ASG 的 AutoScaling 接口完成操作
	_, err := m.service.SetDesiredCapacity(params)
	if err != nil {
		return err
	}

	// Proactively set the ASG size so autoscaler makes better decisions
	asg.curSize = size

	return nil
}
```

#### 2.4.6 DecreaseTargetSize

执行代码同 IncreaseSize，不同的仅仅是参数 delta是负数。

```go
func (ng *AwsNodeGroup) DecreaseTargetSize(delta int) error {
	// 增量必须为负数
  if delta >= 0 {
		return fmt.Errorf("size decrease size must be negative")
	}

	size := ng.asg.curSize
  // 查询目前 ASG 的节点数
	nodes, err := ng.awsManager.GetAsgNodes(ng.asg.AwsRef)
	if err != nil {
		return err
	}
  // 删除的数量不能超过已有数量
	if int(size)+delta < len(nodes) {
		return fmt.Errorf("attempt to delete existing nodes targetSize:%d delta:%d existingNodes: %d",
			size, delta, len(nodes))
	}
  // 方法同 IncreaseSize 中一样
	return ng.awsManager.SetAsgSize(ng.asg, size+delta)
}
```

#### 2.4.7 DeleteNodes

调用链：CloudProvider -> AwsManager -> aws-sdk-go

通过 Node.Spec.ProviderID 拿到实例唯一 id，调用 SDK 时传入 id，执行删除操作

```go
func (ng *AwsNodeGroup) DeleteNodes(nodes []*apiv1.Node) error {
	size := ng.asg.curSize
	if int(size) <= ng.MinSize() {
		return fmt.Errorf("min size reached, nodes will not be deleted")
	}
	refs := make([]*AwsInstanceRef, 0, len(nodes))
	for _, node := range nodes {
    // 判断待删除 Node 是否是当前 ASG
		belongs, err := ng.Belongs(node)
		if err != nil {
			return err
		}
		if belongs != true {
			return fmt.Errorf("%s belongs to a different asg than %s", node.Name, ng.Id())
		}
    // 获取 Node 的 aws 唯一凭证信息
    // providerID 保存在 node.Spec.ProviderID 字段中
		awsref, err := AwsRefFromProviderId(node.Spec.ProviderID)
		if err != nil {
			return err
		}
		refs = append(refs, awsref)
	}
  // 调用 AwsManager 的删除实例方法
	return ng.awsManager.DeleteInstances(refs)
}

// providerID 的格式是：aws:///zone/name
func AwsRefFromProviderId(id string) (*AwsInstanceRef, error) {
	if validAwsRefIdRegex.FindStringSubmatch(id) == nil {
		return nil, fmt.Errorf("wrong id: expected format aws:///<zone>/<name>, got %v", id)
	}
	splitted := strings.Split(id[7:], "/")
	return &AwsInstanceRef{
		ProviderID: id,
		Name:       splitted[1],
	}, nil
}
```

#### 2.4.8 TemplateNodeInfo

- getAsgTemplate：获取 template
- 

```go
func (ng *AwsNodeGroup) TemplateNodeInfo() (*schedulerframework.NodeInfo, error) {
	// 获取 asg 的模板信息
  template, err := ng.awsManager.getAsgTemplate(ng.asg)
	
	// 通过模板构造 Node 对象
	node, err := ng.awsManager.buildNodeFromTemplate(ng.asg, template)
	// 通过调度框架接口构造调度对象，用于 CA 模拟调度
	nodeInfo := schedulerframework.NewNodeInfo(cloudprovider.BuildKubeProxy(ng.asg.Name))
	nodeInfo.SetNode(node)
	return nodeInfo, nil
}
```

### 2.5 AwsManager 实现

通过前面的分析发现，aws接口实现中和底层IaaS操作的很多逻辑都封装到了 AwsManager 中，这里专门针对 AwsManager做分析。



#### 2.5.1 getAsgTemplate

获取 asg 中第一个可用区的模板信息

```go
func (m *AwsManager) getAsgTemplate(asg *asg) (*asgTemplate, error) {
	// 判断是否有可用区信息
  if len(asg.AvailabilityZones) < 1 {
		return nil, fmt.Errorf("unable to get first AvailabilityZone for ASG %q", asg.Name)
	}

  // asg可配置多个az信息， 默认选择 asg 中第一个可用 az
	az := asg.AvailabilityZones[0]
	region := az[0 : len(az)-1]

	if len(asg.AvailabilityZones) > 1 {
		klog.V(4).Infof("Found multiple availability zones for ASG %q; using %s for %s label\n", asg.Name, az, apiv1.LabelFailureDomainBetaZone)
	}
	// 获取可用机型，通过调用 aws-sdk-go 获取
	instanceTypeName, err := m.buildInstanceType(asg)
	
	if t, ok := m.instanceTypes[instanceTypeName]; ok {
		return &asgTemplate{
			InstanceType: t,
			Region:       region,
			Zone:         az,
			Tags:         asg.Tags,
		}, nil
	}
	return nil, fmt.Errorf("ASG %q uses the unknown EC2 instance type %q", asg.Name, instanceTypeName)
}
```

#### 2.5.2 buildNodeFromTemplate

根据 template 构造 node 对象，预调度就是通过虚拟出来的 Node 对象，传给调度框架来实现。

Node 数据的来源包括：

- asg 模板：提供node 的 cpu、memory、机型、az等信息
- asg tag：提供node 的 taint、label 信息

```go
func (m *AwsManager) buildNodeFromTemplate(asg *asg, template *asgTemplate) (*apiv1.Node, error) {
	node := apiv1.Node{}
  // 生成随机字符串，拼接上 {asgName}-asg-{rand.Int63} 作为主机名
	nodeName := fmt.Sprintf("%s-asg-%d", asg.Name, rand.Int63())

	node.ObjectMeta = metav1.ObjectMeta{
		Name:     nodeName,
		SelfLink: fmt.Sprintf("/api/v1/nodes/%s", nodeName),
		Labels:   map[string]string{},
	}
	/
	node.Status = apiv1.NodeStatus{
		Capacity: apiv1.ResourceList{},
	}

  // 将 asg 返回的机器规格信息填充到 node.Status 中，便于后续调度
	node.Status.Capacity[apiv1.ResourcePods] = *resource.NewQuantity(110, resource.DecimalSI)
  // 构造 node 的 cpu 信息
	node.Status.Capacity[apiv1.ResourceCPU] = *resource.NewQuantity(template.InstanceType.VCPU, resource.DecimalSI)
  // 构造 node 的 gpu 信息
	node.Status.Capacity[gpu.ResourceNvidiaGPU] = *resource.NewQuantity(template.InstanceType.GPU, resource.DecimalSI)
  // 构造 node 的 memory 信息
  // asg 的实例类型的 MemroyMb * 1024 * 1024 作为 node 的 memory
	node.Status.Capacity[apiv1.ResourceMemory] = *resource.NewQuantity(template.InstanceType.MemoryMb*1024*1024, resource.DecimalSI)

	resourcesFromTags := extractAllocatableResourcesFromAsg(template.Tags)
	for resourceName, val := range resourcesFromTags {
		node.Status.Capacity[apiv1.ResourceName(resourceName)] = *val
	}

	// TODO: use proper allocatable!!
	node.Status.Allocatable = node.Status.Capacity

	// 生成 node 的 generic 信息，填充到 label
  // - "kubernetes.io/arch"：asg instance 的机型
  // - "kubernetes.io/os"：linux
	// - "node.kubernetes.io/instance-type"
	// - "topology.kubernetes.io/region"
  // - "topology.kubernetes.io/zone"
	// - "kubernetes.io/hostname"
	node.Labels = cloudprovider.JoinStringMaps(node.Labels, buildGenericLabels(template, nodeName))

  // 填充 node.Label 信息
	// 读取所有的 k8s.io/cluster-autoscaler/node-template/label/ 前缀的 tags
	node.Labels = cloudprovider.JoinStringMaps(node.Labels, extractLabelsFromAsg(template.Tags))
	// 填充 node.Spec.Taints 信息
  // 读取所有的 k8s.io/cluster-autoscaler/node-template/taint/ 前缀
  // 且满足正则 "(.*):(?:NoSchedule|NoExecute|PreferNoSchedule)" 的 tags
	node.Spec.Taints = extractTaintsFromAsg(template.Tags)
	// 填充 node.Status.Conditions
	node.Status.Conditions = cloudprovider.BuildReadyConditions()
	return &node, nil
}
```

### 2.6 asgCache

asgCache用于缓存 aws 中所有的 asg 信息

```go
// asgCache 保存 aws 当前所有的 asg 缓存信息
type asgCache struct {
  // 所有的 asg 列表
	registeredAsgs []*asg
  // asg -> instance 的映射
	asgToInstances map[AwsRef][]AwsInstanceRef
  // instance -> asg 的映射
	instanceToAsg  map[AwsInstanceRef]*asg
	mutex          sync.Mutex
	service        autoScalingWrapper
	interrupt      chan struct{}

	asgAutoDiscoverySpecs []asgAutoDiscoveryConfig
	explicitlyConfigured  map[AwsRef]bool
}

// asg 对应 aws 的 AutoScalingGroup
type asg struct {
	AwsRef

	minSize int
	maxSize int
	curSize int

	AvailabilityZones       []string
	LaunchConfigurationName string
	LaunchTemplate          *launchTemplate
	MixedInstancesPolicy    *mixedInstancesPolicy
	Tags                    []*autoscaling.TagDescription
}
```

#### 2.6.1 regenerate

CA 配置自动发现 asg 机制后，该方法会查找所有打了响应标签的 asg，并将asg的基本信息、实例信息同步到本地内存

```go
func (m *asgCache) regenerate() error {
	m.mutex.Lock()
	defer m.mutex.Unlock()

	newInstanceToAsgCache := make(map[AwsInstanceRef]*asg)
	newAsgToInstancesCache := make(map[AwsRef][]AwsInstanceRef)

	// 通过 CA 启动参数中配置的标签，查找符合条件的所有 asg
	refreshNames, err := m.buildAsgNames()
	
  // 根据 asg 名称，查找完整的 asg 详细信息
	groups, err := m.service.getAutoscalingGroupsByNames(refreshNames)
	
 
	for _, group := range groups {
		asg, err := m.buildAsgFromAWS(group)
		if err != nil {
			return err
		}
		exists[asg.AwsRef] = true
		// 注册 asg 到缓存
		asg = m.register(asg)

		newAsgToInstancesCache[asg.AwsRef] = make([]AwsInstanceRef, len(group.Instances))
		// 建立映射关系
		for i, instance := range group.Instances {
			ref := m.buildInstanceRefFromAWS(instance)
			newInstanceToAsgCache[ref] = asg
			newAsgToInstancesCache[asg.AwsRef][i] = ref
		}
	}

	// Unregister no longer existing auto-discovered ASGs
	for _, asg := range m.registeredAsgs {
		if !exists[asg.AwsRef] && !m.explicitlyConfigured[asg.AwsRef] {
			m.unregister(asg)
		}
	}

	m.asgToInstances = newAsgToInstancesCache
	m.instanceToAsg = newInstanceToAsgCache
	return nil
}
```

### 2.7 Aws provider 创建的流程

调用链路：  NewCloudProvider -> buildCloudProvider -> BuildAWS -> CreateAwsManager -> BuildAwsCloudProvider

- AWS账号初始化
  - 读取配置文件
  - 根据配置文件创建 AWSSDKProvider
  - 创建 Session，创建 AwsService
- 构造 asgCache
  - 解析自动发现 asg 的tag等入参信息
  - 自动同步符合 tag 的 aws asg 到本地 asgCache
- 初始化 awsManager

```go
func initializeDefaultOptions(opts *AutoscalerOptions) error {
  ...
  if opts.CloudProvider == nil {
    // 创建一个 CloudProvider
		opts.CloudProvider = cloudBuilder.NewCloudProvider(opts.AutoscalingOptions)
	}
  ...
}

func NewCloudProvider(opts config.AutoscalingOptions) cloudprovider.CloudProvider {
	...
  // 根据 options参数，构建 provider
	provider := buildCloudProvider(opts, do, rl)
	if provider != nil {
		return provider
	}
}

func buildCloudProvider(opts config.AutoscalingOptions, do cloudprovider.NodeGroupDiscoveryOptions, rl *cloudprovider.ResourceLimiter) cloudprovider.CloudProvider {
	switch opts.CloudProviderName {
	...
  // aws 的 provider
	case cloudprovider.AwsProviderName:
		return aws.BuildAWS(opts, do, rl)
  ...
 }
}

func BuildAWS(...) {
  ...
  // 初始化 awsManager
  manager, err := CreateAwsManager(config, do, instanceTypes)
	// 初始化 provider
	provider, err := BuildAwsCloudProvider(manager, rl)
	
	return provider
}

func CreateAwsManager(...) (*AwsManager, error) {
	return createAWSManagerInternal(configReader, discoveryOpts, nil, nil, instanceTypes)
}

func createAWSManagerInternal(
	configReader io.Reader,
	discoveryOpts cloudprovider.NodeGroupDiscoveryOptions,
	autoScalingService *autoScalingWrapper,
	ec2Service *ec2Wrapper,
	instanceTypes map[string]*InstanceType,
) (*AwsManager, error) {
	// 读取配置文件
	cfg, err := readAWSCloudConfig(configReader)
	...
  // 解析自动发现 asg 的入参信息
	specs, err := parseASGAutoDiscoverySpecs(discoveryOpts)
	...
  // 初始化 asgCache
	cache, err := newASGCache(*autoScalingService, discoveryOpts.NodeGroupSpecs, specs)

  // 初始化 awsManager
	manager := &AwsManager{
		autoScalingService: *autoScalingService,
		ec2Service:         *ec2Service,
		asgCache:           cache,
		instanceTypes:      instanceTypes,
	}
	// 执行刷新操作，将 aws 的 asg 信息同步到本地 asgCache
	if err := manager.forceRefresh(); err != nil {
		return nil, err
	}

	return manager, nil
}
  
 
```

#### 2.7.1 BuildAWS

```go
func BuildAWS(opts config.AutoscalingOptions, do cloudprovider.NodeGroupDiscoveryOptions, rl *cloudprovider.ResourceLimiter) cloudprovider.CloudProvider {
	// 读取参数中配置相关的 CloudConfig 文件
  var config io.ReadCloser
	if opts.CloudConfig != "" {
		var err error
		config, err = os.Open(opts.CloudConfig)
		if err != nil {
			klog.Fatalf("Couldn't open cloud provider configuration %s: %#v", opts.CloudConfig, err)
		}
		defer config.Close()
	}

	// 获取机型
	instanceTypes, lastUpdateTime := GetStaticEC2InstanceTypes()
  // 获取静态机型，可能会过时
	if opts.AWSUseStaticInstanceList {
		klog.Warningf("Using static EC2 Instance Types, this list could be outdated. Last update time: %s", lastUpdateTime)
	} else {
    // 实时当前可用区
    // 先读取环境变量：AWS_REGION，找不到再调接口获取
		region, err := GetCurrentAwsRegion()
		...
    // 获取机型
		generatedInstanceTypes, err := GenerateEC2InstanceTypes(region)
		...
	}
	// 创建 AwsManager
  // AwsManager 用于操作 aws 资源
	manager, err := CreateAwsManager(config, do, instanceTypes)
	...
  // 创建 provider
	provider, err := BuildAwsCloudProvider(manager, rl)
	...
  // 注册指标
	RegisterMetrics()
	return provider
}
```



## 三、主流程

关键代码逻辑，梳理成流程图，可以对照查看。[高清图](https://www.processon.com/diagraming/62bd5b377d9c0807352493d1)

![代码图](http://assets.processon.com/chart_image/62bd5b377d9c0807352493d4.png)

### 3.1 main 启动入口

```go
func main() {
  // 选取leader
 	leaderElection := defaultLeaderElectionConfiguration()
	leaderElection.LeaderElect = true

  go func() {
    // 注册指标、快照、监控检查接口
		pathRecorderMux := mux.NewPathRecorderMux("cluster-autoscaler")
		defaultMetricsHandler := legacyregistry.Handler().ServeHTTP
		pathRecorderMux.HandleFunc("/metrics", func(w http.ResponseWriter, req *http.Request) {
			defaultMetricsHandler(w, req)
		})
		if *debuggingSnapshotEnabled {
			pathRecorderMux.HandleFunc("/snapshotz", debuggingSnapshotter.ResponseHandler)
		}
		pathRecorderMux.HandleFunc("/health-check", healthCheck.ServeHTTP)

		err := http.ListenAndServe(*address, pathRecorderMux)
	}()

	if !leaderElection.LeaderElect {
		run(healthCheck, debuggingSnapshotter)
	} else {
    ...
    // 入口函数
    run(healthCheck, debuggingSnapshotter)
    ...
  }
}
```

### 3.2 run

- 初始化 autoscaler
- 调用 autoscaler.Start，后台刷新缓存
- 间隔执行（默认10s）autoscaler.RunOnce 方法，实现扩缩容逻辑

```go
func run(healthCheck *metrics.HealthCheck, debuggingSnapshotter debuggingsnapshot.DebuggingSnapshotter) {
	metrics.RegisterAll(*emitPerNodeGroupMetrics)
	// 构造 autoscaler 对象
	autoscaler, err := buildAutoscaler(debuggingSnapshotter)
	...
  // 在后台启动 autoscaler
	if err := autoscaler.Start(); err != nil {
		klog.Fatalf("Failed to autoscaler background components: %v", err)
	}

	// Autoscale ad infinitum.
	for {
		select {
    // 默认 10s 执行一次
		case <-time.After(*scanInterval):
			{
				...
        // 开始执行
				err := autoscaler.RunOnce(loopStart)
				...
			}
		}
	}
}
```

### 3.3 autoscaler

#### 3.3.1 buildAutoscaler

```go
func buildAutoscaler(debuggingSnapshotter debuggingsnapshot.DebuggingSnapshotter) (core.Autoscaler, error) {
	...
  // Create autoscaler.
	return core.NewAutoscaler(opts)
}
```

#### 3.3.2 NewAutoscaler

```go
func NewAutoscaler(opts AutoscalerOptions) (Autoscaler, errors.AutoscalerError) {
	// 这个方法主要是做一下初始化工作，其中 provider 的初始化在介绍 aws provider 创建流程时介绍过
  // 内部会初始化 awsManager、asgCache，并从云厂商同步 asg信息到本地缓存
  err := initializeDefaultOptions(&opts)
	if err != nil {
		return nil, errors.ToAutoscalerError(errors.InternalError, err)
	}
  // 实例化 AutoScaler
	return NewStaticAutoscaler(
		opts.AutoscalingOptions,
		opts.PredicateChecker,
		opts.ClusterSnapshot,
		opts.AutoscalingKubeClients,
		opts.Processors,
		opts.CloudProvider,
		opts.ExpanderStrategy,
		opts.EstimatorBuilder,
		opts.Backoff,
		opts.DebuggingSnapshotter), nil
}
```

##### initializeDefaultOptions

初始化的内容有：

- Processor：各种前置处理器
- PredicateChecker：扩容前的预调度检查
- CloudProvider：前面介绍过，主要用于操作 IaaS 云资源
- Estimator：评估扩容节点
- Expander：从多个符合扩容条件的 NodeGroup 中选择最终 Node 的策略

```go
func initializeDefaultOptions(opts *AutoscalerOptions) error {
	// 初始化 processor
  if opts.Processors == nil {
		opts.Processors = ca_processors.DefaultProcessors()
	}
	if opts.AutoscalingKubeClients == nil {
		opts.AutoscalingKubeClients = context.NewAutoscalingKubeClients(opts.AutoscalingOptions, opts.KubeClient, opts.EventsKubeClient)
	}
  // 初始化前置校验
	if opts.PredicateChecker == nil {
		predicateCheckerStopChannel := make(chan struct{})
		predicateChecker, err := simulator.NewSchedulerBasedPredicateChecker(opts.KubeClient, predicateCheckerStopChannel)
		if err != nil {
			return err
		}
		opts.PredicateChecker = predicateChecker
	}
  // 初始化快照
	if opts.ClusterSnapshot == nil {
		opts.ClusterSnapshot = simulator.NewBasicClusterSnapshot()
	}
  // 初始化 provider
	if opts.CloudProvider == nil {
		opts.CloudProvider = cloudBuilder.NewCloudProvider(opts.AutoscalingOptions)
	}
  // 初始化 expander 策略
	if opts.ExpanderStrategy == nil {
		expanderStrategy, err := factory.ExpanderStrategyFromStrings(strings.Split(opts.ExpanderNames, ","), opts.CloudProvider,
			opts.AutoscalingKubeClients, opts.KubeClient, opts.ConfigNamespace, opts.GRPCExpanderCert, opts.GRPCExpanderURL)
		if err != nil {
			return err
		}
		opts.ExpanderStrategy = expanderStrategy
	}
  // 初始化 Estimate 策略
	if opts.EstimatorBuilder == nil {
		estimatorBuilder, err := estimator.NewEstimatorBuilder(opts.EstimatorName, estimator.NewThresholdBasedEstimationLimiter(opts.MaxNodesPerScaleUp, opts.MaxNodeGroupBinpackingDuration))
		if err != nil {
			return err
		}
		opts.EstimatorBuilder = estimatorBuilder
	}
  // 初始化 Backoff 策略
	if opts.Backoff == nil {
		opts.Backoff =
			backoff.NewIdBasedExponentialBackoff(opts.InitialNodeGroupBackoffDuration, opts.MaxNodeGroupBackoffDuration, opts.NodeGroupBackoffResetTimeout)
	}

	return nil
}
```

#### 3.3.3 Autoscaler.Start

定时从 cloud provider 获取 node group 以及 node group 下的 instance 信息，并刷新本地缓存

```go
func (a *StaticAutoscaler) Start() error {
	a.clusterStateRegistry.Start()
	return nil
}

func (csr *ClusterStateRegistry) Start() {
	csr.cloudProviderNodeInstancesCache.Start(csr.interrupt)
}

// 后台定时刷新数据
func (cache *CloudProviderNodeInstancesCache) Start(interrupt chan struct{}) {
	go wait.Until(func() {
		cache.Refresh()
	}, CloudProviderNodeInstancesCacheRefreshInterval, interrupt)
}
```

##### Refresh

```go
// Refresh refreshes cache.
func (cache *CloudProviderNodeInstancesCache) Refresh() {
	// 从 cloud provider 获取 node group
  // 调用 cloud provider 的第一个扩展点
	nodeGroups := cache.cloudProvider.NodeGroups()
  // 移除不存在的 node group
	cache.removeEntriesForNonExistingNodeGroupsLocked(nodeGroups)
	for _, nodeGroup := range nodeGroups {
    // 调用 cloud provider 中 node group 接口扩展点
		nodeGroupInstances, err := nodeGroup.Nodes()
		// 更新缓存中的 node group
		cache.updateCacheEntryLocked(nodeGroup, &cloudProviderNodeInstancesCacheEntry{nodeGroupInstances, time.Now()})
	}
}
```

#### 3.3.4 Autoscaler.RunOnce

关键逻辑：

- 获取现有集群所有 Node，以及Node上运行的Pod信息
- 经过几个 Processor 模块处理，将 Node 和 Pod 做分类规整到所属 asgCache 中的 各个asg 下，保存在 nodeInfosForGroups 中
- 获取所有资源不足导致 pending 的 pod
- 经过几个 Processor 模块处理，将未调度 pod 做处理，保存在 unschedulablePodsToHelp 中
- 根据 unschedulablePodsToHelp 判断当前是否需要执行 ScaleUp 进行扩容
- 根据是否配置了缩容参数ScaleDownEnabled，判断是否要进行缩容

核心逻辑伪代码

```go
func (a *StaticAutoscaler) RunOnce() {
  // 获取节点信息
  allNodes, readyNodes := a.obtainNodeLists(a.CloudProvider)
  // 将 Node 信息按照 asg 的id做分类规整
  nodeInfosForGroups := a.processors.TemplateNodeInfoProvider.Process(...)
  // 获取未调度的 pod
  unschedulablePods, err := unschedulablePodLister.List()
  // pod按调度类型分类
  unschedulablePodsToHelp := a.processors.PodListProcessor.Process(unschedulablePods)
  
  if len(unschedulablePodsToHelp) == 0 {
		// 不需要扩容
	} else if a.MaxNodesTotal > 0 && len(readyNodes) >= a.MaxNodesTotal {
		// 扩容达到上限
	} else if allPodsAreNew(unschedulablePodsToHelp, currentTime) {
		// Node 扩容过程中，pod 新创建，等待一定冷却周期再尝试扩容
	} else {
    // 启动扩容
		ScaleUp()
  }
  
  // 如果开启缩容
  if a.ScaleDownEnabled {
    // 缩容逻辑
  }
}
```

RunOnce  实现细节如下：

```go
func (a *StaticAutoscaler) RunOnce(currentTime time.Time) errors.AutoscalerError {
 	...
  // 初始化获取未调度 pod的对象
  unschedulablePodLister := a.UnschedulablePodLister()

  // 获取所有的 node、ready 的node
  allNodes, readyNodes, typedErr := a.obtainNodeLists(a.CloudProvider)
  originalScheduledPods, err := scheduledPodLister.List()
  // 计算集群资源，获取 node.Status.Capacity[resource] 的值
  coresTotal, memoryTotal := calculateCoresMemoryTotal(allNodes, currentTime)

  // 获取 ds 相关pod，后期加入调度器参与模拟调度
	daemonsets, err := a.ListerRegistry.DaemonSetLister().List(labels.Everything())
	// 手动刷新云资源，保持与本地缓存同步
  err = a.AutoscalingContext.CloudProvider.Refresh()
  
  // 找到 pod.Spec.Priority 值小于设定值 ExpendablePodsPriorityCutoff 的 pod
  // 这些 pod 优先级高，不可以被 expend
  nonExpendableScheduledPods := core_utils.FilterOutExpendablePods(originalScheduledPods, a.ExpendablePodsPriorityCutoff)

  // TemplateNodeInfo
  // 将所有运行的 pod，按照不同的 node分类存储好，构造出 NodeInfo 对象，为后续调度准备数据
  // 取出 pod.Spec.NodeName, 依次存储好
  // 依次调用了 
  // 1. MixedTemplateNodeInfoProvider
  // 2. AnnotationNodeInfoProvider
  nodeInfosForGroups, autoscalerError := a.processors.TemplateNodeInfoProvider.Process(autoscalingContext, readyNodes, daemonsets, a.ignoredTaints, currentTime)

  // NodeInfoProcessor
	nodeInfosForGroups, err = a.processors.NodeInfoProcessor.Process(autoscalingContext, nodeInfosForGroups)

  // 获取未注册的 node（在 CA node group 中，但是未注册到 k8s）
  unregisteredNodes := a.clusterStateRegistry.GetUnregisteredNodes()
  if len(unregisteredNodes) > 0 {
		// 删除这些 node
		removedAny, err := removeOldUnregisteredNodes(unregisteredNodes, autoscalingContext,
			a.clusterStateRegistry, currentTime, autoscalingContext.LogRecorder)
	}
  danglingNodes, err := a.deleteCreatedNodesWithErrors()
  
  // 调整 node group size
  fixedSomething, err := fixNodeGroupSize(autoscalingContext, a.clusterStateRegistry, currentTime)

  // 获取未调度 pod
  // 未调度 pod 的排查规则：
  // selector := fields.ParseSelectorOrDie("spec.nodeName==" + "" + ",status.phase!=" +
	//	string(apiv1.PodSucceeded) + ",status.phase!=" + string(apiv1.PodFailed))
  unschedulablePods, err := unschedulablePodLister.List()

  unschedulablePodsToHelp, _ := a.processors.PodListProcessor.Process(a.AutoscalingContext, unschedulablePods)
	unschedulablePodsToHelp = a.filterOutYoungPods(unschedulablePodsToHelp, currentTime)
  
  // 根据未调度的 pod 数量，判断是否需要扩容
	if len(unschedulablePodsToHelp) == 0 {
    // 没有未调度的 pod，不需要扩容
		scaleUpStatus.Result = status.ScaleUpNotNeeded
		klog.V(1).Info("No unschedulable pods")
	} else if a.MaxNodesTotal > 0 && len(readyNodes) >= a.MaxNodesTotal {
    /// 已经达到扩容上限
		scaleUpStatus.Result = status.ScaleUpNoOptionsAvailable
		klog.V(1).Info("Max total nodes in cluster reached")
	} else if allPodsAreNew(unschedulablePodsToHelp, currentTime) {
    // 大量 pod 同时创建，一段时间内不再触发扩容，处于冷却期
		a.processorCallbacks.DisableScaleDownForLoop()
		scaleUpStatus.Result = status.ScaleUpInCooldown
		klog.V(1).Info("Unschedulable pods are very new, waiting one iteration for more")
	} else {
		scaleUpStart := time.Now()
		metrics.UpdateLastTime(metrics.ScaleUp, scaleUpStart)
		// 真正执行扩容动作
		scaleUpStatus, typedErr = ScaleUp(autoscalingContext, a.processors, a.clusterStateRegistry, unschedulablePodsToHelp, readyNodes, daemonsets, nodeInfosForGroups, a.ignoredTaints)
	}
}
```

##### MixedTemplateNodeInfoProvider

将所有的 Node，以及 Node 上运行的 pod，按照 asg 的 id归类保存

```go
func (p *MixedTemplateNodeInfoProvider) Process(ctx *context.AutoscalingContext, nodes []*apiv1.Node, daemonsets []*appsv1.DaemonSet, ignoredTaints taints.TaintKeySet, now time.Time) (map[string]*schedulerframework.NodeInfo, errors.AutoscalerError) {
	...
  // 获取 node 上运行的所有的 pod，key是 node-name，value 是 pod 列表
	podsForNodes, err := getPodsForNodes(ctx.ListerRegistry)
  
  // 定义一个回调函数，处理 node 节点
	processNode := func(node *apiv1.Node) (bool, string, errors.AutoscalerError) {
    // 根据 node信息，调用 clouder provider 扩展点，获取 node group 信息
    // aws: 根据 node.Spec.ProviderID 调用 aws-sdk 获取
		nodeGroup, err := ctx.CloudProvider.NodeGroupForNode(node)
		// 得到 node group 的 id
		id := nodeGroup.Id()
		if _, found := result[id]; !found {
			// 根据给定的node，构造节点信息，确认是否是需要创建的 node
      // getRequiredPodsForNode：将 node 上的 dameonset pod 单独取出来（新节点也必须要运行这些 pod）
      // schedulerframework.NewNodeInfo: 构造新的 node 信息，都是调度框架的函数
      //    pInfo.Update(pod)：计算 pod 的亲和性信息
      //    n.AddPodInfo(...)：计算 cpu、memory、端口占用、pvc 引用等信息
			nodeInfo, err := simulator.BuildNodeInfoForNode(node, podsForNodes)
			if err != nil {
				return false, "", err
			}
      // 修改从 template 中生成的 node 信息，避免使用了重复的主机名
      //    sanitizeTemplateNode：根据 node group 规则自动生成主机名、新增 trait 信息
      //    schedulerframework.NewNodeInfo：更新完主机信息后，再次调用调度框架
			sanitizedNodeInfo, err := utils.SanitizeNodeInfo(nodeInfo, id, ignoredTaints)
			
			result[id] = sanitizedNodeInfo
			return true, id, nil
		}
		return false, "", nil
	}
  
  // 从 Node Group 中扩展新的节点
	for _, nodeGroup := range ctx.CloudProvider.NodeGroups() {
    // nodeGroup.TemplateNodeInfo() 获取 节点模板
    // daemonset.GetDaemonSetPodsForNode: 根据 ds 和 node 返回 pod
    //  schedulerframework.NewNodeInfo：构造完整的 pod
  	nodeInfo, err := utils.GetNodeInfoFromTemplate(nodeGroup, daemonsets, ctx.PredicateChecker, ignoredTaints)
		result[id] = nodeInfo
  }
}
```

##### AnnotationNodeInfoProvider

从 asg 中获取注解信息，并追加到 NodeInfo 中，便于后续参与调度

```go
func (p *AnnotationNodeInfoProvider) Process(ctx *context.AutoscalingContext, nodes []*apiv1.Node, daemonsets []*appsv1.DaemonSet, ignoredTaints taints.TaintKeySet, currentTime time.Time) (map[string]*schedulerframework.NodeInfo, errors.AutoscalerError) {
  // 先经过 mixedTemplateNodeInfoProvider 处理
	nodeInfos, err := p.mixedTemplateNodeInfoProvider.Process(ctx, nodes, daemonsets, ignoredTaints, currentTime)
	
	for _, nodeInfo := range nodeInfos {
    // 拿到所有的 node group
		nodeGroup, err := ctx.CloudProvider.NodeGroupForNode(nodeInfo.Node())
		// 获取  node  group 模板
		template, err := nodeGroup.TemplateNodeInfo()
		// 获取模板 Annotation 信息
		for key, val := range template.Node().Annotations {
			if _, ok := nodeInfo.Node().Annotations[key]; !ok {
        // 将模板 annotation 添加到 node 上
				nodeInfo.Node().Annotations[key] = val
			}
		}
	}
	return nodeInfos, nil
}

```

### 3.4 ScaleUp

- 找到 cloud provider 所有可用的 node group
- 把不可调度的 pod 按照扩容需求进行分组
- 调用
- 将前两步得到的数据作为输入，传给 estimator 模块的装箱算法，得到候选的 pod、node 分配方案
- 将上一步得到的结果，传给 expander 模块，得到最优的分配方案。默认提供好几种策略

伪代码实现关键步骤：

```go
// 前面的动作，将集群所有 Node 和 Pod 做规整，构造 NodeInfo 信息，按 nodeGroupId 建立索引
nodeInfosForGroups, autoscalerError := a.processors.TemplateNodeInfoProvider.Process(...)

func ScaleUp(...) {
  // 获取所有可用的 node group
 	nodeGroups := context.CloudProvider.NodeGroups()
  // 将所有待调度的 pod 按照调度属性分类
  podEquivalenceGroups := buildPodEquivalenceGroups(unschedulablePods)
  expansionOptions := make(map[string]expander.Option, 0)
  // 遍历所有的 node group
  for _, nodeGroup := range nodeGroups {
    // 获取当前 node group 的 nodeInfo 信息
    nodeInfo, found := nodeInfos[nodeGroup.Id()]
    // 计算当前 asg 能够提供的cpu、memory 资源量，确认是否超过限额
    scaleUpResourcesDelta := computeScaleUpResourcesDelta(nodeInfo, nodeGroup, resourceLimiter)
    // 调用 Extimate 模块背包算法，计算出当前 node group 下需要扩展几台 Node，能满足哪些 pod 调度，保存在 option 变量中 
    option, err := computeExpansionOption(podEquivalenceGroups, nodeGroup, nodeInfo, upcomingNodes)
    // 将该 NodeGroup 扩容的情况保存起来
    expansionOptions[nodeGroup.Id()] = option
  }
  // 计算结果重命名为 options
  // 此时有多个满足条件的结果
  options := expansionOptions
  // 调用 Expander 模块，根据启动传入的策略参数，从多个选项中选择最终一个结果
  bestOption := context.ExpanderStrategy.BestOption(options, nodeInfos)
  
  // 如果 NodeGroup 不存在，创建 NodeGroup
  processors.NodeGroupManager.CreateNodeGroup(context, bestOption.NodeGroup)
  
  // 负载均衡策略计算多个 NodeGroup中各自需要扩容的信息
  scaleUpInfos, typedErr := processors.NodeGroupSetProcessor.BalanceScaleUpBetweenGroups()
  for _, info := range scaleUpInfos {
    // 调用 provider 执行扩容
    executeScaleUp(...)
    // 负载均衡
    clusterStateRegistry.Recalculate()
  }
}
```

ScaleUp实现细节如下：

```go
func ScaleUp(...) {
  ...
  // 第一步：找到 cloud provider 所有可用的 node group
  // 返回 node 列表中不在 node group 中的 node 子集
  nodesFromNotAutoscaledGroups, err := utils.FilterOutNodesFromNotAutoscaledGroups(nodes, context.CloudProvider)
  
  // 计算扩容可用的剩余资源
  //  calculateScaleUpCoresMemoryTotal：计算 node group 和非 Node group 所有的资源
  //  sum(nodeGroup.targetSize * nodeInfo.cpu(memory) )
  // computeBelowMax(totalCores, max)：根据 CA 配置的资源限额 - 目前所有已使用资源 = 可扩容的资源余量
	scaleUpResourcesLeft, errLimits := computeScaleUpResourcesLeftLimits(context, processors, nodeGroups, nodeInfos, nodesFromNotAutoscaledGroups, resourceLimiter)

  // Node在NodeGroup中但是没有Registed在kubenetes集群
  // 数量为 newNodes := ar.CurrentTarget - (readiness.Ready + readiness.Unready + readiness.LongUnregistered)
  upcomingNodes := make([]*schedulerframework.NodeInfo, 0)
	for nodeGroup, numberOfNodes := range clusterStateRegistry.GetUpcomingNodes() {
    
  }
  
  processors.NodeGroupListProcessor.Process(...)
  
  // 第二步：将所有待调度 pod 按照扩容需求分类
  podEquivalenceGroups := buildPodEquivalenceGroups(unschedulablePods)
  
  // 评估所有的 node group，哪些不可用跳过，哪些可用
  skippedNodeGroups := map[string]status.Reasons{}
  
  // 外层循环，遍历所有的 NodeGroup
	for _, nodeGroup := range nodeGroups {
    if nodeGroup.Exist() && !clusterStateRegistry.IsNodeGroupSafeToScaleUp(nodeGroup, now) {
      ...
    }
    // 取出当前大小
    currentTargetSize, err := nodeGroup.TargetSize()
    
    // 计算扩容需要的增量资源
    // 取出 node group 下对应的 cpu、memory 信息
    scaleUpResourcesDelta, err := computeScaleUpResourcesDelta(context, processors, nodeInfo, nodeGroup, resourceLimiter)
    
    // 校验是否超过限额，对比可扩容量和待扩容量
    checkResult := scaleUpResourcesLeft.checkScaleUpDeltaWithinLimits(scaleUpResourcesDelta)

    // 第三步：将前两步得到的数据作为输入，传给 estimator 模块的装箱算法，得到候选的 pod、node 分配方案
    option, err := computeExpansionOption(context, podEquivalenceGroups, nodeGroup, nodeInfo, upcomingNodes)
  }

  // 第四步：将上一步得到的结果，传给 expander 模块，得到最优的分配方案
  //  根据expansion (random ,mostpods, price,waste)配置获取最佳伸缩组
  // expander 是表示选择 node group 的策略
  bestOption := context.ExpanderStrategy.BestOption(options, nodeInfos)
  if bestOption != nil && bestOption.NodeCount > 0 {
    
    // 得到需要扩容的节点数
    newNodes := bestOption.NodeCount
    // 判断是否达到扩容上限
    if context.MaxNodesTotal > 0 && len(nodes)+newNodes+len(upcomingNodes) > context.MaxNodesTotal {
    }
    
    // 不存在 node group，创建新的
    if !bestOption.NodeGroup.Exist() {
      // 创建的 ng 包括主的、扩展的
      createNodeGroupResult, err := processors.NodeGroupManager.CreateNodeGroup(context, bestOption.NodeGroup)
      
      // 根据主 ng 创建 nodeinfo
      // 将 daemonset 追加到到 node group 模板创建出来的 node节点 pod 列表中
      // trait 信息追加到新创建 node 的 Spec.Trait 中
      // 填充 node name
      mainCreatedNodeInfo, err := utils.GetNodeInfoFromTemplate(createNodeGroupResult.MainCreatedNodeGroup, daemonSets, context.PredicateChecker, ignoredTaints)
      // 依次创建多个扩展的 ng
			for _, nodeGroup := range createNodeGroupResult.ExtraCreatedNodeGroups {
        option, err2 := computeExpansionOption(context, podEquivalenceGroups, nodeGroup, nodeInfo, upcomingNodes)

      }
      
      // 重新计算缓存中节点信息
      clusterStateRegistry.Recalculate()
      
       // 计算出要扩容的资源
  		newNodes, err = applyScaleUpResourcesLimits(context, processors, newNodes, scaleUpResourcesLeft, nodeInfo, bestOption.NodeGroup, resourceLimiter)
    
      if context.BalanceSimilarNodeGroups {
        // 找到相似的 ng
				similarNodeGroups, typedErr := processors.NodeGroupSetProcessor.FindSimilarNodeGroups(context, bestOption.NodeGroup, nodeInfos)
      }
      
      // 平衡多个 ng
      scaleUpInfos, typedErr := processors.NodeGroupSetProcessor.BalanceScaleUpBetweenGroups(
			context, targetNodeGroups, newNodes)
      
      // 依次遍历所有待扩容的机器，执行扩容操作
      for _, info := range scaleUpInfos {
        // executeScaleUp 会执行 clouder provider 中的 IncreaseSize 方法
			typedErr := executeScaleUp(context, clusterStateRegistry, info, gpu.GetGpuTypeForMetrics(gpuLabel, availableGPUTypes, nodeInfo.Node(), nil), now)
			}
      
     	clusterStateRegistry.Recalculate()
      // 返回扩容成功
			return &status.ScaleUpStatus{
        Result:                  status.ScaleUpSuccessful,
        ScaleUpInfos:            scaleUpInfos,
        PodsRemainUnschedulable: getRemainingPods(podEquivalenceGroups, skippedNodeGroups),
        ConsideredNodeGroups:    nodeGroups,
        CreateNodeGroupResults:  createNodeGroupResults,
        PodsTriggeredScaleUp:    bestOption.Pods,
        PodsAwaitEvaluation:     getPodsAwaitingEvaluation(podEquivalenceGroups, bestOption.NodeGroup.Id()),
      }, nil
    }
    // 返回不需要扩容
    return &status.ScaleUpStatus{
      Result:                  status.ScaleUpNoOptionsAvailable,
      PodsRemainUnschedulable: getRemainingPods(podEquivalenceGroups, skippedNodeGroups),
      ConsideredNodeGroups:    nodeGroups,
    }, nil
}
```

#### 3.4.1 ScaleUpInfo

计算出来的 ScaleUpInfo 会传给 Clouder Provider，用于操作云资源

需要开通的资源数量 delta = ScaleUpInfo.NewSize - ScaleUpInfo.CurrentSize

```go
type ScaleUpInfo struct {
	// Group is the group to be scaled-up
	Group cloudprovider.NodeGroup
	// CurrentSize is the current size of the Group
	CurrentSize int
	// NewSize is the size the Group will be scaled-up to
	NewSize int
	// MaxSize is the maximum allowed size of the Group
	MaxSize int
}
```

#### 3.4.2 computeExpansionOption

这里分为两大块逻辑：预检查 + 背包计算

预检查：

- 遍历所有待调度pod
- 每个 pod 依次去和当前 Node 做预调度，确认一个 Node扩容后可以让 该pod 调度成功
- 将初步筛选出来满足条件的 pod 列表，

背包计算：

通过前面的计算：所有的 pod中，如果扩容一个Node，一定能满足调度条件；但是到底要创建最少多少个 Node，能满足所有的 pod 调度需求，就要经过 Estimate 模块的背包算法了

- 通过给定多个 Pod 和多个 Node，计算出最优的 Node 和 Pod 数量

```go
	func computeExpansionOption(...) {
    // 遍历每个待调度的 pod
    // 用每个 pod 去匹配 node，做模拟调度
		for _, eg := range podEquivalenceGroups {
      // 校验调度
      // 内部调用调度框架
      // p.framework.RunPreFilterPlugins(context.TODO(), state, pod)
      // 	filterStatuses := p.framework.RunFilterPlugins(context.TODO(), state, pod, nodeInfo)
      // 确认调度状态是否正确
    	if err := context.PredicateChecker.CheckPredicates(context.ClusterSnapshot, samplePod, nodeInfo.Node().Name); err == nil {
			// 返回可以调度的 pod
			option.Pods = append(option.Pods, eg.pods...)
      // 标记 pod 理论上可以调度成功
			eg.schedulable = true
    }
      
    // 可以成功调度 pod > 0，开始仿真调度
    // 计算需要的 node 数，和可以成功调度的 pod 数
    if len(option.Pods) > 0 {
			estimator := context.EstimatorBuilder(context.PredicateChecker, context.ClusterSnapshot)
      // 调用 Estimate 模块，后面单独介绍
			option.NodeCount, option.Pods = estimator.Estimate(option.Pods, nodeInfo, option.NodeGroup)
		}

		return option, nil
  }

```

### 3.5 Estimate

前面介绍过，通过上述计算后，所有的 pod中，如果扩容一个Node，一定能满足调度条件；但是到底要创建最少多少个 Node，能满足所有的 pod 调度需求，就要经过 Estimate 模块的背包算法了

#### 3.5.1 优化目标

通过给定多个 Pod 和多个 Node，计算出最优的 Node 和 Pod 数量

##### 输入

- 待调度 pod 列表
- nodeinfo
- NodeGroup

##### 接口

```go
type Estimator interface {
	Estimate([]*apiv1.Pod, *schedulerframework.NodeInfo, cloudprovider.NodeGroup) (int, []*apiv1.Pod)
}
```

##### 输出

- 节点数量
- pod列表

#### 3.5.2 实现分析

- 将这组 pod 所需的cpu、memory 资源 / Node节点能提供的资源，计算出每个 pod 的得分
- 按照得分从高到底排序
- 按照由高到低得分顺序，依次遍历每个 pod，去匹配 Node
- 新创建的 Node 保存到 newNodeNames 数组中
- 如果没有找到，就去创建一个新的 Node。直到所有的 pod 都处理完。

```go
func (e *BinpackingNodeEstimator) Estimate(
  // 计算 pod 的得分
	podInfos := calculatePodScore(pods, nodeTemplate)
	// 按照得分排序
	sort.Slice(podInfos, func(i, j int) bool { return podInfos[i].score > podInfos[j].score })
	// 遍历所有的 pod
  for _, podInfo := range podInfos {
		found := false
		// 确认给定的 pod 是否能调度到 给定的 node 上
    // 内部调用调度框架的 preFilter 依次跟每个 node 过滤一遍， p.framework.RunPreFilterPlugins
		nodeName, err := e.predicateChecker.FitsAnyNodeMatching(e.clusterSnapshot, podInfo.pod, func(nodeInfo *schedulerframework.NodeInfo) bool {
			return newNodeNames[nodeInfo.Node().Name]
		})
		if err == nil {
      // 为 pod 找到合适的 node
			found = true
			scheduledPods = append(scheduledPods, podInfo.pod)
			newNodesWithPods[nodeName] = true
		}

		if !found {
			if lastNodeName != "" && !newNodesWithPods[lastNodeName] {
				continue
			}

      // 添加一个新的 node 进来
			newNodeName, err := e.addNewNodeToSnapshot(nodeTemplate, newNodeNameIndex)
			
			newNodeNameIndex++
			newNodeNames[newNodeName] = true
			lastNodeName = newNodeName

      // 再次尝试调度
      // 如果还是失败：比如设置了 pod 拓扑分布，这种情况无法解决 pending 问题，尝试移除这类 pod
			if err := e.predicateChecker.CheckPredicates(e.clusterSnapshot, podInfo.pod, newNodeName); err != nil {
				continue
			}
			if err := e.clusterSnapshot.AddPod(podInfo.pod, newNodeName); err != nil {
				klog.Errorf("Error adding pod %v.%v to node %v in ClusterSnapshot; %v", podInfo.pod.Namespace, podInfo.pod.Name, newNodeName, err)
				return 0, nil
			}
			newNodesWithPods[newNodeName] = true
			scheduledPods = append(scheduledPods, podInfo.pod)
		}
	}
	return len(newNodesWithPods), scheduledPods
}
```



### 3.6 Expander 策略

#### 3.6.1 概述

通过前面的处理，会返回多个 Option 对象，即有多个可选的组合可以满足此次调度需求（可能只是部分 pod，不是全部pod），

Expander 提供多种策略，在这一组答案中最终选择一个作为最终答案。

选择要扩展的节点组提供的不同策略，通过 --expander=least-waste 参数指定，可以多个策略组合

##### 输入

给定多个 option，选择一个最合适的 option。每个 option 对应一个 NodeGroup、需要调度的 pod、Node数量

```go
// Option describes an option to expand the cluster.
type Option struct {
	NodeGroup cloudprovider.NodeGroup
	NodeCount int
	Debug     string
	Pods      []*apiv1.Pod
}
```

##### 接口

```go
// Strategy describes an interface for selecting the best option when scaling up
type Strategy interface {
	BestOption(options []Option, nodeInfo map[string]*schedulerframework.NodeInfo) *Option
}
```

##### 输出

最终选定的 Option，即：扩容哪个 NodeGroup、扩容该 NodeGroup 的几台机器

#### 3.6.2 实现分析

##### 策略初始化

```go
func ExpanderStrategyFromStrings(...) {
		...
    // 根据不同的策略，使用不同的 Filter
		switch expanderFlag {
		case expander.RandomExpanderName:
			filters = append(filters, random.NewFilter())
		case expander.MostPodsExpanderName:
			filters = append(filters, mostpods.NewFilter())
		case expander.LeastWasteExpanderName:
			filters = append(filters, waste.NewFilter())
		case expander.PriceBasedExpanderName:
			if _, err := cloudProvider.Pricing(); err != nil {
				return nil, err
			}
			filters = append(filters, price.NewFilter(cloudProvider,
				price.NewSimplePreferredNodeProvider(autoscalingKubeClients.AllNodeLister()),
				price.SimpleNodeUnfitness))
		case expander.PriorityBasedExpanderName:
			// It seems other listers do the same here - they never receive the termination msg on the ch.
			// This should be currently OK.
			stopChannel := make(chan struct{})
			lister := kubernetes.NewConfigMapListerForNamespace(kubeClient, stopChannel, configNamespace)
			filters = append(filters, priority.NewFilter(lister.ConfigMaps(configNamespace), autoscalingKubeClients.Recorder))
		case expander.GRPCExpanderName:
			filters = append(filters, grpcplugin.NewFilter(GRPCExpanderCert, GRPCExpanderURL))
		default:
			return nil, errors.NewAutoscalerError(errors.InternalError, "Expander %s not supported", expanderFlag)
		}
		if _, ok := filters[len(filters)-1].(expander.Strategy); ok {
			strategySeen = true
		}
	}
	// 最后追加一个 random 的 fallback
	return newChainStrategy(filters, random.NewStrategy()), nil
}
```

##### 调用策略

```go
func ScaleUp() {
  ...
  bestOption := context.ExpanderStrategy.BestOption(options, nodeInfos)
  ...
}

// 执行策略
func (c *chainStrategy) BestOption(options []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) *expander.Option {
	filteredOptions := options
  // 依次执行所有的 Filter
	for _, filter := range c.filters {
		filteredOptions = filter.BestOptions(filteredOptions, nodeInfo)
		if len(filteredOptions) == 1 {
			return &filteredOptions[0]
		}
	}
	return c.fallback.BestOption(filteredOptions, nodeInfo)
}
```

#### 3.6.3 Filter 接口

Expander 策略是通过多个 Filter 实现的，Filter 定义了统一的接口，和多种实现

接口定义

```go
type Filter interface {
	BestOptions(options []Option, nodeInfo map[string]*schedulerframework.NodeInfo) []Option
}
```

##### leastwaste 实现

- 将所需资源与可用资源计算差值，得到分数
- 取出分数最小的值

```go
func (l *leastwaste) BestOptions(expansionOptions []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) []expander.Option {
	var leastWastedScore float64
	var leastWastedOptions []expander.Option

	for _, option := range expansionOptions {
    // 计算所有 pod 总共需要的 cpu、memory 资源
		requestedCPU, requestedMemory := resourcesForPods(option.Pods)
    // 确认当前的 node group 是否存在
		node, found := nodeInfo[option.NodeGroup.Id()]
		if !found {
      // 不存在就匹配下一个 node group
			klog.Errorf("No node info for: %s", option.NodeGroup.Id())
			continue
		}
		// 找到 Node 能够提供的 cpu、memory 资源
    // cpu = node.Status.Capacity[cpu]
    // memory = node.Status.Capacity[memory]
		nodeCPU, nodeMemory := resourcesForNode(node.Node())
    // 可用资源 = 单节点资源 * nodeGroup数量
		availCPU := nodeCPU.MilliValue() * int64(option.NodeCount)
		availMemory := nodeMemory.Value() * int64(option.NodeCount)
    // 浪费资源 = （可用资源 - 所需资源）/ 可用资源
		wastedCPU := float64(availCPU-requestedCPU.MilliValue()) / float64(availCPU)
		wastedMemory := float64(availMemory-requestedMemory.Value()) / float64(availMemory)
    // 浪费资源数 = cpu浪费 + memory 浪费
		wastedScore := wastedCPU + wastedMemory

		klog.V(1).Infof("Expanding Node Group %s would waste %0.2f%% CPU, %0.2f%% Memory, %0.2f%% Blended\n", option.NodeGroup.Id(), wastedCPU*100.0, wastedMemory*100.0, wastedScore*50.0)

		if wastedScore == leastWastedScore {
			leastWastedOptions = append(leastWastedOptions, option)
		}
		// 取浪费分数最小的选项
		if leastWastedOptions == nil || wastedScore < leastWastedScore {
			leastWastedScore = wastedScore
			leastWastedOptions = []expander.Option{option}
		}
	}

	if len(leastWastedOptions) == 0 {
		return nil
	}

	return leastWastedOptions
}
```

##### mostpods

```go
func (m *mostpods) BestOptions(expansionOptions []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) []expander.Option {
	var maxPods int
	var maxOptions []expander.Option

  // 遍历所有的 option
	for _, option := range expansionOptions {
		if len(option.Pods) == maxPods {
			maxOptions = append(maxOptions, option)
		}

    // 取 pod 数量最大的那个选项
		if len(option.Pods) > maxPods {
			maxPods = len(option.Pods)
			maxOptions = []expander.Option{option}
		}
	}

	if len(maxOptions) == 0 {
		return nil
	}

	return maxOptions
}
```

##### random

```go
func (r *random) BestOptions(expansionOptions []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) []expander.Option {
	// 调用 BestOption
  best := r.BestOption(expansionOptions, nodeInfo)
	if best == nil {
		return nil
	}
	return []expander.Option{*best}
}

func (r *random) BestOption(expansionOptions []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) *expander.Option {
	if len(expansionOptions) <= 0 {
		return nil
	}
	// 从所有备选 option 中随机选择一个
	pos := rand.Int31n(int32(len(expansionOptions)))
	return &expansionOptions[pos]
}
```

##### priority

```go
func (p *priority) BestOptions(expansionOptions []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) []expander.Option {
	
	// 读取名为 cluster-autoscaler-priority-expander，key 为 priorities 的 configmap
  // 将yaml数据转换为 type priorities map[int][]*regexp.Regexp 对象
	priorities, cm, err := p.reloadConfigMap()
	
  // 遍历所有 option
	for _, option := range expansionOptions {
		// 获取 node group 的 id
    id := option.NodeGroup.Id()
		found := false
    // 遍历所有的优先级
		for prio, nameRegexpList := range priorities {
      // 优先级列表中匹配当前的 node group id
      // 匹配不到就跳过
			if !p.groupIDMatchesList(id, nameRegexpList) {
				continue
			}
			found = true
      // 当前优先级低就跳过
			if prio < maxPrio {
				continue
			}
      // 找到优先级最高那个
			if prio > maxPrio {
				maxPrio = prio
				best = nil
			}
			best = append(best, option)

		}
		if !found {
			msg := fmt.Sprintf("Priority expander: node group %s not found in priority expander configuration. "+
				"The group won't be used.", id)
			p.logConfigWarning(cm, "PriorityConfigMapNotMatchedGroup", msg)
		}
	}
	// 优先级失效
	if len(best) == 0 {
		msg := "Priority expander: no priorities info found for any of the expansion options. No options filtered."
		p.logConfigWarning(cm, "PriorityConfigMapNoGroupMatched", msg)
		return expansionOptions
	}

	for _, opt := range best {
		klog.V(2).Infof("priority expander: %s chosen as the highest available", opt.NodeGroup.Id())
	}
	return best
}
```

##### price

选择成本最小的，依赖 cloud provider 的价格模型，aws cloud provider 没有实现，可以不用考虑

```go
// BestOption selects option based on cost and preferred node type.
func (p *priceBased) BestOptions(expansionOptions []expander.Option, nodeInfos map[string]*schedulerframework.NodeInfo) []expander.Option {
	var bestOptions []expander.Option
	....
}
```

##### grpc

```go
func (g *grpcclientstrategy) BestOptions(expansionOptions []expander.Option, nodeInfo map[string]*schedulerframework.NodeInfo) []expander.Option {
	// 判断 grpcClient 参数是否传入
  if g.grpcClient == nil {
		klog.Errorf("Incorrect gRPC client config, filtering no options")
		return expansionOptions
	}
	
  // 调用 grpc 请求
	bestOptionsResponse, err := g.grpcClient.BestOptions(ctx, &protos.BestOptionsRequest{Options: grpcOptionsSlice, NodeMap: grpcNodeMap})
	...
	return options
}

func (c *expanderClient) BestOptions(ctx context.Context, in *BestOptionsRequest, opts ...grpc.CallOption) (*BestOptionsResponse, error) {
	out := new(BestOptionsResponse)
	err := c.cc.Invoke(ctx, "/grpcplugin.Expander/BestOptions", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

grpc对应 proto 文件

```protobuf
// Interface for Expander
service Expander {

  rpc BestOptions (BestOptionsRequest)
    returns (BestOptionsResponse) {}
}
```



### 3.7 缩容实现

#### 3.7.1 缩容概述

缩容和扩容都在同一个定时器中，即默认10s一个检查循环。满足以下所有条件会触发缩容：

- 在改节点上运行的所有 pod 的 cpu、memory的总和 < 节点可分配总额的 50%。（所有参数都可定制）
- 节点上运行的所有 pod（除Daemonset），都可以移动到其他节点（特殊pod可以添加注解禁止CA调度到其他Node）
- Node 没有添加禁用缩减的 Annotation

缩容其他注意事项：

- 一个Node从检查出空闲，持续10min时间内依然空闲，才会被真正移除
- 缩容操作一次之后缩一个，避免不可预期的错误

关键源码实现：

```go
func (a *StaticAutoscaler) RunOnce(...){
  // 缩容逻辑
  if a.ScaleDownEnabled {
    // 获取可能将被删除的 Node 列表，只是初步判断 Node 所在 ASG 实例数是否缩容到最小了
    scaleDownCandidates := GetScaleDownCandidates()
    // 返回某个 Node 被删除后，可能容纳node上 pod 的Node，默认返回所有 nodes
    podDestinations := GetPodDestinationCandidates()
    
    // 关键方法，通过各个维度统计出不再需要的 Node
    // 更新不再需要的 Node 信息，保存在 scaleDown.unneededNodes 中
    scaleDown.UpdateUnneededNodes(podDestinations, scaleDownCandidates)
    
    if scaleDownInCooldown {
      // 缩容冷却中
			scaleDownStatus.Result = status.ScaleDownInCooldown
		} else if scaleDown.nodeDeletionTracker.IsNonEmptyNodeDeleteInProgress() {
      // 正在进行缩容过程中
			scaleDownStatus.Result = status.ScaleDownInProgress
		} else {
    	// 可以开始缩容
      scaleDownStatus := scaleDown.TryToScaleDown(currentTime, pdbs)
    } 
  }
}

```

#### 3.7.2 源码分析

```go
func (a *StaticAutoscaler) RunOnce(currentTime time.Time) errors.AutoscalerError {
  // 扩容逻辑，前面已分析
  ...
  // 缩容逻辑
  if a.ScaleDownEnabled {
    // 特殊处理的 pod
		pdbs, err := pdbLister.List()
		
    // 计算不再需要的 node
    // 保存待缩容的候选 Node
		var scaleDownCandidates []*apiv1.Node
    // 保存可以存放被删除 Node上pod的节点
		var podDestinations []*apiv1.Node

		if a.processors == nil || a.processors.ScaleDownNodeProcessor == nil {
			scaleDownCandidates = allNodes
			podDestinations = allNodes
		} else {
			var err errors.AutoscalerError
      // 初步筛选处理
			scaleDownCandidates, err = a.processors.ScaleDownNodeProcessor.GetScaleDownCandidates(
				autoscalingContext, allNodes)
			// pod选择新的 node
			podDestinations, err = a.processors.ScaleDownNodeProcessor.GetPodDestinationCandidates(autoscalingContext, allNodes)
		}

		// We use scheduledPods (not originalScheduledPods) here, so artificial scheduled pods introduced by processors
		// (e.g unscheduled pods with nominated node name) can block scaledown of given node.
		if typedErr := scaleDown.UpdateUnneededNodes(podDestinations, scaleDownCandidates, currentTime, pdbs); typedErr != nil {
			scaleDownStatus.Result = status.ScaleDownError
			klog.Errorf("Failed to scale down: %v", typedErr)
			return typedErr
		}

		metrics.UpdateDurationFromStart(metrics.FindUnneeded, unneededStart)

		if klog.V(4).Enabled() {
			for key, val := range scaleDown.unneededNodes {
				klog.Infof("%s is unneeded since %s duration %s", key, val.String(), currentTime.Sub(val).String())
			}
		}

		scaleDownInCooldown := a.processorCallbacks.disableScaleDownForLoop ||
			a.lastScaleUpTime.Add(a.ScaleDownDelayAfterAdd).After(currentTime) ||
			a.lastScaleDownFailTime.Add(a.ScaleDownDelayAfterFailure).After(currentTime) ||
			a.lastScaleDownDeleteTime.Add(a.ScaleDownDelayAfterDelete).After(currentTime)
		// In dry run only utilization is updated
		calculateUnneededOnly := scaleDownInCooldown || scaleDown.nodeDeletionTracker.IsNonEmptyNodeDeleteInProgress()

		klog.V(4).Infof("Scale down status: unneededOnly=%v lastScaleUpTime=%s "+
			"lastScaleDownDeleteTime=%v lastScaleDownFailTime=%s scaleDownForbidden=%v "+
			"isDeleteInProgress=%v scaleDownInCooldown=%v",
			calculateUnneededOnly, a.lastScaleUpTime,
			a.lastScaleDownDeleteTime, a.lastScaleDownFailTime, a.processorCallbacks.disableScaleDownForLoop,
			scaleDown.nodeDeletionTracker.IsNonEmptyNodeDeleteInProgress(), scaleDownInCooldown)
		metrics.UpdateScaleDownInCooldown(scaleDownInCooldown)

		if scaleDownInCooldown {
			scaleDownStatus.Result = status.ScaleDownInCooldown
		} else if scaleDown.nodeDeletionTracker.IsNonEmptyNodeDeleteInProgress() {
			scaleDownStatus.Result = status.ScaleDownInProgress
		} else {
			klog.V(4).Infof("Starting scale down")

			// We want to delete unneeded Node Groups only if there was no recent scale up,
			// and there is no current delete in progress and there was no recent errors.
			removedNodeGroups, err := a.processors.NodeGroupManager.RemoveUnneededNodeGroups(autoscalingContext)
			if err != nil {
				klog.Errorf("Error while removing unneeded node groups: %v", err)
			}

			scaleDownStart := time.Now()
			metrics.UpdateLastTime(metrics.ScaleDown, scaleDownStart)
      // 开始尝试缩容
			scaleDownStatus, typedErr := scaleDown.TryToScaleDown(currentTime, pdbs)
			metrics.UpdateDurationFromStart(metrics.ScaleDown, scaleDownStart)
			metrics.UpdateUnremovableNodesCount(scaleDown.getUnremovableNodesCount())

			scaleDownStatus.RemovedNodeGroups = removedNodeGroups

			if scaleDownStatus.Result == status.ScaleDownNodeDeleteStarted {
				a.lastScaleDownDeleteTime = currentTime
				a.clusterStateRegistry.Recalculate()
			}

			if (scaleDownStatus.Result == status.ScaleDownNoNodeDeleted ||
				scaleDownStatus.Result == status.ScaleDownNoUnneeded) &&
				a.AutoscalingContext.AutoscalingOptions.MaxBulkSoftTaintCount != 0 {
        // 
				scaleDown.SoftTaintUnneededNodes(allNodes)
			}

			if a.processors != nil && a.processors.ScaleDownStatusProcessor != nil {
				scaleDownStatus.SetUnremovableNodesInfo(scaleDown.unremovableNodeReasons, scaleDown.nodeUtilizationMap, scaleDown.context.CloudProvider)
				a.processors.ScaleDownStatusProcessor.Process(autoscalingContext, scaleDownStatus)
				scaleDownStatusProcessorAlreadyCalled = true
			}

			if typedErr != nil {
				klog.Errorf("Failed to scale down: %v", typedErr)
				a.lastScaleDownFailTime = currentTime
				return typedErr
			}
		}
	}
	return nil
}
```

##### GetScaleDownCandidates

这一步只是判断哪些 Node 节点所在的 ASG 符合要求

```go
func (n *PreFilteringScaleDownNodeProcessor) GetScaleDownCandidates(ctx *context.AutoscalingContext,
	nodes []*apiv1.Node) ([]*apiv1.Node, errors.AutoscalerError) {
	result := make([]*apiv1.Node, 0, len(nodes))

  // 获取每个 asg 当前的实例个数，保存为 map
	nodeGroupSize := utils.GetNodeGroupSizeMap(ctx.CloudProvider)

  // 遍历所有 node
	for _, node := range nodes {
    // 获取当前 node 所属的 asg
		nodeGroup, err := ctx.CloudProvider.NodeGroupForNode(node)
		// 获取当前 asg 的实例数
		size, found := nodeGroupSize[nodeGroup.Id()]
		// 获取 asg 的最小实例数。当前实例数已经最小了，就跳过不再缩容
		if size <= nodeGroup.MinSize() {
			klog.V(1).Infof("Skipping %s - node group min size reached", node.Name)
			continue
		}
    // 追加到结果中
		result = append(result, node)
	}
	return result, nil
}
```

##### GetPodDestinationCandidates

默认返回所有的 Node

```go
func (n *PreFilteringScaleDownNodeProcessor) GetPodDestinationCandidates(ctx *context.AutoscalingContext,
	nodes []*apiv1.Node) ([]*apiv1.Node, errors.AutoscalerError) {
	return nodes, nil
}
```



##### UpdateUnneededNodes

计算不再需要的node，从以下维度逐一排查：

- 所有的 pod 可以被调度到其他节点
- 资源使用率低于某个阈值
- 其他判断

找到可以移除的节点，放到 unneededNodes 数组中，便于后面移除

```go
// destinationNodes：可以用来安置由于缩容导致被驱逐的pod的节点
// scaleDownCandidates：可以考虑缩容的节点
func (sd *ScaleDown) UpdateUnneededNodes(
	destinationNodes []*apiv1.Node,
	scaleDownCandidates []*apiv1.Node,
	timestamp time.Time,
	pdbs []*policyv1.PodDisruptionBudget,
) errors.AutoscalerError {

  // 第一步：计算节点资源利用率（只计算被管理的节点）
	for _, node := range scaleDownCandidates {
		// 获取节点信息
    nodeInfo, err := sd.context.ClusterSnapshot.NodeInfos().Get(node.Name)
		// 检查节点情况，是否满足缩容
    // 1. 节点是否最近已经被标记为删除，这种节点打了 ToBeDeletedByClusterAutoscaler 的 taint
    // 2. 节点是否有 cluster-autoscaler.kubernetes.io/scale-down-disabled 这个禁止缩容的标签
    // 3. CalculateUtilization 计算资源使用率：累加所有 pod 上容器设置的 cpu、memroy request值
    // 4. isNodeBelowUtilizationThreshold 判断资源使用是否达到阈值（可启动时配置）
		reason, utilInfo := sd.checkNodeUtilization(timestamp, node, nodeInfo)
		
		// 保存可以被删除的节点
		currentlyUnneededNodeNames = append(currentlyUnneededNodeNames, node.Name)
	}

	// 第二步：将候选缩容节点和其他节点分开
	currentCandidates, currentNonCandidates := sd.chooseCandidates(currentlyUnneededNonEmptyNodes)

  // 找到新节点，用于移除候选节点
	nodesToRemove, unremovable, newHints, simulatorErr := simulator.FindNodesToRemove(
		currentCandidates,
		destinations,
		nil,
		sd.context.ClusterSnapshot,
		sd.context.PredicateChecker,
		len(currentCandidates),
		true,
		sd.podLocationHints,
		sd.usageTracker,
		timestamp,
		pdbs)

  //  additionalCandidatesCount 表示用于缩容额外的备选节点数量
	additionalCandidatesCount := sd.context.ScaleDownNonEmptyCandidatesCount - len(nodesToRemove)
	if additionalCandidatesCount > len(currentNonCandidates) {
		additionalCandidatesCount = len(currentNonCandidates)
	}

  // 限制并发缩容数量
	additionalCandidatesPoolSize := int(math.Ceil(float64(len(allNodeInfos)) * sd.context.ScaleDownCandidatesPoolRatio))
	if additionalCandidatesPoolSize < sd.context.ScaleDownCandidatesPoolMinCount {
		additionalCandidatesPoolSize = sd.context.ScaleDownCandidatesPoolMinCount
	}
	if additionalCandidatesPoolSize > len(currentNonCandidates) {
		additionalCandidatesPoolSize = len(currentNonCandidates)
	}
	if additionalCandidatesCount > 0 {
	
    // 找到新节点，用于移除候选节点
		additionalNodesToRemove, additionalUnremovable, additionalNewHints, simulatorErr :=
			simulator.FindNodesToRemove(
				currentNonCandidates[:additionalCandidatesPoolSize],
				destinations,
				nil,
				sd.context.ClusterSnapshot,
				sd.context.PredicateChecker,
				additionalCandidatesCount,
				true,
				sd.podLocationHints,
				sd.usageTracker,
				timestamp,
				pdbs)
	}
  // 将待移除节点保存到 unneededNodes 数组中
  for _, node := range nodesToRemove {
		name := node.Node.Name
		unneededNodesList = append(unneededNodesList, node.Node)
		if val, found := sd.unneededNodes[name]; !found {
			result[name] = timestamp
		} else {
			result[name] = val
		}
	}
	...
}
```

##### TryToScaleDown

```go
func (sd *ScaleDown) TryToScaleDown(
	currentTime time.Time,
	pdbs []*policyv1.PodDisruptionBudget,
) (*status.ScaleDownStatus, errors.AutoscalerError) {

	...
  // 遍历待删除 node 列表
	for nodeName, unneededSince := range sd.unneededNodes {
		
		// 获取 nodeinfo、node 信息
    nodeInfo, err := sd.context.ClusterSnapshot.NodeInfos().Get(nodeName)
		node := nodeInfo.Node()
		// 检查 node 是否打上了禁止删除的 annotation
		if hasNoScaleDownAnnotation(node) {
			klog.V(4).Infof("Skipping %s - scale down disabled annotation found", node.Name)
			sd.addUnremovableNodeReason(node, simulator.ScaleDownDisabledAnnotation)
			continue
		}
		// 获取 node 状态，根据状态做一些处理
		ready, _, _ := kube_util.GetReadinessState(node)
		
    // 计算缩容资源
		scaleDownResourcesDelta, err := sd.computeScaleDownResourcesDelta(sd.context.CloudProvider, node, nodeGroup, resourcesWithLimits)
		// 检查资源限制
		checkResult := scaleDownResourcesLeft.checkScaleDownDeltaWithinLimits(scaleDownResourcesDelta)
		...
		candidateNames = append(candidateNames, node.Name)
		candidateNodeGroups[node.Name] = nodeGroup
	}

  // 寻找一个待移除节点
	nodesToRemove, unremovable, _, err := simulator.FindNodesToRemove(
		candidateNames,
		nodesWithoutMasterNames,
		sd.context.ListerRegistry,
		sd.context.ClusterSnapshot,
		sd.context.PredicateChecker,
		1,
		false,
		sd.podLocationHints,
		sd.usageTracker,
		time.Now(),
		pdbs)

  // 计算时差
	nodeDeletionDuration = time.Now().Sub(nodeDeletionStart)
	sd.nodeDeletionTracker.SetNonEmptyNodeDeleteInProgress(true)

	go func() {
		...
    // 删除节点
		result = sd.deleteNode(toRemove.Node, toRemove.PodsToReschedule, toRemove.DaemonSetPods, nodeGroup)
  }
}
```



## 四、CA 使用注意

[aws官方说明](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/autoscaling.html#ca-deployment-considerations)

### 4.1 asg 自动发现参数配置

- 为 AutoScaling 设置两个标签，便于 CA 自动发现

- 关于跨可用区：

  - 可以设置多个 AutoScaling 组，每个组一个可用区，通过开启--balance-similar-node-groups` 功能。注意：需要为不同的组设置相同的一批标签

  - 也可以设置同一个 AutoScaling 组，但是必须将组设置可跨多个可用区
- 更推荐使用多个 AutoScaling 组

### 4.2 优化节点组：

  - 节点组中的每个节点必须具有相同的调度属性，包括标签、污点和资源
    - 策略中指定的第一个实例类型模拟调度。
    - 如果您的策略具有拥有更多资源的其他实例类型，则在横向扩展后可能会浪费资源。
    - 如果您的策略具有其他实例类型，其资源比原始实例类型少，则 Pod 在实例上调度可能失败。
  - 请使用较多节点配置较少数量的节点组，因为相反的配置可能会对可扩展性产生不利影响。

### 4.3 AutoScaling

- 混合实例策略：支持多个实例类型，配置时推荐使用相似的资源类型：比如：M4`、`M5`、`M5a,` 和 `M5n
- 可以通过 configmap 设置不同 AutoScaling 的优先级
- AutoScaling 的机型也支持权重
- 支持启动配置、启动模板两种模式
- 启动模板里面指定机型
- 启动模板覆盖项支持配置多个机型

### 4.4 Expander 策略

选择要扩展的节点组提供的不同策略，通过 --expander=least-waste 参数指定

可选参数包括：

- random：随机选择

- most-pods：能满足最多 pod 调度的

- Least-waste：最少 cpu 和 memroy

- Price：成本最小

- priority：按用户指定的优先级

- grpc：调用外部 grpc 服务选择扩容节点

  

### 4.5 超额配置

- 通过配置一个空的Deployment，占用资源，如果资源不足优先驱逐，达到尽快扩容的目的

### 4.6 防止pod被驱逐

配置 `cluster-autoscaler.kubernetes.io/safe-to-evict=false 注解，可以确保 pod不被驱逐，pod所在 node 不被缩减
