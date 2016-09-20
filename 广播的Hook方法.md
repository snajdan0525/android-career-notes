　　动态注册的大致流程如下图，后续将逐个分步讲解
![](http://i.imgur.com/ga6VQQs.jpg)
　　不论是静态广播还是动态广播，在使用之前都是需要注册的；动态广播的注册需要借助Context类的registerReceiver方法，而静态广播的注册直接在AndroidManifest.xml中声明即可；我们首先分析一下动态广播的注册过程。
1.首先我们先看一下注册方法
```java
IntentFilter intentFilter = new IntentFilter("com.chan.plugin.receiver");
registerReceiver(new PluginReceiver(), intentFilter);
```
我们先到registerReceiver中去查看：
```java
@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter) {
    return mBase.registerReceiver(receiver, filter);
}
```
　　看到这里的mBase，应该会想到——关于Context的startActivity或者registerReceiver等method其实都是在ContextImpl中真正实现的。我们查看ContextImpl的代码，经过代码跟踪，我发现真正的逻辑实现是在registerReceiverInternal函数中

```java
private Intent registerReceiverInternal(BroadcastReceiver receiver,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context) {
    IIntentReceiver rd = null;
    if (receiver != null) {
		//mPackageInfo是一个LoadedApk实例，它是用来负责处理广播的接收的，
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();//获取主线程的Handler
            }
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        } else {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }
	//获得一个IIntentReceiver接口对象rd，这是一个Binder对象，接下来会把它传给ActivityManagerService，ActivityManagerService在收到相应的广播时，就是通过这个Binder对象来通知MainActivity来接收的。
    try {
        return ActivityManagerNative.getDefault().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName,
                rd, filter, broadcastPermission);//调用了ActivityManangerService中的registerReceiver类
	//ActivityManangerService是识别Activity和Receiver的很重要的类
    } catch (RemoteException e) {
        return null;
    }
}
```
>　　System private API for dispatching intent broadcasts. This is given to the activity manager as part of registering for an intent broadcasts, and is called when it receives intents.
　　这个类是通过AIDL工具生成的，它是一个Binder对象，因此可以用来跨进程传输；文档说的很清楚，它是用来进行广播分发的。

　　由于广播的分发过程是在AMS中进行的，而AMS所在的进程和BroadcastReceiver所在的进程不一样，因此要把广播分发到BroadcastReceiver具体的进程需要进行跨进程通信，这个通信的载体就是IIntentReceiver类。其实这个类的作用跟 Activity生命周期管理 中提到的 IApplicationThread相同，都是App进程给AMS进程用来进行通信的对象。另外，IIntentReceiver是一个接口，从上述代码中可以看出，它的实现类为LoadedApk.ReceiverDispatcher。

接下来进入到ActivityManagerNative的registerReceiver函数中

```java
public Intent registerReceiver(IApplicationThread caller, String packageName,
        IIntentReceiver receiver,
        IntentFilter filter, String perm) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    data.writeString(packageName);
    data.writeStrongBinder(receiver != null ? receiver.asBinder() : null);
    filter.writeToParcel(data, 0);
    data.writeString(perm);
    mRemote.transact(REGISTER_RECEIVER_TRANSACTION, data, reply, 0);
    reply.readException();
    Intent intent = null;
    int haveIntent = reply.readInt();
    if (haveIntent != 0) {
        intent = Intent.CREATOR.createFromParcel(reply);
    }
    reply.recycle();
    data.recycle();
    return intent;
}
```
这个函数通过Binder驱动程序就进入到ActivityManagerService中的registerReceiver函数中去了。下面我们就来看最重要的AMS中的registerReceiver函数：
```java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission) {
    synchronized(this) {
        ProcessRecord callerApp = null;
        if (caller != null) {
			//app进程记录块
            callerApp = getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when registering receiver " + receiver);
            }
            if (callerApp.info.uid != Process.SYSTEM_UID &&
                    !callerApp.pkgList.contains(callerPackage)) {
                throw new SecurityException("Given caller package " + callerPackage
                        + " is not running in process " + callerApp);
            }
        } else {
            callerPackage = null;
        }

        List allSticky = null;

        // Look for any matching sticky broadcasts...
        Iterator actions = filter.actionsIterator();
        if (actions != null) {
            while (actions.hasNext()) {
                String action = (String)actions.next();
                allSticky = getStickiesLocked(action, filter, allSticky);
            }
        } else {
            allSticky = getStickiesLocked(null, filter, allSticky);
        }

        // The first sticky in the list is returned directly back to
        // the client.
        Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;

        if (DEBUG_BROADCAST) Slog.v(TAG, "Register receiver " + filter
                + ": " + sticky);

        if (receiver == null) {
            return sticky;
        }

        ReceiverList rl
            = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
        if (rl == null) {
            rl = new ReceiverList(this, callerApp,
                    Binder.getCallingPid(),
                    Binder.getCallingUid(), receiver);
            if (rl.app != null) {
                rl.app.receivers.add(rl);
            } else {
                try {
                    receiver.asBinder().linkToDeath(rl, 0);
                } catch (RemoteException e) {
                    return sticky;
                }
                rl.linkedToDeath = true;
            }
            mRegisteredReceivers.put(receiver.asBinder(), rl);
        }
        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage, permission);
        rl.add(bf);
        if (!bf.debugCheck()) {
            Slog.w(TAG, "==> For Dynamic broadast");
        }
        mReceiverResolver.addFilter(bf);

        // Enqueue broadcasts for all existing stickies that match
        // this filter.
        if (allSticky != null) {
            ArrayList receivers = new ArrayList();
            receivers.add(bf);

            int N = allSticky.size();
            for (int i=0; i<N; i++) {
                Intent intent = (Intent)allSticky.get(i);
                BroadcastRecord r = new BroadcastRecord(intent, null,
                        null, -1, -1, null, receivers, null, 0, null, null,
                        false, true, true);
                if (mParallelBroadcasts.size() == 0) {
                    scheduleBroadcastsLocked();
                }
                mParallelBroadcasts.add(r);
            }
        }

        return sticky;
    }
}
```
至此动态注册就完毕了


接下来我们看一下静态注册的方法
