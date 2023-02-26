# Client Go源码分析 ListWatch实现


## ListerWatcher接口

ListerWatcher是Lister和Watcher接口的结合体，Lister负责与APIServer通信列出全量对象，后者Watcher负责监听这些对象的增量变化。

> List-Watch机制存在的原因：
>
> 一句话概括：为了提高访问效率。因为k8s资源信息都是保存在etcd中，每一次访问资源都需要客户端通过APIServer进行访问，如果很多的客户端频繁的列举出全量对象（比如列举所有的pod），这会对APIServer进程（或者成为服务）不堪重负。
>
> 因此List-Watch就是为了在本地进行缓存(Indexer)，只需要访问一次APIServer列举出全量对象，并且同步本地缓存内容（Indexer）。后续通过Watch机制监听所有的这类对象的变化，当监听到变化的时候，也只需要同步本地缓存(Indexer)即可。这样就会大大的提高效率，与APIServer的通信也只是资源的增量变化。

### ListerWatcher接口定义

```go
// Lister is any object that knows how to perform an initial list.
type Lister interface {
  // List should return a list type object; the Items field will be extracted, and the
  // ResourceVersion field will be used to start the watch in the right place.
  List(options metav1.ListOptions) (runtime.Object, error)
}

// Watcher is any object that knows how to start a watch on a resource.
type Watcher interface {
  // Watch should begin a watch at the specified version.
  Watch(options metav1.ListOptions) (watch.Interface, error)
}

// ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
type ListerWatcher interface {
  Lister
  Watcher
}
```

从注释中可以看出对上述接口的描述：

* Lister接口的函数List主要实现是返回所有的资源列表；

* Watcher接口的函数Watch开始监听上述资源（需要指定特定的版本resourceVersion是一个全局的ID）；

### ListWatch结构实现接口

```go
// ListFunc knows how to list resources
type ListFunc func(options metav1.ListOptions) (runtime.Object, error)

// WatchFunc knows how to watch resources
type WatchFunc func(options metav1.ListOptions) (watch.Interface, error)

// ListWatch knows how to list and watch a set of apiserver resources.  It satisfies the ListerWatcher interface.
// It is a convenience function for users of NewReflector, etc.
// ListFunc and WatchFunc must not be nil
type ListWatch struct {
  ListFunc  ListFunc
  WatchFunc WatchFunc
  // DisableChunking requests no chunking for this list watcher.
  DisableChunking bool
}
```

具体的实现函数如下：

```go
// List实现了ListerWatcher.List()
// List a set of apiserver resources
func (lw *ListWatch) List(options metav1.ListOptions) (runtime.Object, error) {
  // ListWatch is used in Reflector, which already supports pagination.
  // Don't paginate here to avoid duplication.
  return lw.ListFunc(options)
}

// List实现了ListerWatcher.Watch()
// Watch a set of apiserver resources
func (lw *ListWatch) Watch(options metav1.ListOptions) (watch.Interface, error) {
  return lw.WatchFunc(options)
}
```

从上面可以看到，因为ListerWatcher接口只包含List和Watch两个函数，所以这里的ListWatch struct也只是两个成员函数，所以就认为是ListWatch struct实现了ListerWatcher接口。

> 非常值得注意的是，这里的List函数和Watch成员函数分别调用了ListWatch注册的两个函数，ListFunc和WatchFunc。
>
> 后续所有资源类型的Informer都会注册自己的ListWatch结构，比如在创建下面的Deployment的Informer就会进行注册自己的ListWatch结构（也就是List&Watch函数）

### 使用 ListWatch 的 Informer

后文会介绍，各资源类型都有自己特定的 Informer（codegen 工具自动生成），如 [Deployment Informer](https://github.com/opsnull/kubernetes-dev-docs/blob/master/client-go/k8s.io/client-go/informers/extensions/v1beta1/deployment.go)，它们使用自己资源类型的 ClientSet 来初始化 ListWatch，只返回对应类型的对象：

```go
// 来源于 k8s.io/client-go/informers/extensions/v1beta1/deployment.go
func NewFilteredDeploymentInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
  return cache.NewSharedIndexInformer(
    // 使用特定资源类型的 RESTClient 创建 ListWatch
    &cache.ListWatch{
      ListFunc: func(options v1.ListOptions) (runtime.Object, error) {
        if tweakListOptions != nil {
          tweakListOptions(&options)
        }
        return client.ExtensionsV1beta1().Deployments(namespace).List(options)
      },
      WatchFunc: func(options v1.ListOptions) (watch.Interface, error) {
        if tweakListOptions != nil {
          tweakListOptions(&options)
        }
        return client.ExtensionsV1beta1().Deployments(namespace).Watch(options)
      },
    },
    &extensionsv1beta1.Deployment{},
    resyncPeriod,
    indexers,
  )
}
```

