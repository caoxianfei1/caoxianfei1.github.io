# Client Go整体流程梳理

Client-go的整体流程如下图所示：

![client-go-informer](https://user-images.githubusercontent.com/54139479/221352238-d6796902-ec0d-413d-b5f4-32d5674a55d2.png)

首先是从上面图中可以看到，有几个比较重要的组件：Reflector、Informer、Indexer。

Informer在初始化的时候，会先调用Kubernetes List API获得某种resource类型的全部对象，并且获取对象的resourceVersion缓存在内存中。然后调用watch API去watch这种resource，根据resourceVersion号去进行watch，并维护这份缓存。然后将watch到的Event加入到DeltaFIFO中，Reflector的主要工作就是在不断的watch操作。

## 万恶之源-ListerWatcher Interface

一提到client-go不得不说的就是ListWatch机制，该机制的主要目的是减少APIServer的压力，也就是缓存的概念，List就是第一次访问客户端第一次访问APIServer的时候是全量访问，也就是List出etcd中该类的所有的资源，比如Pod。而Watch就是监听这些已缓存的资源的是否发生了更改（ResourceVersion），如果发生变化的则调用具体的handler（Add、Update和Delete）进行处理。

### 首先看ListerWatcher接口定义

```go
// tools/cache/listwatch.go

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

## List是怎样做的？

在Reflector的List方法中，会单独启用一个协程去进行List操作：

```go
pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
			return r.listerWatcher.List(opts)
		}))
```

pager是进行分页，其中是调用了Reflector成员的listerWatcher的List方法列举出对应opts的所有对象。下面再来看一下这个函数的到底是如何调用API Server的：

```go
// 这里是需要用户自定义Controller
type ListWatch struct {
	ListFunc  ListFunc
	WatchFunc WatchFunc
	// DisableChunking requests no chunking for this list watcher.
	DisableChunking bool
}

// List a set of apiserver resources
func (lw *ListWatch) List(options metav1.ListOptions) (runtime.Object, error) {
	// ListWatch is used in Reflector, which already supports pagination.
	// Don't paginate here to avoid duplication.
	return lw.ListFunc(options)
}
```

从上面可以看出，这个ListWatch是需要自定义，其实在每一种资源中都实现了对ListWatch的注册，比如Pod的Informer函数：

```go
// informers/core/v1/pod.go
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
```

上面的注册的函数是`client.CoreV1().Pods(namespace).List(context.TODO(), options)`。client就是负责与API Server进行通信的客户端：kubernetes.Interface。

## Watch是怎样做的？

同样是在Reflector的ListWatch方法中单独启用一个协程去调用watch函数：

```go
w, err := r.listerWatcher.Watch(options)
```

同样是在listwatch中使用的调用自己注册的watchFunc函数：

```go
// Watch a set of apiserver resources
func (lw *ListWatch) Watch(options metav1.ListOptions) (watch.Interface, error) {
	return lw.WatchFunc(options)
}
```

和上面的Pod中的ListFunc一样，WatchFunc也是自己注册的。拿Pod为例，然后实际调用的就是在PodInformer注册的WatchFunc：

```go
return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
```

其中Watch方法最终调用的是kubernetes/typed/core/v1下的Watch函数：

```go
func (c *pods) Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	opts.Watch = true
	return c.client.Get().
		Namespace(c.ns).
		Resource("pods").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Watch(ctx)
}
```

其实最终是调用了Request下的Watch函数：使用的是Request结构的RestClient(http.Client)进行通信的。

下面是client-go对于请求（request的封装）

```go
type Request struct {
	c *RESTClient

	warningHandler WarningHandler

	rateLimiter flowcontrol.RateLimiter
	backoff     BackoffManager
	timeout     time.Duration
	// 这里应该是最多重试次数
	maxRetries int

	// generic components accessible via method setters
	verb       string
	pathPrefix string
	subpath    string
	params     url.Values
	headers    http.Header

	// structural elements of the request that are part of the Kubernetes API conventions
	namespace    string
	namespaceSet bool
	resource     string
	resourceName string
	subresource  string

	// output
	err  error
	body io.Reader

	retryFn requestRetryFunc
}
```

## Reflector是如何保存该类资源事件到DeltaFIFO中的？

同样是在Reflector的ListAndWatch函数中，在 Watch 之后对其进行操作，使用的函数是watchHandler，具体的代码实现如下所示：	

```go
err = watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.expectedTypeName, r.setLastSyncResourceVersion, r.clock, resyncerrc, stopCh)
```

这里会根据watch的事件类型对其进行处理，相应的调用的是store.Add/store.Update/store.Delete，但是最终对于实现了Store接口的DeltaFIFO来说，都是Add操作。

```go
switch event.Type {
			case watch.Added:
				err := store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", name, event.Object, err))
				}
			case watch.Modified:
				err := store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", name, event.Object, err))
				}
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				err := store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", name, event.Object, err))
				}
```

因为上面的storage的实现是DeltaFIFO，所以这里调用的store.Add/store.Update/store.Delete都是DeltaFIFO的成员函数，具体实现在tools/cache/delta_fifo.go中，下面是对增删改的具体实现：

```go
// Add inserts an item, and puts it in the queue. The item is only enqueued
// if it doesn't already exist in the set.
func (f *DeltaFIFO) Add(obj interface{}) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true
	return f.queueActionLocked(Added, obj)
}

// Update is just like Add, but makes an Updated Delta.
func (f *DeltaFIFO) Update(obj interface{}) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true
	return f.queueActionLocked(Updated, obj)
}

// Delete is just like Add, but makes a Deleted Delta. If the given
// object does not already exist, it will be ignored. (It may have
// already been deleted by a Replace (re-list), for example.)  In this
// method `f.knownObjects`, if not nil, provides (via GetByKey)
// _additional_ objects that are considered to already exist.
func (f *DeltaFIFO) Delete(obj interface{}) error {
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true
	if f.knownObjects == nil {
		if _, exists := f.items[id]; !exists {
			// Presumably, this was deleted when a relist happened.
			// Don't provide a second report of the same deletion.
			return nil
		}
	} else {
		// We only want to skip the "deletion" action if the object doesn't
		// exist in knownObjects and it doesn't have corresponding item in items.
		// Note that even if there is a "deletion" action in items, we can ignore it,
		// because it will be deduped automatically in "queueActionLocked"
		_, exists, err := f.knownObjects.GetByKey(id)
		_, itemsExist := f.items[id]
		if err == nil && !exists && !itemsExist {
			// Presumably, this was deleted when a relist happened.
			// Don't provide a second report of the same deletion.
			return nil
		}
	}

	// exist in items and/or KnownObjects
	return f.queueActionLocked(Deleted, obj)
}
```

从上面的代码中可以看到，所有的事件在最后都调用了queueActionLocked，也就是将该Delta放入了一个map[string]Deltas（f.items->DeltaFIFO成员）。其中map的key就是该对象的key，同时会将该key放入到一个queue（f.queue->DeltaFIFO的成员）中等待进行处理，这样就会对每一个资源（对象）可以保证被顺序处理。

```go
// tools/cache/delta_fifo.go
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	oldDeltas := f.items[id]
	newDeltas := append(oldDeltas, Delta{actionType, obj})
	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
		f.items[id] = newDeltas
		f.cond.Broadcast()
	} else {
    ...
```

从上面的代码中可以看到，其实所有的操作增删改的操作都是append，即将这些对象以及对象事件（Delta）添加至DeltaFIFO中。同时就是还会将该对象的Key添加到queue中，这里的queue的作用就是维护在Pop DeltaFIFO的时候可以将保证顺序弹出。其中Key的计算是由成员keyFunc决定的。每一个资源所对应的key应该都是不一样的。成员queue是用来保存消费顺序的（Pop）。

DeltaFIFO中保存的内容应该是这样的：

<img width="676" alt="image-20221008140948618" src="https://user-images.githubusercontent.com/54139479/221352291-a952f8d5-6204-4bb0-8448-4e4216ed421e.png">

从上面的图中也可以看到，DeltaFIFO对象的queue成员中保存的是resource对应的key，而items成员其实是一个map结构： `map[string]Deltas`, key对应的就是queue中的对象的key,Deltas就是 `[]Delta` 数组，保存的是该资源对象以及该资源对象的事件event

## DeltaFIFO中保存的内容？

DeltaFIFO中保存的是一个个的Deltas列表，Deltas就是某一种对象的Delta列表。

Delta对象由事件类型和对象组成，Deltas就是这些对象组成的列表，所以DeltaFIFO中保存的就是这些一个个对象的列表，比如Pod和Node是由两个不同的列表维护的。

```go
type Delta struct {
	Type   DeltaType
	Object interface{}
}

// Deltas is a list of one or more 'Delta's to an individual object.
// The oldest delta is at index 0, the newest delta is the last one.
type Deltas []Delta
```

DeltaFIFO结构中比较重要的几个成员如下所示，以及几个变量的作用也已经在上面进行注释：

```go
type DeltaFIFO struct {

	// `items` maps a key to a Deltas.
	// Each such Deltas has at least one Delta.
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	// There are no duplicates in `queue`.
	// A key is in `queue` if and only if it is in `items`.
	queue []string
  
  ...
  
  // keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc
}
```

## Event事件是如何消费的？

保存在DeltaFIFO中的事件的对其进行消费是client-go另外一条重要的主线。

在shared_informer.go的Run函数主要启动了两个部分：Controller和sharedProcessor。具体代码如下：

```go
// tools/cache/shared_informer.go 

func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}
	// 创建带有indexer的DeltaFIFO
	// 所以后面使用的FIFO其实就是DeltaFIFO
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          s.indexer,
		EmitDeltaTypeReplaced: true,
	})

	//  在controller定义的Config,用于创建Controller
	// Config中包含了DeltaFIFO/ListerWatcher两个重要组件,同时还有用于处理Delta的Process
	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		// 这里注册的就是处理Delta的函数（ProcessFunc）
		// 这个函数在Delta从FIFO中被弹出来之前被调用,调用顺序是：
		// 这个也是WatchEvent消费过程：Controller.Run()->Controller.ProcessLoop()->queue.Pop()->sharedIndexInformer.HandleDeltas()
		Process:           s.HandleDeltas,
		WatchErrorHandler: s.watchErrorHandler,
	}

	// 根据Config对象创建一个Controller。这里会创建一个函数块，函数块的目的是加锁
	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// 调用时间处理函数，处理DeltaFIFO
	// 启用Process
	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	// 这里调用了sharedProcessor.run方法
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	// 启动controller--启动reflector--ListerWatcher同步apiServer数据，并且watch apiServer将event加入到DeltaFIFO中
	// 同时调用controller.processLoop函数进行处理
	s.controller.Run(stopCh)
}
```

### Controller做了什么

Controller的Run方法做了三件事：

1. 创建并启动新的Reflector，调用Reflector的Run函数进行ListAndWatch，ListWatch会将监听到的对象事件保存到DeltaFIFO中；
2. 并且会调用controller的processLoop函数完成对DeltaFIFO的消费，即从Reflector的queue成员中依次弹出要处理的对象，并且调用PopProcessFunc函数处理；
3. processFunc其实就是HandleDeltas函数，随后HandleDeltas调用了processDeltas，该函数是核心工作内容：（1）添加（更新/删除）Delta的obj成员到Indexer(也就是本地缓存)中；（2）添加（更新/删除）Delta的obj成员到AddChan（workqueue）中等待sharedProcessor进行一次从queue（chan）中取出进行处理。

```go
// tools/cache/controller.go
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

从上面可以看到，这里的处理函数实际是config配置的Process成员，在shared_informer.go中，这里的Process函数是HandleDeltas函数。

```go
// tools/cache/shared_informer.go Run()
	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		// 这里注册的就是处理Delta的函数（ProcessFunc）
		// 这个函数在Delta从FIFO中被弹出来之前被调用,调用顺序是：
		// 这个也是WatchEvent消费过程：Controller.Run()->Controller.ProcessLoop()->queue.Pop()->sharedIndexInformer.HandleDeltas()
		Process:           s.HandleDeltas,
		WatchErrorHandler: s.watchErrorHandler,
	}
```

HandleDealtas函数调用了processDeltas函数，该函数的具体的实现如下：

```go
// tools/cache/controller.go processDeltas()
func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
	deltas Deltas,
) error {
	// from oldest to newest
	for _, d := range deltas {
		obj := d.Object
		if transformer != nil {
			var err error
			obj, err = transformer(obj)
			if err != nil {
				return err
			}
		}

		// 这里会完成两个工作：
		// 1. 更新Local Store(Indexer);
		// 2. 完成事件分发（这里实际上还没有真正的处理事件，而是调用了sharedIndexInformer.OnAdd/OnUpdate/OnDelete）
		//       在sharedIndexInformer的这些函数中，完成了事件分发。分发的依据就是根据事件的类型（add/update/delete），
		//       具体的看sharedIndexInformer.OnAdd/OnUpdate/OnDelete下面的distribute函数
		switch d.Type {
		case Sync, Replaced, Added, Updated:
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
				handler.OnUpdate(old, obj)
			} else {
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj)
			}
		case Deleted:
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil
}
```

从代码可以看出，该函数主要使用for遍历弹出的一个Deltas（某一个obj的列表）。根据Deltas的数据类型进行区分，主要做了两部分工作：

1. 更新Local Store(Indexer) ，就是将这些Delta添加/更新/删除(有Type和Object,这里只添加/更新/删除了Object,也就是本地对象缓存Indexer中)，所以这里也是保证Indexer中和etcd数据库中的数据一致的实现。实时更新。

2. 完成事件分发。（这里实际上还没有真正的处理事件，而是调用了sharedIndexInformer.OnAdd/OnUpdate/OnDelete）。分发的依据就是根据事件的类型（add/update/delete）。

   > 所谓的分发就是添加到workqueue（开具第一张图中介绍）中等待处理。

这里以Add事件进行说明，追踪的是`handler.OnAdd(obj)`这个方法。

上面的processDeltas函数传入的第一个参数是sharedIndexInformer对象，又因为sharedIndexInformer实现了ResourceEventHandler接口，所以上面的`handler.OnAdd(obj)`最终会调用的是sharedIndexInformer的OnAdd方法，该方法如下：

```go
// tools/cache/shared_informer.go OnAdd()
func (s *sharedIndexInformer) OnAdd(obj interface{}) {
	// Invocation of this function is locked under s.blockDeltas, so it is
	// save to distribute the notification
	s.cacheMutationDetector.AddObject(obj)
	// 这里传入的目标还是从Deltas弹出的Delta对象。调用历史：
	// for:controller.processLoop()->ProcessFunc->sharedIndexInformer.HandleDeltas()
	// ->Controller.processDeltas(这里传入的参数应该是ResourceEventHandler,因为sharedIndexInformer实现了该接口，所以传入的是sharedIndexInformer对象)
	// ->handler.OnAdd/OnUpdate/OnDelete = sharedIndexInformer.OnAdd/OnUpdate/OnDelete
	// ->distribute
	s.processor.distribute(addNotification{newObj: obj}, false)
}
```

<span style="color:orange;">上面的distribute()函数会完成事件的分发。如前所述，distribute其实就是将处理的DeltaFIFO的Obj添加到addChan中，等待处理。</span>

> 这里的obj类型就是DeltaFIFO item(Delta)中的对象，类型如下所示：
>
> <span style="color:red;">[{Add, obj1},{Update, obj1},{Delete, obj1}]</span>

```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			// non-sync messages are delivered to every listener
			listener.add(obj)
		case isSyncing:
			// sync messages are delivered to every syncing listenter
			listener.add(obj)
		default:
			// skipping a sync obj for a non-syncing listener
		}
	}
}
```

上面完成事件分发其实也就是将事件通知添加到addCh中。其中for循环的listener就是processListener。

```go
func (p *processorListener) add(notification interface{}) {
  p.addCh <- notification  // 其实这里传入的notification就是Delta对象:{Add, obj1}
}
```

#### 小结

直至上面流程结束，Controller的任务（controller.run函数）的任务就结束了。总结就是干了三件大事：

1. 创建并启动Reflector，进行ListWatch操作，将结果添加到DeltaFIFO中；
2. 并且会调用controller的processLoop函数完成对DeltaFIFO的消费，即从Reflector的queue成员中依次弹出要处理的对象，并且调用PopProcessFunc函数处理；
3. processFunc其实就是HandleDeltas函数，随后HandleDeltas调用了processDeltas，该函数是核心工作内容：（1）添加（更新/删除）Delta的obj成员到Indexer(也就是本地缓存)中；（2）添加（更新/删除）Delta的obj成员到AddChan（workqueue）中等待sharedProcessor进行一次从queue（chan）中取出进行处理。

### sharedProcessor是如何处理addChan中的对象的

然后是交由sharedProcessor的run函数进行处理。run函数的调用也是在sharedIndexInformer中单独的协程中进行处理：

```go
wg.StartWithChannel(processorStopCh, s.processor.run)
```

这里会启动两个协程同时去处理：

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		// 同时启动两个协程去进行run和pop
		for listener := range p.listeners {
			p.wg.Start(listener.run) // run
			p.wg.Start(listener.pop) // pop
		}
		p.listenersStarted = true
	}()
	<-stopCh

	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()
	for listener := range p.listeners {
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}

	// Wipe out list of listeners since they are now closed
	// (processorListener cannot be re-used)
	p.listeners = nil

	// Reset to false since no listeners are running
	p.listenersStarted = false

	p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```

在pop的时候借助了一个无限大的循环队列（buffer.RingGrowing），原因是：**pop作为addCh 的消费逻辑 必须非常快，而下游nextCh 的消费函数run 执行的速度看业务而定，中间要通过pendingNotifications 缓冲。**

最后run函数也非常简单，就是调用了ResourceEventHandler的方法：

```go
// tools/cache/shared_informer run()
func (p *processorListener) run() {
	// this call blocks until the channel is closed.  When a panic happens during the notification
	// we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
	// the next notification will be attempted.  This is usually better than the alternative of never
	// delivering again.
	stopCh := make(chan struct{})
	wait.Until(func() {
		for next := range p.nextCh {
			// type updateNotification struct
			switch notification := next.(type) {
			case updateNotification:
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			// type addNotification struct
			case addNotification:
				p.handler.OnAdd(notification.newObj)
			// type deleteNotification struct
			case deleteNotification:
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		// the only way to get here is if the p.nextCh is empty and closed
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```

下面的一个图很好的表示了处理流程：

<img width="940" alt="image-20221008152847963" src="https://user-images.githubusercontent.com/54139479/221352344-7464036d-2263-4928-82b6-d262e6194163.png">

> **如果 event 处理较慢，则会导致pendingNotifications 积压，event 处理的延迟增大.**

