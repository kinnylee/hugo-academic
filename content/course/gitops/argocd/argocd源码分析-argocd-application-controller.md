# argocd源码分析-argocd-application-controller

## 概述

- cluster存放在secret中
- 缓存支持两种：redis、memory

## CRD

pkg/apis/application/v1alpha1/types.go

```go
type AppProject struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata" protobuf:"bytes,1,opt,name=metadata"`
	Spec              AppProjectSpec   `json:"spec" protobuf:"bytes,2,opt,name=spec"`
	Status            AppProjectStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type AppProjectSpec struct {
  // 被用来部署的仓库地址列表信息
	SourceRepos []string `json:"sourceRepos,omitempty" protobuf:"bytes,1,name=sourceRepos"`
  // 目标部署列表
	Destinations []ApplicationDestination `json:"destinations,omitempty" protobuf:"bytes,2,name=destination"`
  // 描述
	Description string `json:"description,omitempty" protobuf:"bytes,3,opt,name=description"`
	// Roles are user defined RBAC roles associated with this project
	Roles []ProjectRole `json:"roles,omitempty" protobuf:"bytes,4,rep,name=roles"`
	// ClusterResourceWhitelist contains list of whitelisted cluster level resources
	ClusterResourceWhitelist []metav1.GroupKind `json:"clusterResourceWhitelist,omitempty" protobuf:"bytes,5,opt,name=clusterResourceWhitelist"`
	// NamespaceResourceBlacklist contains list of blacklisted namespace level resources
	NamespaceResourceBlacklist []metav1.GroupKind `json:"namespaceResourceBlacklist,omitempty" protobuf:"bytes,6,opt,name=namespaceResourceBlacklist"`
	// OrphanedResources specifies if controller should monitor orphaned resources of apps in this project
	OrphanedResources *OrphanedResourcesMonitorSettings `json:"orphanedResources,omitempty" protobuf:"bytes,7,opt,name=orphanedResources"`
	// SyncWindows controls when syncs can be run for apps in this project
	SyncWindows SyncWindows `json:"syncWindows,omitempty" protobuf:"bytes,8,opt,name=syncWindows"`
	// NamespaceResourceWhitelist contains list of whitelisted namespace level resources
	NamespaceResourceWhitelist []metav1.GroupKind `json:"namespaceResourceWhitelist,omitempty" protobuf:"bytes,9,opt,name=namespaceResourceWhitelist"`
	// List of PGP key IDs that commits to be synced to must be signed with
	SignatureKeys []SignatureKey `json:"signatureKeys,omitempty" protobuf:"bytes,10,opt,name=signatureKeys"`
	// ClusterResourceBlacklist contains list of blacklisted cluster level resources
	ClusterResourceBlacklist []metav1.GroupKind `json:"clusterResourceBlacklist,omitempty" protobuf:"bytes,11,opt,name=clusterResourceBlacklist"`
}

// 应用部署的目标集群信息
type ApplicationDestination struct {
	// Server overrides the environment server value in the ksonnet app.yaml
	Server string `json:"server,omitempty" protobuf:"bytes,1,opt,name=server"`
	// Namespace overrides the environment namespace value in the ksonnet app.yaml
  // 目标集群的命名空间
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,2,opt,name=namespace"`
	// Name of the destination cluster which can be used instead of server (url) field
	Name string `json:"name,omitempty" protobuf:"bytes,3,opt,name=name"`

	// nolint:govet
	isServerInferred bool `json:"-"`
}
```

## Controller

### 函数入口

cmd/argocd-application-controller/commands/argocd-application-controller.go

```go
appController, err := controller.NewApplicationController(
				namespace,
				settingsMgr,
				kubeClient,
				appClient,
				repoClientset,
				cache,
				kubectl,
				resyncDuration,
				time.Duration(selfHealTimeoutSeconds)*time.Second,
				metricsPort,
				kubectlParallelismLimit,
				clusterFilter)
...
go appController.Run(ctx, statusProcessors, operationProcessors)
```

### NewApplicationController

负责注册增、删、改、查的回调函数

```go
projInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			if key, err := cache.MetaNamespaceKeyFunc(obj); err == nil {
				ctrl.projectRefreshQueue.Add(key)
			}
		},
		UpdateFunc: func(old, new interface{}) {
			if key, err := cache.MetaNamespaceKeyFunc(new); err == nil {
				ctrl.projectRefreshQueue.Add(key)
			}
		},
		DeleteFunc: func(obj interface{}) {
			if key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj); err == nil {
				ctrl.projectRefreshQueue.Add(key)
			}
		},
	})
```

### newApplicationInformerAndLister

注册的增删改查回调函数，分别如下：

- AddFunc：
- UpdateFunc：automatedSyncEnabled

```go
func (ctrl *ApplicationController) newApplicationInformerAndLister() (cache.SharedIndexInformer, applisters.ApplicationLister) {
	informer := cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (apiruntime.Object, error) {
				return ctrl.applicationClientset.ArgoprojV1alpha1().Applications(ctrl.namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				return ctrl.applicationClientset.ArgoprojV1alpha1().Applications(ctrl.namespace).Watch(context.TODO(), options)
			},
		},
		&appv1.Application{},
		ctrl.statusRefreshTimeout,
		cache.Indexers{
			cache.NamespaceIndex: func(obj interface{}) ([]string, error) {
				app, ok := obj.(*appv1.Application)
				if ok {
					if err := argo.ValidateDestination(context.Background(), &app.Spec.Destination, ctrl.db); err != nil {
						ctrl.setAppCondition(app, appv1.ApplicationCondition{Type: appv1.ApplicationConditionInvalidSpecError, Message: err.Error()})
					}
				}

				return cache.MetaNamespaceIndexFunc(obj)
			},
			orphanedIndex: func(obj interface{}) (i []string, e error) {
				app, ok := obj.(*appv1.Application)
				if !ok {
					return nil, nil
				}

				proj, err := ctrl.getAppProj(app)
				if err != nil {
					return nil, nil
				}
				if proj.Spec.OrphanedResources != nil {
					return []string{app.Spec.Destination.Namespace}, nil
				}
				return nil, nil
			},
		},
	)
	lister := applisters.NewApplicationLister(informer.GetIndexer())
	informer.AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				if !ctrl.canProcessApp(obj) {
					return
				}
				key, err := cache.MetaNamespaceKeyFunc(obj)
				if err == nil {
					ctrl.appRefreshQueue.Add(key)
					ctrl.appOperationQueue.Add(key)
				}
			},
			UpdateFunc: func(old, new interface{}) {
				if !ctrl.canProcessApp(new) {
					return
				}

				key, err := cache.MetaNamespaceKeyFunc(new)
				if err != nil {
					return
				}
				var compareWith *CompareWith
				oldApp, oldOK := old.(*appv1.Application)
				newApp, newOK := new.(*appv1.Application)
				if oldOK && newOK && automatedSyncEnabled(oldApp, newApp) {
					log.WithField("application", newApp.Name).Info("Enabled automated sync")
					compareWith = CompareWithLatest.Pointer()
				}
				ctrl.requestAppRefresh(newApp.Name, compareWith, nil)
				ctrl.appOperationQueue.Add(key)
			},
			DeleteFunc: func(obj interface{}) {
				if !ctrl.canProcessApp(obj) {
					return
				}
				// IndexerInformer uses a delta queue, therefore for deletes we have to use this
				// key function.
				key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
				if err == nil {
					ctrl.appRefreshQueue.Add(key)
				}
			},
		},
	)
	return informer, lister
}
```



### Run函数

```go
func (ctrl *ApplicationController) Run(ctx context.Context, statusProcessors int, operationProcessors int) {
	...
  // 启动informer
	go ctrl.appInformer.Run(ctx.Done())
	go ctrl.projInformer.Run(ctx.Done())

	errors.CheckError(ctrl.stateCache.Init())

	if !cache.WaitForCacheSync(ctx.Done(), ctrl.appInformer.HasSynced, ctrl.projInformer.HasSynced) {
		log.Error("Timed out waiting for caches to sync")
		return
	}

	go func() { errors.CheckError(ctrl.stateCache.Run(ctx)) }()
	go func() { errors.CheckError(ctrl.metricsServer.ListenAndServe()) }()

	for i := 0; i < statusProcessors; i++ {
		go wait.Until(func() {
      // app处理逻辑
			for ctrl.processAppRefreshQueueItem() {
			}
		}, time.Second, ctx.Done())
	}

	for i := 0; i < operationProcessors; i++ {
		go wait.Until(func() {
			for ctrl.processAppOperationQueueItem() {
			}
		}, time.Second, ctx.Done())
	}

	go wait.Until(func() {
		for ctrl.processAppComparisonTypeQueueItem() {
		}
	}, time.Second, ctx.Done())

	go wait.Until(func() {
		for ctrl.processProjectQueueItem() {
		}
	}, time.Second, ctx.Done())
	<-ctx.Done()
}
```

#### processAppRefreshQueueItem

```go
func (ctrl *ApplicationController) processAppRefreshQueueItem() (processNext bool) {
	...
  // 获取到Application
	app := origApp.DeepCopy()
	...
	if comparisonLevel == ComparisonWithNothing {
		managedResources := make([]*appv1.ResourceDiff, 0)
    // 获取缓存
		if err := ctrl.cache.GetAppManagedResources(app.Name, &managedResources); err != nil {
			logCtx.Warnf("Failed to get cached managed resources for tree reconciliation, fallback to full reconciliation")
		} else {
			var tree *appv1.ApplicationTree
      // 获取资源信息
			if tree, err = ctrl.getResourceTree(app, managedResources); err == nil {
				app.Status.Summary = tree.GetSummary()
				if err := ctrl.cache.SetAppResourcesTree(app.Name, tree); err != nil {
					logCtx.Errorf("Failed to cache resources tree: %v", err)
					return
				}
			}

			ctrl.persistAppStatus(origApp, &app.Status)
			return
		}
	}

	project, hasErrors := ctrl.refreshAppConditions(app)
	if hasErrors {
		app.Status.Sync.Status = appv1.SyncStatusCodeUnknown
		app.Status.Health.Status = health.HealthStatusUnknown
		ctrl.persistAppStatus(origApp, &app.Status)
		return
	}

	var localManifests []string
	if opState := app.Status.OperationState; opState != nil && opState.Operation.Sync != nil {
		localManifests = opState.Operation.Sync.Manifests
	}

	revision := app.Spec.Source.TargetRevision
	if comparisonLevel == CompareWithRecent {
		revision = app.Status.Sync.Revision
	}

	now := metav1.Now()
	compareResult := ctrl.appStateManager.CompareAppState(app, project, revision, app.Spec.Source, refreshType == appv1.RefreshTypeHard, localManifests)
	for k, v := range compareResult.timings {
		logCtx = logCtx.WithField(k, v.Milliseconds())
	}

	ctrl.normalizeApplication(origApp, app)

	tree, err := ctrl.setAppManagedResources(app, compareResult)
	if err != nil {
		logCtx.Errorf("Failed to cache app resources: %v", err)
	} else {
		app.Status.Summary = tree.GetSummary()
	}

	if project.Spec.SyncWindows.Matches(app).CanSync(false) {
		syncErrCond := ctrl.autoSync(app, compareResult.syncStatus, compareResult.resources)
		if syncErrCond != nil {
			app.Status.SetConditions(
				[]appv1.ApplicationCondition{*syncErrCond},
				map[appv1.ApplicationConditionType]bool{appv1.ApplicationConditionSyncError: true},
			)
		} else {
			app.Status.SetConditions(
				[]appv1.ApplicationCondition{},
				map[appv1.ApplicationConditionType]bool{appv1.ApplicationConditionSyncError: true},
			)
		}
	} else {
		logCtx.Info("Sync prevented by sync window")
	}

	if app.Status.ReconciledAt == nil || comparisonLevel == CompareWithLatest {
		app.Status.ReconciledAt = &now
	}
	app.Status.Sync = *compareResult.syncStatus
	app.Status.Health = *compareResult.healthStatus
	app.Status.Resources = compareResult.resources
	sort.Slice(app.Status.Resources, func(i, j int) bool {
		return resourceStatusKey(app.Status.Resources[i]) < resourceStatusKey(app.Status.Resources[j])
	})
	app.Status.SourceType = compareResult.appSourceType
	ctrl.persistAppStatus(origApp, &app.Status)
	return
}
```

#### processAppOperationQueueItem

#### processAppComparisonTypeQueueItem

#### processProjectQueueItem

