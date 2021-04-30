### 以下分析：利用已有的dpeloyment资源，实现自定义controller逻辑
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
