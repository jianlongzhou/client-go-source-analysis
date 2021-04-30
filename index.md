![image](https://user-images.githubusercontent.com/41672087/116676748-efbbed00-a9d9-11eb-88ff-2b25120b59b3.png)

## 以下分析：以deployment controller为例分析client-go用法
```go
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
controller := NewController(
   kubeClient, exampleClient,
   kubeInformerFactory.Apps().V1().Deployments())

kubeInformerFactory.Start(stopCh)

if err = controller.Run(2, stopCh); err != nil {
   klog.Fatalf("Error running controller: %s", err.Error())
}
```
### SharedInformerFactory结构
kubeClient:clientset
defaultResync:30s
使用sharedInformerFactory的好处：比如很多个模块都需要使用pod对象，没必要都创建一个pod informer，用factor存储每种资源的一个informer，这里的informer实现是shareIndexInformer
NewSharedInformerFactory调用了NewSharedInformerFactoryWithOptions，将返回一个sharedInformerFactory对象
```go
type sharedInformerFactory struct {
   client           kubernetes.Interface //clientset
   namespace        string //关注的namepace，可以通过WithNamespace Option配置
   tweakListOptions internalinterfaces.TweakListOptionsFunc
   lock             sync.Mutex
   defaultResync    time.Duration //前面传过来的时间，如30s
   customResync     map[reflect.Type]time.Duration //针对每一个informer，用户配置的resync时间，通过WithCustomResyncConfig这个Option配置

   informers map[reflect.Type]cache.SharedIndexInformer //针对每种类型资源存储一个informer，informer的类型是ShareIndexInformer
   // startedInformers is used for tracking which informers have been started.
   // This allows Start() to be called multiple times safely.
   startedInformers map[reflect.Type]bool //每个informer是否都启动了
}
```

sharedInformerFactory对象的关键方法：
#### 创建一个sharedInformerFactory
```go
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
   factory := &sharedInformerFactory{
      client:           client,
      namespace:        v1.NamespaceAll, //可以看到默认是监听所有ns下的指定资源
      defaultResync:    defaultResync,
      informers:        make(map[reflect.Type]cache.SharedIndexInformer),
      startedInformers: make(map[reflect.Type]bool),
      customResync:     make(map[reflect.Type]time.Duration),
   }
   return factory
}
```

#### 启动factory下的所有informer
```go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
   f.lock.Lock()
   defer f.lock.Unlock()

   for informerType, informer := range f.informers {
      if !f.startedInformers[informerType] {
//调用放入是informer的Run方法，具体实现看下面
         go informer.Run(stopCh)
         f.startedInformers[informerType] = true
      }
   }
}
```

#### 等待ShareIndexInformer的cache被同步
等待每一个ShareIndexInformer的cache被同步，具体怎么算同步其实是看ShareIndexInformer里面的store对象的实现
```go
func (f *sharedInformerFactory) WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool {
//获取每一个已经启动的informer
   informers := func() map[reflect.Type]cache.SharedIndexInformer {
      f.lock.Lock()
      defer f.lock.Unlock()

      informers := map[reflect.Type]cache.SharedIndexInformer{}
      for informerType, informer := range f.informers {
         if f.startedInformers[informerType] {
            informers[informerType] = informer
         }
      }
      return informers
   }()

   res := map[reflect.Type]bool{}
   // 等待他们的cache被同步，具体怎么同步的看下面informer的实现
   for informType, informer := range informers {
      res[informType] = cache.WaitForCacheSync(stopCh, informer.HasSynced)
   }
   return res
}
```

#### factory为自己添加informer
添加完成之后，上面factory的start方法就可以启动了
obj:如deployment{}
newFunc:一个可以用来创建指定informer的方法，k8s为每一个内置的对象都实现了这个方法，比如创建deployment的ShareIndexInformer的方法
```go
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
   f.lock.Lock()
   defer f.lock.Unlock()
   //根据对象类型判断factory中是否已经有对应informer
   informerType := reflect.TypeOf(obj)
   informer, exists := f.informers[informerType]
   if exists {
      return informer
   }

   resyncPeriod, exists := f.customResync[informerType]
   if !exists {
      resyncPeriod = f.defaultResync
   }
   //没有就根据newFunc创建一个，并存在map中
   informer = newFunc(f.client, resyncPeriod)
   f.informers[informerType] = informer

   return informer
}
```

##### deployment的shareIndexInformer对应的newFunc的实现
```go
//可以看到创建deploymentInformer时传递了一个带索引的缓存，附带了一个namespace索引，后面可以了解带索引的缓存实现，比如可以支持查询：某个namespace下的所有pod
func (f *deploymentInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
   return NewFilteredDeploymentInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

// 先看看下面的shareIndexInformer结构
func NewFilteredDeploymentInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
   return cache.NewSharedIndexInformer(
      //定义对象的ListWatch方法，这里直接用的是clientset中的方法
      &cache.ListWatch{
         ListFunc: func(options v1.ListOptions) (runtime.Object, error) {
            if tweakListOptions != nil {
               tweakListOptions(&options)
            }
            return client.AppsV1beta1().Deployments(namespace).List(options)
         },
         WatchFunc: func(options v1.ListOptions) (watch.Interface, error) {
            if tweakListOptions != nil {
               tweakListOptions(&options)
            }
            return client.AppsV1beta1().Deployments(namespace).Watch(options)
         },
      },
      &appsv1beta1.Deployment{},
      resyncPeriod, //创建factory是制定的时间，30s
      indexers,
   )
}
```

### shareIndexInformer结构
indexer：底层缓存，其实就是一个map记录对象，再通过一些其他map在插入删除对象是根据索引函数维护索引key如ns与对象pod的关系
controller：informer内部的一个controller，这个controller包含reflector：根据用户定义的ListWatch方法获取对象并更新增量队列DeltaFIFO
processor：知道如何处理DeltaFIFO队列中的对象，实现是sharedProcessor{}
listerWatcher：知道如何list对象和watch对象的方法
objectType：deployment{}
resyncCheckPeriod: factory创建是指定的defaultResync，给自己的controller的reflector每隔多少s调用shouldResync
defaultEventHandlerResyncPeriod：factory创建是指定的defaultResync，通过AddEventHandler增加的handler的resync默认值
```go
type sharedIndexInformer struct {
   indexer    Indexer //informer中的底层缓存
   controller Controller

   processor             *sharedProcessor
   cacheMutationDetector MutationDetector //忽略，测试用

   // This block is tracked to handle late initialization of the controller
   listerWatcher ListerWatcher
   objectType    runtime.Object

   // resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
   // shouldResync to check if any of our listeners need a resync.
   resyncCheckPeriod time.Duration
   // defaultEventHandlerResyncPeriod is the default resync period for any handlers added via
   // AddEventHandler (i.e. they don't specify one and just want to use the shared informer's default
   // value).
   defaultEventHandlerResyncPeriod time.Duration
   // clock allows for testability
   clock clock.Clock

   started, stopped bool
   startedLock      sync.Mutex

   // blockDeltas gives a way to stop all event distribution so that a late event handler
   // can safely join the shared informer.
   blockDeltas sync.Mutex
}
```
sharedIndexInformer对象的关键方法：

#### sharedIndexInformer的Run方法
前面factory的start就是调了这个
```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()
   //创建一个DeltaFIFO，用于shareIndexInformer.controller.refector
   fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)
   //shareIndexInformer中的controller的配置
   cfg := &Config{
      Queue:            fifo,
      ListerWatcher:    s.listerWatcher,
      ObjectType:       s.objectType,
      FullResyncPeriod: s.resyncCheckPeriod, // 30s
      RetryOnError:     false,
      ShouldResync:     s.processor.shouldResync,

      Process: s.HandleDeltas,
   }

   func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()
      // 这里New一个具体的controller
      s.controller = New(cfg)
      s.controller.(*controller).clock = s.clock
      s.started = true
   }()

   // Separate stop channel because Processor should be stopped strictly after controller
   processorStopCh := make(chan struct{})
   var wg wait.Group
   defer wg.Wait()              // Wait for Processor to stop
   defer close(processorStopCh) // Tell Processor to stop
   // 调用process.run启动所有的listener，回调用户配置的EventHandler
   wg.StartWithChannel(processorStopCh, s.processor.run)

   // 启动controller
   s.controller.Run(stopCh)
}
```

#### 为shareIndexInformer创建controller
创建Controller的New方法会生成一个reflector
```go
func New(c *Config) Controller {
   ctlr := &controller{
      config: *c,
      clock:  &clock.RealClock{},
   }
   return ctlr
}
更多字段的配置是在Run的时候
func (c *controller) Run(stopCh <-chan struct{}) {
   // 使用config创建一个Reflector
   r := NewReflector(
      c.config.ListerWatcher, // deployment的listWatch方法
      c.config.ObjectType, // deployment{}
      c.config.Queue, //DeltaFIFO
      c.config.FullResyncPeriod, //30s
   )
   r.ShouldResync = c.config.ShouldResync //来自sharedProcessor的方法
   r.clock = c.clock

   c.reflectorMutex.Lock()
   c.reflector = r
   c.reflectorMutex.Unlock()

   var wg wait.Group
   defer wg.Wait()
   // 启动reflector，执行ListWatch方法
   wg.StartWithChannel(stopCh, r.Run)
   // 不断执行processLoop，这个方法其实就是从DeltaFIFO pop出对象，再调用reflector的Process（其实是shareIndexInformer的HandleDeltas方法）处理
   wait.Until(c.processLoop, time.Second, stopCh)
}
```

#### DeltaFIFO pop出来的对象处理逻辑
先看看controller怎么处理DeltaFIFO中的对象
注意DeltaFIFO中的Deltas的结构，是一个slice，保存同一个对象的所有增量事件
![image](https://user-images.githubusercontent.com/41672087/116666059-19224c00-a9cd-11eb-945c-955cce9eacd1.png)
```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
   s.blockDeltas.Lock()
   defer s.blockDeltas.Unlock()

   // from oldest to newest
   for _, d := range obj.(Deltas) {
      switch d.Type {
      case Sync, Added, Updated:
         isSync := d.Type == Sync
         // 对象先通过shareIndexInformer中的indexer更新到缓存
         if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
            if err := s.indexer.Update(d.Object); err != nil {
               return err
            }
            // 如果informer的本地缓存更新成功，那么就调用shareProcess分发对象给用户自定义controller处理
            // 可以看到，对EventHandler来说，本地缓存已经存在该对象就认为是update
            s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
         } else {
            if err := s.indexer.Add(d.Object); err != nil {
               return err
            }
            s.processor.distribute(addNotification{newObj: d.Object}, isSync)
         }
      case Deleted:
         if err := s.indexer.Delete(d.Object); err != nil {
            return err
         }
         s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
      }
   }
   return nil
}
```

#### reflector.run发起ListWatch
```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
   // 以版本号ResourceVersion=0开始首次list
   options := metav1.ListOptions{ResourceVersion: "0"}

   if err := func() error {
      initTrace := trace.New("Reflector ListAndWatch", trace.Field{"name", r.name})
      var list runtime.Object
      go func() {
         // 获取list的结果
         list, err = pager.List(context.Background(), options)
         close(listCh)
      }()
      listMetaInterface, err := meta.ListAccessor(list)
      // 根据结果更新版本号，用于接下来的watch
      resourceVersion = listMetaInterface.GetResourceVersion()
      items, err := meta.ExtractList(list)
      // 这里的syncWith是把首次list到的结果通过DeltaFIFO的Replce方法批量添加到队列
      // 队列提供了Resync方法用于判断Replace批量插入的对象是否都pop出去了，factory/informer的WaitForCacheSync就是调用了DeltaFIFO的的Resync方法
      if err := r.syncWith(items, resourceVersion); err != nil {
         return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
      }
      r.setLastSyncResourceVersion(resourceVersion)
   }(); err != nil {
      return err
   }

   for {
      // start the clock before sending the request, since some proxies won't flush headers until after the first watch event is sent
      start := r.clock.Now()
      w, err := r.listerWatcher.Watch(options)
      // watchhandler处理watch到的数据，即把对象根据watch.type增加到DeltaFIFO中
      if err := r.watchHandler(start, w, &resourceVersion, resyncerrc, stopCh); err != nil {
         if err != errorStopRequested {
            switch {
            case apierrs.IsResourceExpired(err):
               klog.V(4).Infof("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
            default:
               klog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
            }
         }
         return nil
      }
   }
}
```

### 底层缓存的实现
shareIndexInformer中带有一个缓存indexer
查看最前面的类图，我们可以知道：

Indexer、Queue接口和cache结构体都实现了顶层的Store接口
cache结构体持有threadSafeStore对象，该结构体是线程安全的，具备索引查找能力的map

threadSafeMap的结构如下：
items:存储具体的对象，比如key为ns/podName，value为pod{}
Indexers:一个map[string]IndexFunc结构，其中key为索引的名称，如’namespace’字符串，value则是一个具体的索引函数
Indices:一个map[string]Index结构，其中key也是索引的名称，value是一个map[string]sets.String结构，其中key是具体的namespace，如default这个ns，vlaue则是这个ns下的按照索引函数求出来的值的集合，比如default这个ns下的所有pod对象名称

通过在向items插入对象的过程中，遍历所有的Indexers中的索引函数，根据索引函数存储索引key到value的集合关系，以下图式结构可以很好的说明

![image](https://user-images.githubusercontent.com/41672087/116666278-5981ca00-a9cd-11eb-9570-8ee6eb447d05.png)
```go
type threadSafeMap struct {
   lock  sync.RWMutex
   items map[string]interface{}

   // indexers maps a name to an IndexFunc
   indexers Indexers
   // indices maps a name to an Index
   indices Indices
}

// Indexers maps a name to a IndexFunc
type Indexers map[string]IndexFunc

// Indices maps a name to an Index
type Indices map[string]Index
type Index map[string]sets.String
```

#### 缓存中增加对象
以向上面的结构中增加一个对象为例
```go
func (c *threadSafeMap) Add(key string, obj interface{}) {
   c.lock.Lock()
   defer c.lock.Unlock()
   oldObject := c.items[key]
   //存储对象
   c.items[key] = obj
   //更新索引
   c.updateIndices(oldObject, obj, key)
}

// updateIndices modifies the objects location in the managed indexes, if this is an update, you must provide an oldObj
// updateIndices must be called from a function that already has a lock on the cache
func (c *threadSafeMap) updateIndices(oldObj interface{}, newObj interface{}, key string) {
   // if we got an old object, we need to remove it before we add it again
   if oldObj != nil {
      // 这是一个更新操作，先删除原对象的索引记录
      c.deleteFromIndices(oldObj, key)
   }
   // 枚举所有添加的索引函数
   for name, indexFunc := range c.indexers {
      //根据索引函数计算obj对应的
      indexValues, err := indexFunc(newObj)
      if err != nil {
         panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
      }
      index := c.indices[name]
      if index == nil {
         index = Index{}
         c.indices[name] = index
      }
      //索引函数计算出多个value，也可能是一个，比如pod的ns就只有一个值，pod的label可能就有多个值
      for _, indexValue := range indexValues {
         //比如namespace索引，根据indexValue=default，获取default对应的ji he再把当前对象插入
         set := index[indexValue]
         if set == nil {
            set = sets.String{}
            index[indexValue] = set
         }
         set.Insert(key)
      }
   }
}
```

#### MetaNamespaceIndexFunc索引函数
一个典型的索引函数MetaNamespaceIndexFunc，方便查询时可以根据namespace获取该namespace下的所有对象
```go
// MetaNamespaceIndexFunc is a default index function that indexes based on an object's namespace
func MetaNamespaceIndexFunc(obj interface{}) ([]string, error) {
   meta, err := meta.Accessor(obj)
   if err != nil {
      return []string{""}, fmt.Errorf("object has no meta: %v", err)
   }
   return []string{meta.GetNamespace()}, nil
}
```

#### Index方法利用索引查找对象
提供利用索引来查询的能力，Index方法可以根据索引名称和对象，查询所有的关联对象
例如通过Index(“namespace”, &metav1.ObjectMeta{Namespace: namespace})获取指定ns下的所有对象
具体可以参考tools/cache/listers.go#ListAllByNamespace
```go
func (c *threadSafeMap) Index(indexName string, obj interface{}) ([]interface{}, error) {
   c.lock.RLock()
   defer c.lock.RUnlock()

   indexFunc := c.indexers[indexName]
   if indexFunc == nil {
      return nil, fmt.Errorf("Index with name %s does not exist", indexName)
   }

   indexKeys, err := indexFunc(obj)
   if err != nil {
      return nil, err
   }
   index := c.indices[indexName]

   var returnKeySet sets.String
   //例如namespace索引
   if len(indexKeys) == 1 {
      // In majority of cases, there is exactly one value matching.
      // Optimize the most common path - deduping is not needed here.
      returnKeySet = index[indexKeys[0]]
   //例如label索引
   } else {
      // Need to de-dupe the return list.
      // Since multiple keys are allowed, this can happen.
      returnKeySet = sets.String{}
      for _, indexKey := range indexKeys {
         for key := range index[indexKey] {
            returnKeySet.Insert(key)
         }
      }
   }

   list := make([]interface{}, 0, returnKeySet.Len())
   for absoluteKey := range returnKeySet {
      list = append(list, c.items[absoluteKey])
   }
   return list, nil
}
```

### reflector中的deltaFIFO实现
shareIndexInformer.controller.reflector中的deltaFIFO实现
items：记录deltaFIFO中的对象，注意map的value是一个Delta slice
queue：记录上面items中的key
populated：队列中是否填充过数据，LIST时调用Replace或调用Delete/Add/Update都会置为true
initialPopulationCount：前面首次List的时候获取到的数据就会调用Replace批量增加到队列，同时设置initialPopulationCount为List到的对象数量，每次Pop出来会减一，由于判断是否把首次批量插入的数据都POP出去了
keyFunc：知道怎么从对象中解析出对应key的函数
knownObjects：这个其实就是shareIndexInformer中的indexer底层缓存的引用，可以认为和etcd中的数据一致
```go
type DeltaFIFO struct {
   // lock/cond protects access to 'items' and 'queue'.
   lock sync.RWMutex
   cond sync.Cond

   // We depend on the property that items in the set are in
   // the queue and vice versa, and that all Deltas in this
   // map have at least one Delta.
   // 这里的Deltas是[]Delta类型
   items map[string]Deltas
   queue []string

   // populated is true if the first batch of items inserted by Replace() has been populated
   // or Delete/Add/Update was called first.
   populated bool
   // initialPopulationCount is the number of items inserted by the first call of Replace()
   initialPopulationCount int

   // keyFunc is used to make the key used for queued item
   // insertion and retrieval, and should be deterministic.
   keyFunc KeyFunc

   // knownObjects list keys that are "known", for the
   // purpose of figuring out which items have been deleted
   // when Replace() or Delete() is called.
   // 这个其实就是shareIndexInformer中的indexer底层缓存的引用
   knownObjects KeyListerGetter

   // Indication the queue is closed.
   // Used to indicate a queue is closed so a control loop can exit when a queue is empty.
   // Currently, not used to gate any of CRED operations.
   closed     bool
   closedLock sync.Mutex
}

type Delta struct {
   Type   DeltaType
   Object interface{}
}

// Deltas is a list of one or more 'Delta's to an individual object.
// The oldest delta is at index 0, the newest delta is the last one.
type Deltas []Delta
```

DeltaFIFO关键的方法：
每次从DeltaFIFO Pop出一个对象，f.initialPopulationCount会减一，初始值为List时的对象数量
前面的Informer的WaitForCacheSync最终就是调用了这个HasSynced方法，因为前面Pop出对象的处理方法HandleDeltas中，
会先调用indexder把对象存起来，所以这个HasSynced相当于判断本地缓存是否首次同步完成

#### deltaFIFO是否sync
即从reflector list到的数据是否pop完
```go
func (f *DeltaFIFO) HasSynced() bool {
   f.lock.Lock()
   defer f.lock.Unlock()
   return f.populated && f.initialPopulationCount == 0
}

在队列中给指定的对象append一个Delta
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
   id, err := f.KeyOf(obj)
   if err != nil {
      return KeyError{obj, err}
   }

   newDeltas := append(f.items[id], Delta{actionType, obj})
   newDeltas = dedupDeltas(newDeltas)

   if len(newDeltas) > 0 {
      if _, exists := f.items[id]; !exists {
         f.queue = append(f.queue, id)
      }
      f.items[id] = newDeltas
      f.cond.Broadcast()
   } else {
      // We need to remove this from our map (extra items in the queue are
      // ignored if they are not in the map).
      delete(f.items, id)
   }
   return nil
}
```

#### 从deltaFIFO pop出对象
从队列中Pop出一个方法，并由函数process来处理，就是shareIndexInformer的HandleDeltas
```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
   f.lock.Lock()
   defer f.lock.Unlock()
   for {
      for len(f.queue) == 0 {
         // When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
         // When Close() is called, the f.closed is set and the condition is broadcasted.
         // Which causes this loop to continue and return from the Pop().
         if f.IsClosed() {
            return nil, ErrFIFOClosed
         }

         f.cond.Wait()
      }
      id := f.queue[0]
      f.queue = f.queue[1:]
      if f.initialPopulationCount > 0 {
         f.initialPopulationCount--
      }
      item, ok := f.items[id]
      if !ok {
         // Item may have been deleted subsequently.
         continue
      }
      delete(f.items, id)
      err := process(item)
      // 如果没有处理成功，那么就会重新加到deltaFIFO队列中
      if e, ok := err.(ErrRequeue); ok {
         f.addIfNotPresent(id, item)
         err = e.Err
      }
      // Don't need to copyDeltas here, because we're transferring
      // ownership to the caller.
      return item, err
   }
}
```

#### 向deltaFIFO批量插入对象
批量向队列插入数据的方法，当有knownObjects存在时，knownObjects才表示已有数据
```go
func (f *DeltaFIFO) Replace(list []interface{}, resourceVersion string) error {
   f.lock.Lock()
   defer f.lock.Unlock()
   keys := make(sets.String, len(list))

   for _, item := range list {
      key, err := f.KeyOf(item)
      if err != nil {
         return KeyError{item, err}
      }
      keys.Insert(key)
      // 通过Replace添加到队列的Delta type都是Sync
      if err := f.queueActionLocked(Sync, item); err != nil {
         return fmt.Errorf("couldn't enqueue object: %v", err)
      }
   }

// 底层的缓存不应该会是nil
   if f.knownObjects == nil {
      // Do deletion detection against our own list.
      queuedDeletions := 0
      for k, oldItem := range f.items {
         if keys.Has(k) {
            continue
         }
         // 当knownObjects为空时，如果item中存在对象不在新来的list中，那么该对象被认为要被删除
         var deletedObj interface{}
         if n := oldItem.Newest(); n != nil {
            deletedObj = n.Object
         }
         queuedDeletions++
         if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
            return err
         }
      }

      if !f.populated {
         f.populated = true
         // While there shouldn't be any queued deletions in the initial
         // population of the queue, it's better to be on the safe side.
         f.initialPopulationCount = len(list) + queuedDeletions
      }

      return nil
   }

   // Detect deletions not already in the queue.
   knownKeys := f.knownObjects.ListKeys()
   // 记录这次替换相当于在缓存中删除多少对象
   queuedDeletions := 0
   // 枚举每一个缓存对象的key，看看在不在即将用来替换delta队列的list中
   for _, k := range knownKeys {
      if keys.Has(k) {
         continue
      }
      // 对象在缓存，但不在list中，说明替换操作完成后，这个对象相当于被删除了
     // 注意这里的所谓替换，对deltaFIFO来说，是给队列中的对应对象增加一个delete增量queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj})
   // 真正删除缓存是在shareIndexInformer中的HandleDeltas中
      deletedObj, exists, err := f.knownObjects.GetByKey(k)
      if err != nil {
         deletedObj = nil
         klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
      } else if !exists {
         deletedObj = nil
         klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
      }
      queuedDeletions++
      if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
         return err
      }
   }
     // 设置f.initialPopulationCount，大于0表示首次插入的对象还没有pop出去
   if !f.populated {
      f.populated = true
      f.initialPopulationCount = len(list) + queuedDeletions
   }

   return nil
}
```

#### Resync方法
```go
// 所谓的resync，其实就是把knownObjects即缓存中的对象全部再通过queueActionLocked(Sync, obj)加到队列
func (f *DeltaFIFO) Resync() error {
   f.lock.Lock()
   defer f.lock.Unlock()

   if f.knownObjects == nil {
      return nil
   }

   keys := f.knownObjects.ListKeys()
   for _, k := range keys {
      if err := f.syncKeyLocked(k); err != nil {
         return err
      }
   }
   return nil
}

func (f *DeltaFIFO) syncKeyLocked(key string) error {
   obj, exists, err := f.knownObjects.GetByKey(key)

   // If we are doing Resync() and there is already an event queued for that object,
   // we ignore the Resync for it. This is to avoid the race, in which the resync
   // comes with the previous value of object (since queueing an event for the object
   // doesn't trigger changing the underlying store <knownObjects>.
   id, err := f.KeyOf(obj)
   if err != nil {
      return KeyError{obj, err}
   }
   // 如上述注释，在resync时，如果deltaFIFO中该对象还存在其他delta没处理，那么忽略这次的resync
   // 因为调用queueActionLocked是增加delta是通过append的，且处理对象的增量delta时，是从oldest到newdest的
   // 所以如果某个对象还存在增量没处理，再append就可能导致后处理的delta是旧的对象
   if len(f.items[id]) > 0 {
      return nil
   }

   if err := f.queueActionLocked(Sync, obj); err != nil {
      return fmt.Errorf("couldn't queue object: %v", err)
   }
   return nil
}
```

### shareProcess结构
shareIndexInformer中具有一个shareProcess结构，用于分发deltaFIFO的对象，调用用户配置的EventHandler处理

可以看到shareIndexInformer中的process直接通过&sharedProcessor{clock: realClock}初始化
如下为sharedProcessor结构：
listenersStarted：listeners中包含的listener是否都已经启动了
listeners：已添加的listener
syncingListeners：正在处于同步状态的listener
listeners和syncingListeners保存的listener是一致的，区别在于当从deltaFIFO分发过来的对象是由resync触发的，那么会由syncingListeners的来处理
```go
// NewSharedIndexInformer creates a new instance for the listwatcher.
func NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
   realClock := &clock.RealClock{}
   sharedIndexInformer := &sharedIndexInformer{
      processor:                       &sharedProcessor{clock: realClock},
      indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
      listerWatcher:                   lw,
      objectType:                      objType,
      resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
      defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
      cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", objType)),
      clock:                           realClock,
   }
   return sharedIndexInformer
}

type sharedProcessor struct {
   listenersStarted bool
   listenersLock    sync.RWMutex
   listeners        []*processorListener
   syncingListeners []*processorListener
   clock            clock.Clock
   wg               wait.Group
}
```

#### 为sharedProcessor添加listener
在sharedProcessor中添加一个listener
```go
func (p *sharedProcessor) addListenerLocked(listener *processorListener) {
   p.listeners = append(p.listeners, listener)
   p.syncingListeners = append(p.syncingListeners, listener)
}
```

#### 启动sharedProcessor中的的listener
sharedProcessor启动所有的listener，可以看到syncingListeners中的listener是没有被启动的
是通过调用listener.run和listener.pop来启动一个listener，两个方法具体作用看下文processorListener说明
```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
   func() {
      p.listenersLock.RLock()
      defer p.listenersLock.RUnlock()
      for _, listener := range p.listeners {
         p.wg.Start(listener.run)
         p.wg.Start(listener.pop)
      }
      p.listenersStarted = true
   }()
   <-stopCh
   p.listenersLock.RLock()
   defer p.listenersLock.RUnlock()
   for _, listener := range p.listeners {
      close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
   }
   p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```

#### sharedProcessor分发对象
```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
   p.listenersLock.RLock()
   defer p.listenersLock.RUnlock()

   if sync {
      for _, listener := range p.syncingListeners {
         listener.add(obj)
      }
   } else {
      for _, listener := range p.listeners {
         listener.add(obj)
      }
   }
}
```

### processorListener结构
sharedProcessor中的listener具体的类型：运转逻辑就是把用户通过addCh增加的事件发送到nextCh供run方法取出回调Eventhandler，因为addCh和nectCh都是无缓冲channel，所以中间引入ringBuffer做缓存
processorListener是sharedIndexInformer调用AddEventHandler时创建并添加到sharedProcess
对于一个Informer，可以多次调用AddEventHandler来添加多个listener，但我们的一般的使用场景应该都只会添加一个，作用都是类似enqueue到workQueue
addCh和nextCh：都将被初始化为无缓冲的chan
pendingNotifications：一个无容量限制的环形缓冲区，可以理解为可以无限存储的队列，用来存储deltaFIFO分发过来的消息
```go
type processorListener struct {
   nextCh chan interface{}
   addCh  chan interface{}

   handler ResourceEventHandler

   // pendingNotifications is an unbounded ring buffer that holds all notifications not yet distributed.
   // There is one per listener, but a failing/stalled listener will have infinite pendingNotifications
   // added until we OOM.
   // TODO: This is no worse than before, since reflectors were backed by unbounded DeltaFIFOs, but
   // we should try to do something better.
   pendingNotifications buffer.RingGrowing

   // requestedResyncPeriod is how frequently the listener wants a full resync from the shared informer
   requestedResyncPeriod time.Duration
   // resyncPeriod is how frequently the listener wants a full resync from the shared informer. This
   // value may differ from requestedResyncPeriod if the shared informer adjusts it to align with the
   // informer's overall resync check period.
   resyncPeriod time.Duration
   // nextResync is the earliest time the listener should get a full resync
   nextResync time.Time
   // resyncLock guards access to resyncPeriod and nextResync
   resyncLock sync.Mutex
}
```

#### 在listener中添加事件
shareProcessor中的distribute方法调用的是listener的add来向addCh增加消息，注意addCh是无缓冲的chan，依赖pop不断从addCh取出数据
```go
func (p *processorListener) add(notification interface{}) {
   p.addCh <- notification
}
```

#### listener的run方法回调EventHandler
listener的run方法不断的从nextCh中获取notification，并根据notification的类型来调用用户自定的EventHandler
```go
func (p *processorListener) run() {
   // this call blocks until the channel is closed.  When a panic happens during the notification
   // we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
   // the next notification will be attempted.  This is usually better than the alternative of never
   // delivering again.
   stopCh := make(chan struct{})
   wait.Until(func() {
      // this gives us a few quick retries before a long pause and then a few more quick retries
      err := wait.ExponentialBackoff(retry.DefaultRetry, func() (bool, error) {
         for next := range p.nextCh {
            switch notification := next.(type) {
            case updateNotification:
               p.handler.OnUpdate(notification.oldObj, notification.newObj)
            case addNotification:
               p.handler.OnAdd(notification.newObj)
            case deleteNotification:
               p.handler.OnDelete(notification.oldObj)
            default:
               utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
            }
         }
         // the only way to get here is if the p.nextCh is empty and closed
         return true, nil
      })

      // the only way to get here is if the p.nextCh is empty and closed
      if err == nil {
         close(stopCh)
      }
   }, 1*time.Minute, stopCh)
}
```

#### pop方法：addCh到nextCh的对象传递
这个pop的逻辑相对比较绕，最终目的就是把分发到addCh的数据从nextCh或者pendingNotifications取出来
notification变量记录下一次要被放到p.nextCh供pop方法取出的对象
开始seletct时必然只有case2可能ready
Case2做的事可以描述为：从p.addCh获取对象，如果临时变量notification还是nil，说明需要往notification赋值，供case1推送到p.nextCh
如果notification已经有值了，那个当前从p.addCh取出的值要先放到环形缓冲区中

Case1做的事可以描述为：看看能不能把临时变量notification推送到nextCh（nil chan会阻塞在读写操作上），可以写的话，说明这个nextCh是p.nextCh，写成功之后，需要从缓存中取出一个对象放到notification为下次执行这个case做准备，如果缓存是空的，通过把nextCh chan设置为nil来禁用case1，以便case2位notification赋值

```go
func (p *processorListener) pop() {
   defer utilruntime.HandleCrash()
   defer close(p.nextCh) // Tell .run() to stop

   //nextCh没有利用make初始化，将阻塞在读和写上
   var nextCh chan<- interface{}
   //notification初始值为nil
   var notification interface{}
   for {
      select {
      // 执行这个case，相当于给p.nextCh添加来自p.addCh的内容
      case nextCh <- notification:
         // Notification dispatched
         var ok bool
         //前面的notification已经加到p.nextCh了， 为下一次这个case再次ready做准备
         notification, ok = p.pendingNotifications.ReadOne()
         if !ok { // Nothing to pop
            nextCh = nil // Disable this select case
         }
      //第一次select只有这个case ready
      case notificationToAdd, ok := <-p.addCh:
         if !ok {
            return
         }
         if notification == nil { // No notification to pop (and pendingNotifications is empty)
            // Optimize the case - skip adding to pendingNotifications
            //为notification赋值
            notification = notificationToAdd
            //唤醒第一个case
            nextCh = p.nextCh
         } else { // There is already a notification waiting to be dispatched
            //select没有命中第一个case，那么notification就没有被消耗，那么把从p.addCh获取的对象加到缓存中
            p.pendingNotifications.WriteOne(notificationToAdd)
         }
      }
   }
}
```

Ref:
https://zhuanlan.zhihu.com/p/202611841
https://cloudnative.to/blog/client-go-study/
https://gzh.readthedocs.io/en/latest/blogs/Kubernetes/2020-10-11-client-go%E7%B3%BB%E5%88%97%E4%B9%8B1---client-go%E4%BB%A3%E7%A0%81%E8%AF%A6%E8%A7%A3.html
https://www.cnblogs.com/f-ck-need-u/p/9994508.html

![image](https://user-images.githubusercontent.com/41672087/116673725-53dcb200-a9d6-11eb-9d67-9e30b6ea33e5.png)
