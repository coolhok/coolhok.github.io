---
layout: post
title: k8sApiService store实现
subtitle: k8s存储实现
categories: k8s
tags: [k8s, kubeApiService, store]
---

## store实现
[代码传送门](https://github.com/kubernetes/apiserver/blob/master/pkg/registry/generic/registry/store.go#L95)
[单测case](https://github.com/kubernetes/apiserver/blob/master/pkg/registry/generic/registry/store_test.go#L174)
```golang
// Store implements pkg/api/rest.StandardStorage. It's intended to be
// embeddable and allows the consumer to implement any non-generic functions
// that are required. This object is intended to be copyable so that it can be
// used in different ways but share the same underlying behavior.
//
// All fields are required unless specified.
//
// The intended use of this type is embedding within a Kind specific
// RESTStorage implementation. This type provides CRUD semantics on a Kubelike
// resource, handling details like conflict detection with ResourceVersion and
// semantics. The RESTCreateStrategy, RESTUpdateStrategy, and
// RESTDeleteStrategy are generic across all backends, and encapsulate logic
// specific to the API.
//
// TODO: make the default exposed methods exactly match a generic RESTStorage
type Store struct {
	// NewFunc returns a new instance of the type this registry returns for a
	// GET of a single object, e.g.:
	//
	// curl GET /apis/group/version/namespaces/my-ns/myresource/name-of-object
	NewFunc func() runtime.Object

	// NewListFunc returns a new list of the type this registry; it is the
	// type returned when the resource is listed, e.g.:
	//
	// curl GET /apis/group/version/namespaces/my-ns/myresource
	NewListFunc func() runtime.Object

	// DefaultQualifiedResource is the pluralized name of the resource.
	// This field is used if there is no request info present in the context.
	// See qualifiedResourceFromContext for details.
	DefaultQualifiedResource schema.GroupResource

	// KeyRootFunc returns the root etcd key for this resource; should not
	// include trailing "/".  This is used for operations that work on the
	// entire collection (listing and watching).
	//
	// KeyRootFunc and KeyFunc must be supplied together or not at all.
	KeyRootFunc func(ctx context.Context) string

	// KeyFunc returns the key for a specific object in the collection.
	// KeyFunc is called for Create/Update/Get/Delete. Note that 'namespace'
	// can be gotten from ctx.
	//
	// KeyFunc and KeyRootFunc must be supplied together or not at all.
	KeyFunc func(ctx context.Context, name string) (string, error)

	// ObjectNameFunc returns the name of an object or an error.
	ObjectNameFunc func(obj runtime.Object) (string, error)

	// TTLFunc returns the TTL (time to live) that objects should be persisted
	// with. The existing parameter is the current TTL or the default for this
	// operation. The update parameter indicates whether this is an operation
	// against an existing object.
	//
	// Objects that are persisted with a TTL are evicted once the TTL expires.
	TTLFunc func(obj runtime.Object, existing uint64, update bool) (uint64, error)

	// PredicateFunc returns a matcher corresponding to the provided labels
	// and fields. The SelectionPredicate returned should return true if the
	// object matches the given field and label selectors.
	PredicateFunc func(label labels.Selector, field fields.Selector) storage.SelectionPredicate

	// EnableGarbageCollection affects the handling of Update and Delete
	// requests. Enabling garbage collection allows finalizers to do work to
	// finalize this object before the store deletes it.
	//
	// If any store has garbage collection enabled, it must also be enabled in
	// the kube-controller-manager.
	EnableGarbageCollection bool

	// DeleteCollectionWorkers is the maximum number of workers in a single
	// DeleteCollection call. Delete requests for the items in a collection
	// are issued in parallel.
	DeleteCollectionWorkers int

	// Decorator is an optional exit hook on an object returned from the
	// underlying storage. The returned object could be an individual object
	// (e.g. Pod) or a list type (e.g. PodList). Decorator is intended for
	// integrations that are above storage and should only be used for
	// specific cases where storage of the value is not appropriate, since
	// they cannot be watched.
	Decorator ObjectFunc
	// CreateStrategy implements resource-specific behavior during creation.
	CreateStrategy rest.RESTCreateStrategy
	// AfterCreate implements a further operation to run after a resource is
	// created and before it is decorated, optional.
	AfterCreate ObjectFunc

	// UpdateStrategy implements resource-specific behavior during updates.
	UpdateStrategy rest.RESTUpdateStrategy
	// AfterUpdate implements a further operation to run after a resource is
	// updated and before it is decorated, optional.
	AfterUpdate ObjectFunc

	// DeleteStrategy implements resource-specific behavior during deletion.
	DeleteStrategy rest.RESTDeleteStrategy
	// AfterDelete implements a further operation to run after a resource is
	// deleted and before it is decorated, optional.
	AfterDelete ObjectFunc
	// ReturnDeletedObject determines whether the Store returns the object
	// that was deleted. Otherwise, return a generic success status response.
	ReturnDeletedObject bool
	// ShouldDeleteDuringUpdate is an optional function to determine whether
	// an update from existing to obj should result in a delete.
	// If specified, this is checked in addition to standard finalizer,
	// deletionTimestamp, and deletionGracePeriodSeconds checks.
	ShouldDeleteDuringUpdate func(ctx context.Context, key string, obj, existing runtime.Object) bool
	// ExportStrategy implements resource-specific behavior during export,
	// optional. Exported objects are not decorated.
	ExportStrategy rest.RESTExportStrategy
	// TableConvertor is an optional interface for transforming items or lists
	// of items into tabular output. If unset, the default will be used.
	TableConvertor rest.TableConvertor

	// Storage is the interface for the underlying storage for the
	// resource. It is wrapped into a "DryRunnableStorage" that will
	// either pass-through or simply dry-run.
	Storage DryRunnableStorage
	// StorageVersioner outputs the <group/version/kind> an object will be
	// converted to before persisted in etcd, given a list of possible
	// kinds of the object.
	// If the StorageVersioner is nil, apiserver will leave the
	// storageVersionHash as empty in the discovery document.
	StorageVersioner runtime.GroupVersioner
	// Called to cleanup clients used by the underlying Storage; optional.
	DestroyFunc func()
}
```
### create
[create](https://github.com/kubernetes/apiserver/blob/master/pkg/registry/generic/registry/store.go#L372)

#### BeforeCreate
```golang
type RESTCreateStrategy interface {
	runtime.ObjectTyper
	// The name generator is used when the standard GenerateName field is set.
	// The NameGenerator will be invoked prior to validation.
	names.NameGenerator

	// NamespaceScoped returns true if the object must be within a namespace.
	NamespaceScoped() bool
	// PrepareForCreate is invoked on create before validation to normalize
	// the object.  For example: remove fields that are not to be persisted,
	// sort order-insensitive list fields, etc.  This should not remove fields
	// whose presence would be considered a validation error.
	//
	// Often implemented as a type check and an initailization or clearing of
	// status. Clear the status because status changes are internal. External
	// callers of an api (users) should not be setting an initial status on
	// newly created objects.
	PrepareForCreate(ctx context.Context, obj runtime.Object)
	// Validate returns an ErrorList with validation errors or nil.  Validate
	// is invoked after default fields in the object have been filled in
	// before the object is persisted.  This method should not mutate the
	// object.
	Validate(ctx context.Context, obj runtime.Object) field.ErrorList
	// Canonicalize allows an object to be mutated into a canonical form. This
	// ensures that code that operates on these objects can rely on the common
	// form for things like comparison.  Canonicalize is invoked after
	// validation has succeeded but before the object has been persisted.
	// This method may mutate the object. Often implemented as a type check or
	// empty method.
	Canonicalize(obj runtime.Object)
}
```
#### 创建过程
最终通过 Storage实现创建,Storage是etcd封装其已经实现了事务
```golang
	key, err := e.KeyFunc(ctx, name)
	if err != nil {
		return nil, err
	}
	qualifiedResource := e.qualifiedResourceFromContext(ctx)
	ttl, err := e.calculateTTL(obj, 0, false)
	if err != nil {
		return nil, err
	}
	out := e.NewFunc()
	if err := e.Storage.Create(ctx, key, obj, out, ttl, dryrun.IsDryRun(options.DryRun)); err != nil {
		err = storeerr.InterpretCreateError(err, qualifiedResource, name)
		err = rest.CheckGeneratedNameError(e.CreateStrategy, err, obj)
		if !kubeerr.IsAlreadyExists(err) {
			return nil, err
		}
		if errGet := e.Storage.Get(ctx, key, "", out, false); errGet != nil {
			return nil, err
		}
		accessor, errGetAcc := meta.Accessor(out)
		if errGetAcc != nil {
			return nil, err
		}
		if accessor.GetDeletionTimestamp() != nil {
			msg := &err.(*kubeerr.StatusError).ErrStatus.Message
			*msg = fmt.Sprintf("object is being deleted: %s", *msg)
		}
		return nil, err
	}
	if e.AfterCreate != nil {
		if err := e.AfterCreate(out); err != nil {
			return nil, err
		}
	}
	if e.Decorator != nil {
		if err := e.Decorator(out); err != nil {
			return nil, err
		}
	}
```
### get
[get](https://github.com/kubernetes/apiserver/blob/master/pkg/registry/generic/registry/store.go#L718)
```golang
func (e *Store) Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
	obj := e.NewFunc()
	key, err := e.KeyFunc(ctx, name)
	if err != nil {
		return nil, err
	}
	if err := e.Storage.Get(ctx, key, storage.GetOptions{ResourceVersion: options.ResourceVersion}, obj); err != nil {
		return nil, storeerr.InterpretGetError(err, e.qualifiedResourceFromContext(ctx), name)
	}
	if e.Decorator != nil {
		e.Decorator(obj)
	}
	return obj, nil
}
```

### delete
#### Finalizers
[Finalizers](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/finalizers/)

Finalizer 是带有命名空间的键，告诉 Kubernetes 等到特定的条件被满足后， 再完全删除被标记为删除的资源。 Finalizer 提醒控制器清理被删除的对象拥有的资源。

当你告诉 Kubernetes 删除一个指定了 Finalizer 的对象时， Kubernetes API 会将该对象标记为删除，使其进入只读状态。 此时控制平面或其他组件会采取 Finalizer 所定义的行动， 而目标对象仍然处于终止中（Terminating）的状态。 这些行动完成后，控制器会删除目标对象相关的 Finalizer。 当 metadata.finalizers 字段为空时，Kubernetes 认为删除已完成。

你可以使用 Finalizer 控制资源的垃圾收集。 例如，你可以定义一个 Finalizer，在删除目标资源前清理相关资源或基础设施。

你可以通过使用 Finalizers 提醒控制器 在删除目标资源前执行特定的清理任务， 来控制资源的垃圾收集。

Finalizers 通常不指定要执行的代码。 相反，它们通常是特定资源上的键的列表，类似于注解。 Kubernetes 自动指定了一些 Finalizers，但你也可以指定你自己的。

Finalizers 如何工作 
当你使用清单文件创建资源时，你可以在 metadata.finalizers 字段指定 Finalizers。 当你试图删除该资源时，管理该资源的控制器会注意到 finalizers 字段中的值， 并进行以下操作：

修改对象，将你开始执行删除的时间添加到 metadata.deletionTimestamp 字段。
将该对象标记为只读，直到其 metadata.finalizers 字段为空。
然后，控制器试图满足资源的 Finalizers 的条件。 每当一个 Finalizer 的条件被满足时，控制器就会从资源的 finalizers 字段中删除该键。 当该字段为空时，垃圾收集继续进行。 你也可以使用 Finalizers 来阻止删除未被管理的资源。

一个常见的 Finalizer 的例子是 kubernetes.io/pv-protection， 它用来防止意外删除 PersistentVolume 对象。 当一个 PersistentVolume 对象被 Pod 使用时， Kubernetes 会添加 pv-protection Finalizer。 如果你试图删除 PersistentVolume，它将进入 Terminating 状态， 但是控制器因为该 Finalizer 存在而无法删除该资源。 当 Pod 停止使用 PersistentVolume 时， Kubernetes 清除 pv-protection Finalizer，控制器就会删除该卷。

属主引用、标签和 Finalizers
与标签类似， 属主引用 描述了 Kubernetes 中对象之间的关系，但它们作用不同。 当一个控制器 管理类似于 Pod 的对象时，它使用标签来跟踪相关对象组的变化。 例如，当 Job 创建一个或多个 Pod 时， Job 控制器会给这些 Pod 应用上标签，并跟踪集群中的具有相同标签的 Pod 的变化。

Job 控制器还为这些 Pod 添加了属主引用，指向创建 Pod 的 Job。 如果你在这些 Pod 运行的时候删除了 Job， Kubernetes 会使用属主引用（而不是标签）来确定集群中哪些 Pod 需要清理。

当 Kubernetes 识别到要删除的资源上的属主引用时，它也会处理 Finalizers。

在某些情况下，Finalizers 会阻止依赖对象的删除， 这可能导致目标属主对象，保持在只读状态的时间比预期的长，且没有被完全删除。 在这些情况下，你应该检查目标属主和附属对象上的 Finalizers 和属主引用，来排查原因。
#### 垃圾回收
[垃圾收集](https://kubernetes.io/zh/docs/concepts/architecture/garbage-collection/#cascading-deletion)
垃圾收集是 Kubernetes 用于清理集群资源的各种机制的统称。 垃圾收集允许系统清理如下资源：

失败的 Pod
已完成的 Job
不再存在属主引用的对象
未使用的容器和容器镜像
动态制备的、StorageClass 回收策略为 Delete 的 PV 卷
阻滞或者过期的 CertificateSigningRequest (CSRs)
在以下情形中删除了的节点对象：
当集群使用云控制器管理器运行于云端时；
当集群使用类似于云控制器管理器的插件运行在本地环境中时。
节点租约对象
属主与依赖 
Kubernetes 中很多对象通过属主引用 链接到彼此。属主引用（Owner Reference）可以告诉控制面哪些对象依赖于其他对象。 Kubernetes 使用属主引用来为控制面以及其他 API 客户端在删除某对象时提供一个 清理关联资源的机会。在大多数场合，Kubernetes 都是自动管理属主引用的。

属主关系与某些资源所使用的的标签和选择算符 不同。例如，考虑一个创建 EndpointSlice 对象的 Service 对象。Service 对象使用标签来允许控制面确定哪些 EndpointSlice 对象被该 Service 使用。除了标签，每个被 Service 托管的 EndpointSlice 对象还有一个属主引用属性。 属主引用可以帮助 Kubernetes 中的不同组件避免干预并非由它们控制的对象。

说明：
根据设计，系统不允许出现跨名字空间的属主引用。名字空间作用域的依赖对象可以指定集群作用域或者名字空间作用域的属主。 名字空间作用域的属主必须存在于依赖对象所在的同一名字空间。 如果属主位于不同名字空间，则属主引用被视为不存在，而当检查发现所有属主都已不存在时， 依赖对象会被删除。

集群作用域的依赖对象只能指定集群作用域的属主。 在 1.20 及更高版本中，如果一个集群作用域的依赖对象指定了某个名字空间作用域的类别作为其属主， 则该对象被视为拥有一个无法解析的属主引用，因而无法被垃圾收集处理。

在 1.20 及更高版本中，如果垃圾收集器检测到非法的跨名字空间 ownerReference， 或者某集群作用域的依赖对象的 ownerReference 引用某名字空间作用域的类别， 系统会生成一个警告事件，其原因为 OwnerRefInvalidNamespace，involvedObject 设置为非法的依赖对象。你可以通过运行 kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace 来检查是否存在这类事件。

级联删除 
Kubernetes 会检查并删除那些不再拥有属主引用的对象，例如在你删除了 ReplicaSet 之后留下来的 Pod。当你删除某个对象时，你可以控制 Kubernetes 是否要通过一个称作 级联删除（Cascading Deletion）的过程自动删除该对象的依赖对象。 级联删除有两种类型，分别如下：

前台级联删除
后台级联删除
你也可以使用 Kubernetes Finalizers 来控制垃圾收集机制如何以及何时删除包含属主引用的资源。

前台级联删除
在前台级联删除中，正在被你删除的对象首先进入 deletion in progress 状态。 在这种状态下，针对属主对象会发生以下事情：

Kubernetes API 服务器将对象的 metadata.deletionTimestamp 字段设置为对象被标记为要删除的时间点。
Kubernetes API 服务器也会将 metadata.finalizers 字段设置为 foregroundDeletion。
在删除过程完成之前，通过 Kubernetes API 仍然可以看到该对象。
当属主对象进入删除过程中状态后，控制器删除其依赖对象。控制器在删除完所有依赖对象之后， 删除属主对象。这时，通过 Kubernetes API 就无法再看到该对象。

在前台级联删除过程中，唯一的可能阻止属主对象被删除的依赖对象是那些带有 ownerReference.blockOwnerDeletion=true 字段的对象。 参阅使用前台级联删除 以了解进一步的细节。

后台级联删除
在后台级联删除过程中，Kubernetes 服务器立即删除属主对象，控制器在后台清理所有依赖对象。 默认情况下，Kubernetes 使用后台级联删除方案，除非你手动设置了要使用前台删除， 或者选择遗弃依赖对象。

参阅使用后台级联删除 以了解进一步的细节。

被遗弃的依赖对象 
当 Kubernetes 删除某个属主对象时，被留下来的依赖对象被称作被遗弃的（Orphaned）对象。 默认情况下，Kubernetes 会删除依赖对象。要了解如何重载这种默认行为，可参阅 删除属主对象和遗弃依赖对象。

未使用容器和镜像的垃圾收集 
kubelet 会每五分钟对未使用的镜像执行一次垃圾收集， 每分钟对未使用的容器执行一次垃圾收集。 你应该避免使用外部的垃圾收集工具，因为外部工具可能会破坏 kubelet 的行为，移除应该保留的容器。

要配置对未使用容器和镜像的垃圾收集选项，可以使用一个 配置文件，基于 KubeletConfiguration 资源类型来调整与垃圾搜集相关的 kubelet 行为。

容器镜像生命期 
Kubernetes 通过其镜像管理器（Image Manager）来管理所有镜像的生命周期， 该管理器是 kubelet 的一部分，工作时与 cadvisor 协同。 kubelet 在作出垃圾收集决定时会考虑如下磁盘用量约束：

HighThresholdPercent
LowThresholdPercent
磁盘用量超出所配置的 HighThresholdPercent 值时会触发垃圾收集， 垃圾收集器会基于镜像上次被使用的时间来按顺序删除它们，首先删除的是最老的镜像。 kubelet 会持续删除镜像，直到磁盘用量到达 LowThresholdPercent 值为止。

容器垃圾收集 
kubelet 会基于如下变量对所有未使用的容器执行垃圾收集操作，这些变量都是你可以定义的：

MinAge：kubelet 可以垃圾回收某个容器时该容器的最小年龄。设置为 0 表示禁止使用此规则。
MaxPerPodContainer：每个 Pod 可以包含的已死亡的容器个数上限。设置为小于 0 的值表示禁止使用此规则。
MaxContainers：集群中可以存在的已死亡的容器个数上限。设置为小于 0 的值意味着禁止应用此规则。
除以上变量之外，kubelet 还会垃圾收集除无标识的以及已删除的容器，通常从最老的容器开始。

当保持每个 Pod 的最大数量的容器（MaxPerPodContainer）会使得全局的已死亡容器个数超出上限 （MaxContainers）时，MaxPerPodContainer 和 MaxContainers 之间可能会出现冲突。 在这种情况下，kubelet 会调整 MaxPerPodContainer 来解决这一冲突。 最坏的情形是将 MaxPerPodContainer 降格为 1，并驱逐最老的容器。 此外，当隶属于某已被删除的 Pod 的容器的年龄超过 MinAge 时，它们也会被删除。

### update
[更新](https://github.com/kubernetes/apiserver/blob/master/pkg/registry/generic/registry/store.go#L507)

更新过程若出现errEmptiedFinalizers,则更新会变删除