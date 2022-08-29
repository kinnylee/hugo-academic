# argocd源码分析-util包

## health

### 健康状态评估

[官方说明](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/health.md)

#### Deployment、Replicaset、StatefulSet、DaemonSet

- 实际年代和期望年代一致（generation）
- 更新的副本数和期望的副本数一致（replicas）

#### Service

- 如果type是LoadBalancer，status.loadBalancer.ingress非空（hostname或ip至少一个非空）

#### Ingress

- status.loadBalancer.ingress非空（hostname或ip至少一个非空）

#### PersistentVolumeClaim

- status.phase 是 Bound 状态

#### 自定义资源

有两种方式配置自定义资源健康检查。

- 方式一：在argocd-cm这个configmap中配置
- 方式二：修改源码支持。脚本存放路径：argo-cd/resource_customizations/{group}/{kind}，并提供两个文件
  - health.lua
  - health_test.yaml

### 资源状态

```go
type HealthStatusCode string

const (
	// Indicates that health assessment failed and actual health status is unknown
	HealthStatusUnknown HealthStatusCode = "Unknown"
	// Progressing health status means that resource is not healthy but still have a chance to reach healthy state
	HealthStatusProgressing HealthStatusCode = "Progressing"
	// Resource is 100% healthy
	HealthStatusHealthy HealthStatusCode = "Healthy"
	// Assigned to resources that are suspended or paused. The typical example is a
	// [suspended](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/#suspend) CronJob.
	HealthStatusSuspended HealthStatusCode = "Suspended"
	// Degrade status is used if resource status indicates failure or resource could not reach healthy state
	// within some timeout.
	HealthStatusDegraded HealthStatusCode = "Degraded"
	// Indicates that resource is missing in the cluster.
	HealthStatusMissing HealthStatusCode = "Missing"
)
```

### 获取资源状态

```go
// 获取每个资源的健康状态，写回到status中，以健康状态最糟糕的状态作为应用最终的状态
func SetApplicationHealth(resStatuses []*appv1.ResourceStatus, liveObjs []*unstructured.Unstructured, resourceOverrides map[string]appv1.ResourceOverride, filter func(obj *unstructured.Unstructured) bool) (*appv1.HealthStatus, error) {
	var savedErr error
	appHealth := appv1.HealthStatus{Status: health.HealthStatusHealthy}
	for i, liveObj := range liveObjs {
		var healthStatus *health.HealthStatus
		var err error
		if liveObj == nil {
      // 集群中没有该资源对象，状态就是 Missing
			healthStatus = &health.HealthStatus{Status: health.HealthStatusMissing}
		} else {
      // filter 是个过滤器，可以忽略资源的健康状态
			if filter(liveObj) {
				healthStatus, err = health.GetResourceHealth(liveObj, lua.ResourceHealthOverrides(resourceOverrides))
				if err != nil && savedErr == nil {
					savedErr = err
				}
			}
		}
		if healthStatus != nil {
			resHealth := appv1.HealthStatus{Status: healthStatus.Status, Message: healthStatus.Message}
			resStatuses[i].Health = &resHealth
			ignore := ignoreLiveObjectHealth(liveObj, resHealth)
			if !ignore && health.IsWorse(appHealth.Status, healthStatus.Status) {
				appHealth.Status = healthStatus.Status
			}
		}
	}
	return &appHealth, savedErr
}
```

### 获取k8s资源的健康状态

这个函数是在 gitops-engine 仓库中实现的

```go
func GetResourceHealth(obj *unstructured.Unstructured, healthOverride HealthOverride) (health *HealthStatus, err error) {
  // deletion time 不为空，返回 Progressing 状态
	if obj.GetDeletionTimestamp() != nil {
		return &HealthStatus{
			Status:  HealthStatusProgressing,
			Message: "Pending deletion",
		}, nil
	}

  // 如果自己实现了heatlh健康状态的方法，就用自定义方法，否则采用内置的资源状态检查
	if healthOverride != nil {
    // argocd中自定义资源监控状态是调用 lua 脚本确认资源监控状态
    // 资源健康状态对应的lua脚本路径：{group}/{kind}/health.lua
		health, err := healthOverride.GetResourceHealth(obj)
		if err != nil {
			health = &HealthStatus{
				Status:  HealthStatusUnknown,
				Message: err.Error(),
			}
			return health, err
		}
		if health != nil {
			return health, nil
		}
	}

	gvk := obj.GroupVersionKind()
	switch gvk.Group {
	case "apps":
		switch gvk.Kind {
		case kube.DeploymentKind:
      // deployment 健康状态
			health, err = getDeploymentHealth(obj)
		case kube.StatefulSetKind:
      // statefulset 健康状态
			health, err = getStatefulSetHealth(obj)
		case kube.ReplicaSetKind:
      // replicaset 健康状态
			health, err = getReplicaSetHealth(obj)
		case kube.DaemonSetKind:
      // daemonset 健康状态
			health, err = getDaemonSetHealth(obj)
		}
	case "extensions":
		switch gvk.Kind {
		case kube.DeploymentKind:
			health, err = getDeploymentHealth(obj)
		case kube.IngressKind:
			health, err = getIngressHealth(obj)
		case kube.ReplicaSetKind:
			health, err = getReplicaSetHealth(obj)
		case kube.DaemonSetKind:
			health, err = getDaemonSetHealth(obj)
		}
	case "argoproj.io":
		switch gvk.Kind {
		case "Workflow":
			health, err = getArgoWorkflowHealth(obj)
		}
	case "apiregistration.k8s.io":
		switch gvk.Kind {
		case kube.APIServiceKind:
			health, err = getAPIServiceHealth(obj)
		}
	case "networking.k8s.io":
		switch gvk.Kind {
		case kube.IngressKind:
			health, err = getIngressHealth(obj)
		}
	case "":
		switch gvk.Kind {
		case kube.ServiceKind:
			health, err = getServiceHealth(obj)
		case kube.PersistentVolumeClaimKind:
			health, err = getPVCHealth(obj)
		case kube.PodKind:
			health, err = getPodHealth(obj)
		}
	case "batch":
		switch gvk.Kind {
		case kube.JobKind:
			health, err = getJobHealth(obj)
		}
	case "autoscaling":
		switch gvk.Kind {
		case kube.HorizontalPodAutoscalerKind:
			health, err = getHPAHealth(obj)
		}
	}
	if err != nil {
		health = &HealthStatus{
			Status:  HealthStatusUnknown,
			Message: err.Error(),
		}
	}
	return health, err
}
```

### 调用链

- CompareAppState（argo-cd/controller/state.go)
- SetApplicationHealth

## lua

lua模块扩展了 gitops-engine 中提供的资源健康检查接口GetResourceHealth，实现了自定义资源健康检查

### 健康检查扩展接口

```go
type HealthOverride interface {
	GetResourceHealth(obj *unstructured.Unstructured) (*HealthStatus, error)
}
```

### Lua 实现健康检查接口

```go
func (overrides ResourceHealthOverrides) GetResourceHealth(obj *unstructured.Unstructured) (*health.HealthStatus, error) {
  // 创建 lua 虚拟机
	luaVM := VM{
		ResourceOverrides: overrides,
	}
  // 获取资源对象的健康检查 lua 脚本
	script, err := luaVM.GetHealthScript(obj)
	if err != nil {
		return nil, err
	}
	if script == "" {
		return nil, nil
	}
  // 执行 lua 脚本，获得健康状态
	result, err := luaVM.ExecuteHealthLua(obj, script)
	if err != nil {
		return nil, err
	}
	return result, nil
}
```

### VM 结构体

vm 定义了一个实现了 luaVm 的结构体

```go
type VM struct {
  // 记录所有自定义健康检查的资源
	ResourceOverrides map[string]appv1.ResourceOverride
	// UseOpenLibs flag to enable open libraries. Libraries are always disabled while running, but enabled during testing to allow the use of print statements
	UseOpenLibs bool
}
```

### 获取资源对应的 lua 脚本

```go
func (vm VM) GetHealthScript(obj *unstructured.Unstructured) (string, error) {
  // 获取key，格式为 {group}/{kind}
	key := getConfigMapKey(obj)
  // 从 vm 中获取资源对应的脚本
	if script, ok := vm.ResourceOverrides[key]; ok && script.HealthLua != "" {
		return script.HealthLua, nil
	}
  // 入参为 {group}/{kind}/health.lua
  // 读取这个文件的内容
	return vm.getPredefinedLuaScripts(key, healthScriptFile)
}
```

### 执行lua脚本获取健康状态

```go
func (vm VM) ExecuteHealthLua(obj *unstructured.Unstructured, script string) (*health.HealthStatus, error) {
  // 真正执行 lua 脚本的代码
  l, err := vm.runLua(obj, script)
	if err != nil {
		return nil, err
	}
  // 后续都是对lua脚本结果的处理
	returnValue := l.Get(-1)
	if returnValue.Type() == lua.LTTable {
		jsonBytes, err := luajson.Encode(returnValue)
		if err != nil {
			return nil, err
		}
		healthStatus := &health.HealthStatus{}
		err = json.Unmarshal(jsonBytes, healthStatus)
		if err != nil {
			return nil, err
		}
		if !isValidHealthStatusCode(healthStatus.Status) {
			return &health.HealthStatus{
				Status:  health.HealthStatusUnknown,
				Message: invalidHealthStatus,
			}, nil
		}

		return healthStatus, nil
	}
	return nil, fmt.Errorf(incorrectReturnType, "table", returnValue.Type().String())
}
```

### lua执行引擎

- NewState：初始化
- DoString：开始执行

```go
func (vm VM) runLua(obj *unstructured.Unstructured, script string) (*lua.LState, error) {
	l := lua.NewState(lua.Options{
		SkipOpenLibs: !vm.UseOpenLibs,
	})
	defer l.Close()
	// Opens table library to allow access to functions to manipulate tables
	for _, pair := range []struct {
		n string
		f lua.LGFunction
	}{
		{lua.LoadLibName, lua.OpenPackage},
		{lua.BaseLibName, lua.OpenBase},
		{lua.TabLibName, lua.OpenTable},
		// load our 'safe' version of the os library
		{lua.OsLibName, OpenSafeOs},
	} {
		if err := l.CallByParam(lua.P{
			Fn:      l.NewFunction(pair.f),
			NRet:    0,
			Protect: true,
		}, lua.LString(pair.n)); err != nil {
			panic(err)
		}
	}
	// preload our 'safe' version of the os library. Allows the 'local os = require("os")' to work
	l.PreloadModule(lua.OsLibName, SafeOsLoader)

	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()
	l.SetContext(ctx)
	objectValue := decodeValue(l, obj.Object)
	l.SetGlobal("obj", objectValue)
  // 调用lua库的执行方法
	err := l.DoString(script)
	return l, err
}
```

## helm

helm模块主要关于使用helm进行chart包安装相关逻辑

### helm提供的接口

```go
type Client interface {
	CleanChartCache(chart string, version *semver.Version) error
	ExtractChart(chart string, version *semver.Version) (string, io.Closer, error)
  // 获取给定的 helm 仓库的索引信息
	GetIndex() (*Index, error)
	TestHelmOCI() (bool, error)
}
```

### helm客户端初始化

helmClient有两个地方被使用

```go
func NewClient(repoURL string, creds Creds, enableOci bool) Client {
	return NewClientWithLock(repoURL, creds, globalLock, enableOci)
}
```

#### reposerver-Service初始化

argocd/reposerver/repository/repository.go

```go
func NewService(metricsServer *metrics.MetricsServer, cache *reposervercache.Cache, initConstants RepoServerInitConstants) *Service {
	...
	return &Service{
		...
		newGitClient:              git.NewClient,
     // 初始化 helm 客户端
		newHelmClient: func(repoURL string, creds helm.Creds, enableOci bool) helm.Client {
			return helm.NewClientWithLock(repoURL, creds, sync.NewKeyLock(), enableOci)
		},
		initConstants: initConstants,
		now:           time.Now,
	}
}
```

#### application获取revision

这个方法主要由同步资源时的Sync方法调用，通过gitClient获取给定git地址的最新提交版本信息

argocd/server/application/application.go

```go

func (s *Server) resolveRevision(ctx context.Context, app *appv1.Application, syncReq *application.ApplicationSyncRequest) (string, string, error) {
	...
	if app.Spec.Source.IsHelm() {
		// helm 客户端初始化
		client := helm.NewClient(repo.Repo, repo.GetHelmCreds(), repo.EnableOCI || app.Spec.Source.IsHelmOci())
    // 获取 helm 仓库的 index
    index, err := client.GetIndex()
		...
    // 获取索引中指定的 chart 包信息
		entries, err := index.GetEntries(app.Spec.Source.Chart)
    ...
		// 获取最大的版本信息
		version, err := entries.MaxVersion(constraints)
		...
	} else {
		// git 客户端初始化
		gitClient, err := git.NewClient(repo.Repo, repo.GetGitCreds(), repo.IsInsecure(), repo.IsLFSEnabled())
		...
	}
}
```

### helm获取索引

helm仓库的索引有一定规范，存放在仓库根目录下的index.yaml文件中。通过helm index 命令可以创建或更新这个索引文件。索引内容主要包含所有的chart包，以及chart包的版本信息。具体内容可以查看helm源码分析系列文章

```go
func (c *nativeHelmChart) GetIndex() (*Index, error) {
	start := time.Now()
  // 通过http请求，获取helm仓库路径下 index.yaml 文件内容
	data, err := c.loadRepoIndex()
  ...
	index := &Index{}
  // 将获取的内容反序列化为 Index 对象
	err = yaml.NewDecoder(bytes.NewBuffer(data)).Decode(index)
  ...
}
```

#### 索引存储结构

这里的Entry是helm中的Entry缩减版，完整的索引中包含很多信息，这里只保留了版本和创建时间。完整的Entry可以查看笔者分享的helm源码序列文章。

```go
// 索引信息
type Index struct {
  // chart包集合，key是chart的名称，value是数组
	Entries map[string]Entries
}

// 一个 chart+version 组成唯一的 Entry
type Entries []Entry

type Entry struct {
	Version string
	Created time.Time
}
```

### Helm

helm封装了执行helm命令相关的功能

```go
type Helm interface {
	// Template returns a list of unstructured objects from a `helm template` command
	Template(opts *TemplateOpts) (string, error)
  // 返回参数列表，底层调用flatVals将所有的参数合并。文件支持http地址和本地地址
	GetParameters(valuesFiles []string) (map[string]string, error)
	// DependencyBuild runs `helm dependency build` to download a chart's dependencies
	DependencyBuild() error
  // helm2 需要执行的 helm init --client-only
  // helm3 直接返回
	Init() error
	// Dispose deletes temp resources
	Dispose()
}
```

### NewHelmApp

生成执行helm相关的命令行参数

```go
func NewHelmApp(workDir string, repos []HelmRepository, isLocal bool, version string) (Helm, error) {
	cmd, err := NewCmd(workDir, version)
	if err != nil {
		return nil, err
	}
	cmd.IsLocal = isLocal

	return &helm{repos: repos, cmd: *cmd}, nil
}
```

#### flatVals

这个函数主要是将多个values平铺合并，形成完整的参数。这部分内容helm源码中也有自己的实现逻辑。

```go
func flatVals(input interface{}, output map[string]string, prefixes ...string) {
	switch i := input.(type) {
	case map[string]interface{}:
		for k, v := range i {
			flatVals(v, output, append(prefixes, k)...)
		}
	case []interface{}:
		for j, v := range i {
			flatVals(v, output, append(prefixes[0:len(prefixes)-1], fmt.Sprintf("%s[%v]", prefixes[len(prefixes)-1], j))...)
		}
	default:
		output[strings.Join(prefixes, ".")] = fmt.Sprintf("%v", i)
	}
}
```



## git

git模块主要负责从git上拉取代码

### git提供的接口

```go
type Client interface {
	Root() string
  // 初始化一个本地仓库，对应 git init 命令
	Init() error
  // git fetch
	Fetch(revision string) error
  // get checkout 
	Checkout(revision string) error
	LsRefs() (*Refs, error)
	LsRemote(revision string) (string, error)
	LsFiles(path string) ([]string, error)
	LsLargeFiles() ([]string, error)
	CommitSHA() (string, error)
	RevisionMetadata(revision string) (*RevisionMetadata, error)
	VerifyCommitSignature(string) (string, error)
}
```

### git客户端初始化

git客户端被使用的两个地方，调用的地方同前面介绍的 helm 模块

```go
func NewClient(rawRepoURL string, creds Creds, insecure bool, enableLfs bool) (Client, error) {
	root := filepath.Join(os.TempDir(), strings.Replace(NormalizeGitURL(rawRepoURL), "/", "_", -1))
	if root == os.TempDir() {
		return nil, fmt.Errorf("Repository '%s' cannot be initialized, because its root would be system temp at %s", rawRepoURL, root)
	}
	return NewClientExt(rawRepoURL, root, creds, insecure, enableLfs)
}
```

#### reposerver-Service初始化

argocd/reposerver/repository/repository.go

```go
func NewService(metricsServer *metrics.MetricsServer, cache *reposervercache.Cache, initConstants RepoServerInitConstants) *Service {
	...
	return &Service{
		parallelismLimitSemaphore: parallelismLimitSemaphore,
		repoLock:                  repoLock,
		cache:                     cache,
		metricsServer:             metricsServer,
    // 初始化 git 客户端
		newGitClient:              git.NewClient,
		newHelmClient: func(repoURL string, creds helm.Creds, enableOci bool) helm.Client {
			return helm.NewClientWithLock(repoURL, creds, sync.NewKeyLock(), enableOci)
		},
		initConstants: initConstants,
		now:           time.Now,
	}
}
```

#### application获取revision

这里同前面介绍的 helmClient

argocd/server/application/application.go

```go
func (s *Server) resolveRevision(ctx context.Context, app *appv1.Application, syncReq *application.ApplicationSyncRequest) (string, string, error) {
	...
	if app.Spec.Source.IsHelm() {
		// helm 客户端初始化
		client := helm.NewClient(repo.Repo, repo.GetHelmCreds(), repo.EnableOCI || app.Spec.Source.IsHelmOci())
		...
	} else {
		// git 客户端初始化
		gitClient, err := git.NewClient(repo.Repo, repo.GetGitCreds(), repo.IsInsecure(), repo.IsLFSEnabled())
    // 获取版本
    revision, err = gitClient.LsRemote(ambiguousRevision)
		...
	}
}
```



## app

app包下有两个子目录：discovery和path

### discovery

discovery包下提供了判断一个路径下的编排文件是什么类型的应用（helm、kustomize、ksonnet）

Discover方法被调用的地方：argocd/reposerver/repository/repository.go:ListApps方法，查看app列表时用到

```go
// 入参是一个给定的路径
func Discover(root string) (map[string]string, error) {
	apps := make(map[string]string)
  // 遍历路径下所有的文件
	err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}
		if info.IsDir() {
			return nil
		}
		dir, err := filepath.Rel(root, filepath.Dir(path))
		if err != nil {
			return err
		}
		base := filepath.Base(path)
    // 如果文件名包含 params.libsonnet 并且所在文件夹以 components 作为后缀，则是 ksonnet 类型的应用
		if base == "params.libsonnet" && strings.HasSuffix(dir, "components") {
			apps[filepath.Dir(dir)] = "Ksonnet"
		}
    // 如果文件中包含 Chart.yaml 结尾的文件，则是 helm 类型的应用
		if strings.HasSuffix(base, "Chart.yaml") {
			apps[dir] = "Helm"
		}
    // kustomize的判断看后面的分析，只要包含三个文件中任意一个就符号条件
		if kustomize.IsKustomization(base) {
			apps[dir] = "Kustomize"
		}
		return nil
	})
	return apps, err
}

// kustomize类型应用的判断，只要包含以下三个文件中任意一个，就是 kustomize 类型应用
var KustomizationNames = []string{"kustomization.yaml", "kustomization.yml", "Kustomization"}

func IsKustomization(path string) bool {
	for _, kustomization := range KustomizationNames {
		if path == kustomization {
			return true
		}
	}
	return false
}
```

### path

path包中只提供了一个Path方法，主要是用于拼接给定的git地址和相对路径，组成完整的路径

```go
func Path(root, path string) (string, error) {
	if filepath.IsAbs(path) {
		return "", fmt.Errorf("%s: app path is absolute", path)
	}
  // 拼接给定的仓库的根路径和相对路径
	appPath := filepath.Join(root, path)
	if !strings.HasPrefix(appPath, filepath.Clean(root)) {
		return "", fmt.Errorf("%s: app path outside root", path)
	}
	info, err := os.Stat(appPath)
	if os.IsNotExist(err) {
		return "", fmt.Errorf("%s: app path does not exist", path)
	}
	if err != nil {
		return "", err
	}
  // 应用的路径必须的文件夹
	if !info.IsDir() {
		return "", fmt.Errorf("%s: app path is not a directory", path)
	}
	return appPath, nil
}
```



