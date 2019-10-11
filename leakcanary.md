# LeakCanary原理浅析



废话少说，直接从入口开始：

```
public static RefWatcher install(Application application) {
  return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
      .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
      .buildAndInstall();
}
```

builder模式构建了一个RefWatcher对象,`listenerServiceClass()`方法绑定了一个后台服务`DisplayLeakService`

这个服务主要用来分析内存泄漏结果并发送通知。你可以继承并重写这个类来进行一些自定义操作，比如上传分析结果等。

我们看最后buildAndInstall\(\)方法：

```
  public RefWatcher buildAndInstall() {
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      LeakCanary.enableDisplayLeakActivity(context);
      ActivityRefWatcher.installOnIcsPlus((Application) context, refWatcher);
    }
    return refWatcher;
  }
```

build\(\)方法主要是配置：

`watchExecutor` : 线程控制器，在 `onDestroy()`之后并且主线程空闲时执行内存泄漏检测

`debuggerControl`: 判断是否处于调试模式，调试模式中不会进行内存泄漏检测

`gcTrigger`: 用于 `GC`

`watchExecutor`首次检测到可能的内存泄漏，会主动进行GC,GC之后会再检测一次，仍然泄漏的判定为内存泄漏，进行后续操作

`heapDumper`: `dump`内存泄漏处的 `heap`信息，写入 `hprof`文件

`heapDumpListener`: 解析完`hprof`文件并通知`DisplayLeakService`弹出提醒

`excludedRefs`: 排除可以忽略的泄漏路径

  


