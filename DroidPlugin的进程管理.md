　　这一篇通过分析代码试图搞清楚以下3个问题：
　　• 插件进程是如何被hook住的？
　　• 插件进程die是如何被检测到的？
　　• 插件进程是如何被管理的？
## 一、插件进程是如何被hook住的？ ##
　　在写宿主程序的时候，我们知道需要在Application的onCreate()和attachBaseContext()里调用PluginHelper的API来安装hook。但是，插件程序本身是不会调用这些API的，那么被启动的插件程序是如何被hook住的呢？
　　首先我们要再次回顾一下activity启动的一些细节，图比较大切成了两张，缩进表示该方法是在上一级方法里调用的子方法。先看左半边图：
![](http://i.imgur.com/mTSgY0F.png)
　　比较简单，宿主在一个新进程里启动插件的时候，AMS会向插件进程的ActivityThread发起两个调用：bindApplication()和scheduleLaunchAcitivity()。再看右半边图：
![](http://i.imgur.com/J4qJEPm.png)
　　这张图主要描述了这两个调用具体干了什么，实际上它们只是向ActivityThread的mH里发送了两个消息，真正干活的是mH（看过第二篇的可能有印象，我们把mH里面的mCallback替换成了我们的PluginCallback）。
　　看看bindApplication()具体做了哪些事情：

- 调用getPackageInfoNoCheck()创建一个LoadedApk对象，注意，由于AMS并不知道关于插件的事情，所以这里加载的还是宿主apk！所以实际上插件是启动不起来的，具体怎么处理的后面会介绍。创建的LoadedApk会放到一个mPackages的map中。
- 创建Instrumentation，调用LoadedApk.makeApplication()创建Application，注意只是创建，并没有调用Application的onCreate()。创建Application会放到一个mAllApplications的list中。

　　再看看scheduleLaunchActivity()具体做了哪些事情：

- 通过Instrumentation加载、创建activity对象，这里会用到class loader。
- 从mPackages中取出之前创建的LoadedApk，再次调用它的makeApplication()方法。这次调用和上次不同，由于Application已经创建过了所以会直接拿出来用，另外传入的第二个参数instrumentation不为空，因此会调用Application的onCreate()方法。
 
　　说了这么多，都只是Android默认的运行流程。那么DroidPlugin是如何让插件被加载和启动的呢？先上一张图描述一下概况，以免后面分析代码的时候会晕。我们和上一张图对比一下看主要区别在哪里：

其实说穿了也很简单，我们不是在PluginCallback里hook了handleLaunchActivity()方法吗？那就在这个hook里手动加载一下插件apk，创建LoadedApk对象并调用其makeApplication()方法，创建Application并调用其onCreate()。在这一切都做完以后，继续往下执行，调用宿主Application的onCreate()。
也就是说，其实创建了两个Application对象，先调用插件Application的onCreate()，再调用宿主Application的onCreate()。这样就回答了文章开头提出的第一个问题了：在宿主Application的onCreate()里，我们是安装了所有hook的，这样插件apk就也被hook住啦~~
思路已经清楚了，下面分析代码，重点看一下PluginProcessManager的2个关键API：preLoadApk()和preMakeApplication()。现在明白为什么这两个方法要带“pre”前缀了，因为新创建的Application确实是被先调用的呀。
preLoadApk()大家可能还有印象，是在PluginCallback里曾经露过脸，看一下具体代码：
[java] view plain copy 
public static void preLoadApk(Context hostContext, ComponentInfo pluginInfo) throws IOException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, PackageManager.NameNotFoundException, ClassNotFoundException {  
    boolean found = false;  
    synchronized (sPluginLoadedApkCache) {  
        Object object = ActivityThreadCompat.currentActivityThread();  
        if (object != null) {  
            Object mPackagesObj = FieldUtils.readField(object, "mPackages");  
            Object containsKeyObj = MethodUtils.invokeMethod(mPackagesObj, "containsKey", pluginInfo.packageName)  
            if (containsKeyObj instanceof Boolean && !(Boolean) containsKeyObj) {  
                final Object loadedApk;  
                ... ...  
                loadedApk = MethodUtils.invokeMethod(object, "getPackageInfoNoCheck", pluginInfo.applicationInfo, CompatibilityInfoCompat.DEFAULT_COMPATIBILITY_INFO());  
    //-------------------------------------------------------------------------------------  
        String optimizedDirectory = PluginDirHelper.getPluginDalvikCacheDir(hostContext, pluginInfo.packageName);  
        String libraryPath = PluginDirHelper.getPluginNativeLibraryDir(hostContext, pluginInfo.packageName);  
        String apk = pluginInfo.applicationInfo.publicSourceDir;  
        ... ...  
        classloader = new PluginClassLoader(apk, optimizedDirectory, libraryPath, ClassLoader.getSystemClassLoader());  
        synchronized (loadedApk) {  
            FieldUtils.writeDeclaredField(loadedApk, "mClassLoader", classloader);  
        }  
        ... ...  
        found = true;  
    //-------------------------------------------------------------------------------------  
    if (found) {  
        PluginProcessManager.preMakeApplication(hostContext, pluginInfo);  
    }  
}  
根据代码的功能大致分为3段：
1. 判断ActivityThread的mPackages字段是否包含插件包，如果不包含，则调用getPackageInfoNoCheck()加载apk，获取LoadedApk对象。
mPackages是ActivityThread里的一个map：key是包名，value是对应的LoadedApk对象。getPackageInfoNoCheck()会创建一个新的LoadedApk对象，里面包含了apk的所有信息，有了这些信息以后就可以启动插件程序了。这个对象会被放进mPackages中待日后使用。
2. 替换掉LoadedApk对象的mClassLoader字段
LoadedApk的class loader最终会被传给Instrumentation，用来加载插件apk中的类。默认的class loader是一个PathClassLoader，这里替换成了PluginClassLoader，目的和之前一样是为了解决奇酷手机support V4库加载的问题。
3. 调用preMakeApplication()，下面节选了preMakeApplication()的代码：
[java] view plain copy 
private static void preMakeApplication(Context hostContext, ComponentInfo pluginInfo) {  
    try {  
        final Object loadedApk = sPluginLoadedApkCache.get(pluginInfo.packageName);  
        if (loadedApk != null) {  
            Object mApplication = FieldUtils.readField(loadedApk, "mApplication");  
            if (mApplication != null) {  
                return;  
            }  
        ... ...  
        MethodUtils.invokeMethod(loadedApk, "makeApplication", false, ActivityThreadCompat.getInstrumentation());  
        ... ...  
}  
首先判断LoadedApk的mApplication字段是否为空，这段感觉有点多余，因为makeApplication()方法的开头也会先判断一下的。然后就是调用makeApplication()方法啦，这样插件apk的Application就被创建出来了，onCreate()也会被执行。 
二、插件进程die是如何被检测到的？
上面分析过了，插件进程启动的时候，也会创建宿主Application并调用其onCreate()，因此会调用到PluginHelper的applicationOnCreate()方法。
[java] view plain copy 
public void applicationOnCreate(final Context baseContext) {  
    mContext = baseContext;  
    initPlugin(baseContext);  
}  
在initPlugin()里执行了下面两个步骤：
• 添加一个ServiceConnection（PluginHelper实现了ServiceConnection接口）
• 调用PluginManager的init()方法去连接PluginManagerService
[java] view plain copy 
private void initPlugin(Context baseContext) {  
        ... ...  
    PluginManager.getInstance().addServiceConnection(PluginHelper.this);  
    PluginManager.getInstance().init(baseContext);  
        ... ...  
}  
[java] view plain copy
// PluginManager.java  
public void init(Context hostContext) {  
    mHostContext = hostContext;  
    connectToService();  
}  
看一下connectToService()：
[java] view plain copy 
public void connectToService() {  
    if (mPluginManager == null) {  
        try {  
            Intent intent = new Intent(mHostContext, PluginManagerService.class);  
            intent.setPackage(mHostContext.getPackageName());  
            mHostContext.startService(intent);  
            mHostContext.bindService(intent, this, Context.BIND_AUTO_CREATE);  
        } catch (Exception e) {  
            Log.e(TAG, "connectToService", e);  
        }  
    }  
}  
首先startService()，然后再bindService()，这样即使解绑了，服务还是可以继续保持运行。连接上服务以后，会调用PluginManager的onServiceConnected()，这一步比较关键：
[java] view plain copy 
public void onServiceConnected(final ComponentName componentName, final IBinder iBinder) {  
    ... ...  
    mPluginManager.waitForReady();  
    mPluginManager.registerApplicationCallback(new IApplicationCallback.Stub() {  
        @Override  
        public Bundle onCallback(Bundle extra) throws RemoteException {  
            return extra;  
        }  
    });  
    ... ...  
}  
看到没，这里注册了一个ApplicationCallback，这是一个自定义的AIDL远程调用接口，会调用到远端的IPluginManagerImpl，进而调用进BaseActivityManagerService：
[java] view plain copy
public boolean registerApplicationCallback(int callingPid, int callingUid, IApplicationCallback callback) {  
    return mRemoteCallbackList.register(callback, new ProcessCookie(callingPid, callingUid));  
}  
注意，这可不是一个普通的list哦，这是一个RemoteCallbackList，在binder对端死掉的时候，会收到一个通知，这样就能知道插件进程是死是活了，具体的处理放到onProcessDied()里去实现。看一下这个类的实现：
[java] view plain copy
private class MyRemoteCallbackList extends RemoteCallbackList<IApplicationCallback> {  
    @Override  
    public void onCallbackDied(IApplicationCallback callback, Object cookie) {  
        super.onCallbackDied(callback, cookie);  
        if (cookie != null && cookie instanceof ProcessCookie) {  
            ProcessCookie p = (ProcessCookie) cookie;  
            onProcessDied(p.pid, p.uid);  
        }  
    }  
}  
最后我们看一下RemoteCallbackList的register()方法，在注册callback的时候会调用binder的linkToDeath，这样当对端死掉的时候就能收到通知啦，就是这么简单：
[java] view plain copy 
public boolean register(E callback, Object cookie) {  
    synchronized (mCallbacks) {  
        if (mKilled) {  
            return false;  
        }  
        IBinder binder = callback.asBinder();  
        try {  
            Callback cb = new Callback(callback, cookie);  
            binder.linkToDeath(cb, 0);  
            mCallbacks.put(binder, cb);  
            return true;  
        } catch (RemoteException e) {  
            return false;  
        }  
    }  
}  
至此，插件进程die如何被检测到的问题就搞清楚了，如下图：

三、插件进程是如何被管理的？
目前DroidPlugin的进程管理还是比较粗糙的，没有考虑task affinity，策略也比较简单粗暴。
主要逻辑实现在MyActivityManagerService里，代码就不贴了比较简单，文字总结一下。
1. MyActivityManagerService里维护了两个进程列表：
    • 一个叫StaticProcessList，包含了AndroidManifest.xml里注册的所有进程
    • 一个叫RunningProcessList，包含了所有已经在运行的进程
2. 每次要启动插件时，首先在RunningProcessList里查找看是否有符合条件的进程：
    • 如果该进程加载过这个包，并且进程名与插件包一致，返回直接使用
    • 否则，遍历该进程加载的所有包，如果与插件包签名一致，返回直接使用
    • 使用这种方式，相同签名的多个插件包会运行在同一个进程中
3. 如果RunningProcessList里找不到，就到StaticProcessList里去查找看有没有符合条件的进程：
    • 如果该进程在运行，但是是个空进程，也就是没有启动任何插件包，返回直接使用
    • 如果该进程在运行但进程名为空，且该进程加载过这个包或者与插件包签名一致，返回直接使用
    • 如果该进程没有运行，返回使用之
找到合适的进程以后，还要选出未被占用stub组件（activity/service/provider)，然后把该组件的targetProcessName设置为目标进程。
 
另外还有个问题：如果进程不够用了怎么办？需要设计一个进程回收策略。
每次selectStubXXX()、activity或者service调用onDestroy()、以及进程die的时候，都会调用runProcessGC()回收进程资源（好像少了个判断？进程数量超过一个阈值的时候才需要回收吧）。如果是插件进程（非宿主进程），且不是持久进程：
    • 按进程优先级排序，>=IMPORTANCE_SERVICE优先级的杀掉（数字越低优先级越高）
    • 没有任何activity、service、provider的空进程，杀掉
    • 没有activity，只有service的进程，获取该进程的所有服务，调用stopSelf()停掉服务