# argocd源码分析-reposerver

## 概述

## 源码

### apiclient

#### RepoServerServiceClient

```go
type RepoServerServiceClient interface {
	// GenerateManifest generates manifest for application in specified repo name and revision
	GenerateManifest(ctx context.Context, in *ManifestRequest, opts ...grpc.CallOption) (*ManifestResponse, error)
	// Returns a list of refs (eg. branches and tags) in the repo
	ListRefs(ctx context.Context, in *ListRefsRequest, opts ...grpc.CallOption) (*Refs, error)
	// ListApps returns a list of apps in the repo
	ListApps(ctx context.Context, in *ListAppsRequest, opts ...grpc.CallOption) (*AppList, error)
	// Generate manifest for application in specified repo name and revision
	GetAppDetails(ctx context.Context, in *RepoServerAppDetailsQuery, opts ...grpc.CallOption) (*RepoAppDetailsResponse, error)
	// Get the meta-data (author, date, tags, message) for a specific revision of the repo
	GetRevisionMetadata(ctx context.Context, in *RepoServerRevisionMetadataRequest, opts ...grpc.CallOption) (*v1alpha1.RevisionMetadata, error)
	// GetHelmCharts returns list of helm charts in the specified repository
	GetHelmCharts(ctx context.Context, in *HelmChartsRequest, opts ...grpc.CallOption) (*HelmChartsResponse, error)
}
```



#### RepoServerServiceServer

```go
type RepoServerServiceServer interface {
  //  生成指定仓库名称和版本，判断出应用类型，并生成应用相关的编排文件信息
  // 通过回调函数 runManifestGen，执行生成逻辑
	GenerateManifest(context.Context, *ManifestRequest) (*ManifestResponse, error)
  // 返回给定 git 仓库的所有refs列表（包括分支和tags两类）
	ListRefs(context.Context, *ListRefsRequest) (*Refs, error)
  // 返回给定 git 仓库地址的所有应用列表
	ListApps(context.Context, *ListAppsRequest) (*AppList, error)
  // 生成指定仓库名称和版本，判断出应用类型，并生成应用相关参数（helm、kustomize、jsonnet）
	GetAppDetails(context.Context, *RepoServerAppDetailsQuery) (*RepoAppDetailsResponse, error)
  // 获取 git 仓库的元信息，包括：作者、创建日期、tags、提交信息等
	GetRevisionMetadata(context.Context, *RepoServerRevisionMetadataRequest) (*v1alpha1.RevisionMetadata, error)
  // 返回指定 helm 仓库中所有的 chart 包列表
	GetHelmCharts(context.Context, *HelmChartsRequest) (*HelmChartsResponse, error)
}
```

#### NewRepoServerClient



创建 grpc 客户端

### GenerateManifest

这里传入的 s.runManifestGen 回调函数，是生成编排文件的核心逻辑

```go
// 根据请求生成待部署应用的编排文件
func (s *Service) GenerateManifest(ctx context.Context, q *apiclient.ManifestRequest) (*apiclient.ManifestResponse, error) {
	resultUncast, err := s.runRepoOperation(ctx, q.Revision, q.Repo, q.ApplicationSource, q.VerifySignature,
		func(cacheKey string, firstInvocation bool) (bool, interface{}, error) {
			return s.getManifestCacheEntry(cacheKey, q, firstInvocation)
		}, func(repoRoot, commitSHA, cacheKey string, ctxSrc operationContextSrc) (interface{}, error) {
			return s.runManifestGen(repoRoot, commitSHA, cacheKey, ctxSrc, q)
		}, operationSettings{sem: s.parallelismLimitSemaphore, noCache: q.NoCache, allowConcurrent: q.ApplicationSource.AllowsConcurrentProcessing()})
	result, ok := resultUncast.(*apiclient.ManifestResponse)
	if result != nil && !ok {
		return nil, errors.New("unexpected result type")
	}

	return result, err
}
```

### runRepoOperation

#### 功能说明

从git仓库或者chart包仓库下载包，并执行特定的操作。调用getCached函数返回缓存，如果缓存获取不到数据，则调用operation获取值

#### 核心逻辑

- 获取revision
- 根据source类型决定是创建GitClient还是HelmClient。（判断依据：application对象的chart字段是否为空）
- 调用getCached从缓存获取数据
- 如果是helm类型的应用，通过helmClient执行以下操作：
  - 先调用CleanChartCache清除缓存
  - 下载chart包到本地，并返回本地路径
  - 调用operator执行后续操作
- 如果是git类型的应用，通过gitClient执行以下操作：
  - 获取git的commitid
  - 调用getCached获取缓存
  - 调用operation执行后续操作

```go
func (s *Service) runRepoOperation(
	ctx context.Context,
	revision string,
	repo *v1alpha1.Repository,
	source *v1alpha1.ApplicationSource,
	verifyCommit bool,
	getCached func(cacheKey string, firstInvocation bool) (bool, interface{}, error),
	operation func(repoRoot, commitSHA, cacheKey string, ctxSrc operationContextSrc) (interface{}, error),
  settings operationSettings) (interface{}, error) {
  ...
}
```

#### operator：runManifestGen

operator是谁呢？正是前面注册的回调函数 s.runManifestGen

```go
func (s *Service) runManifestGen(repoRoot, commitSHA, cacheKey string, ctxSrc operationContextSrc, q *apiclient.ManifestRequest) (interface{}, error) {
	...
	if err == nil {
    // 调用 GenerateManifests
		manifestGenResult, err = GenerateManifests(ctx.appPath, repoRoot, commitSHA, q, false)
	}
	...
}
```

#### GenerateManifests

```go
func GenerateManifests(appPath, repoRoot, revision string, q *apiclient.ManifestRequest, isLocal bool) (*apiclient.ManifestResponse, error) {
	// 获取应用类型
	appSourceType, err := GetAppSourceType(q.ApplicationSource, appPath, q.AppName)
	...
  // 根据不同的类型，转调不同的实现逻辑
	switch appSourceType {
  // ksonnet 类型应用处理
	case v1alpha1.ApplicationSourceTypeKsonnet:
		targetObjs, dest, err = ksShow(q.AppLabelKey, appPath, q.ApplicationSource.Ksonnet) 
  // helm 应用处理，将模板渲染成真正执行的yaml文件
  // 内部封装调用 helm template 这个命令行
	case v1alpha1.ApplicationSourceTypeHelm:
		targetObjs, err = helmTemplate(appPath, repoRoot, env, q, isLocal)
  // kustomize 应用处理
	case v1alpha1.ApplicationSourceTypeKustomize:
		kustomizeBinary := ""
		if q.KustomizeOptions != nil {
			kustomizeBinary = q.KustomizeOptions.BinaryPath
		}
		k := kustomize.NewKustomizeApp(appPath, q.Repo.GetGitCreds(), repoURL, kustomizeBinary)
		targetObjs, _, err = k.Build(q.ApplicationSource.Kustomize, q.KustomizeOptions)
	case v1alpha1.ApplicationSourceTypePlugin:
		targetObjs, err = runConfigManagementPlugin(appPath, env, q, q.Repo.GetGitCreds())
	case v1alpha1.ApplicationSourceTypeDirectory:
		var directory *v1alpha1.ApplicationSourceDirectory
		if directory = q.ApplicationSource.Directory; directory == nil {
			directory = &v1alpha1.ApplicationSourceDirectory{}
		}
		targetObjs, err = findManifests(appPath, repoRoot, env, *directory)
	}
	if err != nil {
		return nil, err
	}

  // 遍历所有生成的编排文件对象，将yaml文件转换成string
	...
	res := apiclient.ManifestResponse{
		Manifests:  manifests,
		SourceType: string(appSourceType),
	}
	if dest != nil {
		res.Namespace = dest.Namespace
		res.Server = dest.Server
	}
	return &res, nil
}
```

