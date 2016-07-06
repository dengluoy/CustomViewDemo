LeakCanary用法
===========
***


##监控Activity泄漏

``` java

	public class ExampleApplication extends Application {
	
    public static RefWatcher getRefWatcher(Context context) {
        ExampleApplication application = (ExampleApplication) context.getApplicationContext();
        return application.refWatcher;
    }

    private RefWatcher refWatcher;

    @Override public void onCreate() {
        super.onCreate();
        refWatcher = LeakCanary.install(this);
    }
}
```



`LeakCanary.install()` 返回一个配置好了的`RefWatcher`实例。它同时安装了`ActivityRefWatcher`来监控Activity泄露。即当`Activity.onDestroy()`被调用之后，如果这个Activity没有被销毁，Leaks_APP中显示。


___LeakCanary 自动检测 Activity 泄漏只支持 Android ICS 以上版本。因为Application.registerActivityLifecycleCallbacks() 是在 API 14 引入的。如果要在 ICS 之前监测 Activity 泄漏，可以重载 Activity.onDestroy() 方法，然后在这个方法里调用 RefWatcher.watch(this) 来实现___




##监控Fragment泄漏

``` java
	
	public abstract class BaseFragment extends Fragment {

    @Override 
    public void onDestroy() {
        super.onDestroy();
        RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
        refWatcher.watch(this);
    }
}
```

与Activity同理在Leask__APP中查看

##监控其他对象泄漏

``` java

	....
	//LeakCanary最大的一个缺陷就是我需要知道一个对象的确切生命周期，并在我们认为其生命周期应结
	//束的时间点进行watch。对于Android平台来说，常见的场景就是Activity、Fragment等对象的
	//onDestory中
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(someObjNeedGced);
```

___这里解释为啥LeakCanary在进行内存泄漏时需要手动调用watch去监测一个对象，因为我们需要一个合适的时机去用一个弱引用把这个对象使用ReferenceQueue监测起来___




##集成LeakCanary库

``` java

	dependencies {
    debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
    releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
}

```

在 debug 版本上，集成 LeakCanary 库，并执行内存泄漏监测，而在 release 版本上，集成一个无操作的 wrapper ，这样对程序性能就不会有影响。


###LeakCanary 的机制如下：

* RefWatcher.watch() 会以监控对象来创建一个 KeyedWeakReference 弱引用对象

* 在 AndroidWatchExecutor 的后台线程里，来检查弱引用已经被清除了，如果没被清除，则执行一次 GC

* 如果弱引用对象仍然没有被清除，说明内存泄漏了，系统就导出 hprof 文件，保存在 app 的文件系统目录下

* HeapAnalyzerService 启动一个单独的进程，使用 HeapAnalyzer 来分析 hprof 文件。它使用另外一个开源库 HAHA。

* HeapAnalyzer 通过查找 KeyedWeakReference 弱引用对象来查找内在泄漏

* HeapAnalyzer 计算 KeyedWeakReference 所引用对象的最短强引用路径，来分析内存泄漏，并且构建出对象引用链出来。

* 内存泄漏信息送回给 DisplayLeakService，它是运行在 app 进程里的一个服务。然后在设备通知栏显示内存泄漏信息。


###获取hprof文件

``` java 
	
	File heapDumpFile = new File("heapdump.hprof");
	Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
```
