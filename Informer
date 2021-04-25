https://zhuanlan.zhihu.com/p/202611841

### 以下分析：利用已有的dpeloyment资源，实现自定义controller逻辑
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
controller := NewController(
   kubeClient, exampleClient,
   kubeInformerFactory.Apps().V1().Deployments())

kubeInformerFactory.Start(stopCh)

if err = controller.Run(2, stopCh); err != nil {
   klog.Fatalf("Error running controller: %s", err.Error())
}

### kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
kubeClient:clientset
defaultResync:30s

使用sharedInformerFactory的好处：比如很多个模块都需要使用pod对象，没必要都创建一个pod informer，用factor存储每种资源的一个informer，这里的informer实现是shareIndexInformer
NewSharedInformerFactory调用了NewSharedInformerFactoryWithOptions，将返回一个sharedInformerFactory对象
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

sharedInformerFactory对象的关键方法：

#### 创建一个sharedInformerFactory
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

#### 启动factory下的所有informer
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

#### 等待每一个ShareIndexInformer的cache被同步，具体怎么算同步其实是看ShareIndexInformer里面的store对象的实现
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
// 等待他们的cache被同步
   for informType, informer := range informers {
      res[informType] = cache.WaitForCacheSync(stopCh, informer.HasSynced)
   }
   return res
}


#### InformerFor: factory为自己添加informer的方法，添加完成之后，上面factor的start方法就可以启动了
obj:如deployment{}
newFunc:一个可以用来创建指定informer的方法，k8s为每一个内置的对象都实现了这个方法，比如创建deployment的ShareIndexInformer的方法在informers/apps/v1beta1/deployment.go#defaultInformer
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

##### deployment的shareIndexInformer对应的newFunc的实现
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


##### shareIndexInformer结构
indexer：底层缓存，其实就是一个map记录对象，再通过一些其他map在插入删除对象是根据索引函数维护索引key如ns与对象pod的关系
controller：informer内部的一个controller，这个controller包含reflector：根据用户定义的ListWatch方法获取对象并更新增量队列DeltaFIFO
processor：知道如何处理DeltaFIFO队列中的对象，实现是sharedProcessor{}
listerWatcher：知道如何list对象和watch对象的方法
objectType：deployment{}
resyncCheckPeriod: factory创建是指定的defaultResync，给自己的controller的reflector每隔多少s调用shouldResync
defaultEventHandlerResyncPeriod：factory创建是指定的defaultResync，通过AddEventHandler增加的handler的resync默认值
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

sharedIndexInformer对象的关键方法：
##### sharedIndexInformer的Run方法，前面factory的start就是调了这个
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
// 调用process.run方法不断分发对象，回掉用户配置的EventHandler
   wg.StartWithChannel(processorStopCh, s.processor.run)

// 启动controller
   s.controller.Run(stopCh)
}


##### 为shareIndexInformer New一个包含reflector的controller
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


##### 先看看controller怎么处理DeltaFIFO中的对象
注意DeltaFIFO中的Deltas的结构，是一个slice，保存同一个对象的所有增量事件

func (s *sharedIndexInforme
r) HandleDeltas(obj interface{}) error {
   s.blockDeltas.Lock()
   defer s.blockDeltas.Unlock()

   // from oldest to newest
   for _, d := range obj.(Deltas) {
// 这里对象的type是怎么决定的，比如update是根据谁和谁比较认为是update？
      switch d.Type {
      case Sync, Added, Updated:
         isSync := d.Type == Sync
        // 对象先通过shareIndexInformer中的indexer更新到缓存
         if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
            if err := s.indexer.Update(d.Object); err != nil {
               return err
            }
        // 如果informer的本地缓存更新成功，那么就调用shareProcess分发对象给用户自定义controller处理
        // 可以看到，对EventHandler来说，本地缓存存在该对象就认为是update
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



##### reflector.run会执行如下ListWatch方法
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
      // watchhandler处理watch到的数据，将对象根据watch.type增加到DeltaFIFO中
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
