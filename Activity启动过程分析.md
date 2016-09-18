
![](http://i.imgur.com/qtz6WkO.jpg)

调用Android.app.Activity里的startActivity函数

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```

接着经过两次函数调用最终调用了

```java
startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options)
```

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    if (mParent == null) {
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);//这里调用了mInstrumentation的execStartActivity
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        final View decor = mWindow != null ? mWindow.peekDecorView() : null;
        if (decor != null) {
            decor.cancelPendingInputEvents();
        }
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            // Note we want to go through this method for compatibility with
            // existing applications that may have overridden it.
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
    if (options != null && !isTopOfTask()) {
        mActivityTransitionState.startExitOutTransition(this, options);
    }
}
```

这里调用了mInstrumentation的execStartActivity，这是一个hide类型的函数,真正的打开Activity也是在这个函数中实现,需要注意的是，在IApplicationThread接口中有几个很重要的接口函数：
**scheduleLaunchActivity
scheduleSendResult
scheduleResumeActivity
schedulePauseActivity
scheduleStopActivity**等等，用来对Activity的状态进行调度，IApplicationThread的实现类就是大名鼎鼎的ActivityThread内部类**ApplicationThread**。

```java
 	 * @param who The Context from which the activity is being started.
     * @param contextThread The main thread of the Context from which the     activity
     * is being started.
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    if (mActivityMonitors != null) {
        synchronized (mSync) {
			//先查找一遍看是否存在这个activity
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
		//真正打开activity的地方，核心功能在whoThread中完成。
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
    }
    return null;
}
```
```java
 int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName()
```

　　getDefault返回的是ActivityManagerProxy对象的引用是一个Binder对象，他能够使用ActivityManagerService服务，现在我们切入到AMS的startActivity代码中：
ActivityManagerNative实际上就是ActivityManagerService这个远程对象的Binder代理对象；每次需要与AMS打交道的时候，需要借助这个代理对象通过驱动进而完成IPC调用
```java
public final int startActivity(IApplicationThread caller,
        Intent intent, String resolvedType, Uri[] grantedUriPermissions,
        int grantedMode, IBinder resultTo,
        String resultWho, int requestCode, boolean onlyIfNeeded, boolean debug,
        String profileFile, ParcelFileDescriptor profileFd, boolean autoStopProfiler) {
    return mMainStack.startActivityMayWait(caller, -1, intent, resolvedType,
            grantedUriPermissions, grantedMode, resultTo, resultWho,
            requestCode, onlyIfNeeded, debug, profileFile, profileFd, autoStopProfiler,
            null, null);
}
```
　　上面代码中的mMainStack是一个ActivityStack对象，AMS调用ActivityStack的一系列方法来准备要启动的Activity的相关信息。我们平时说的什么任务栈啊都在这个类中有涉及。
　　接着看ActivityStack的startActivityMayWait：

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        Intent intent, String resolvedType, Uri[] grantedUriPermissions,
        int grantedMode, IBinder resultTo,
        String resultWho, int requestCode, boolean onlyIfNeeded,
        boolean debug, String profileFile, ParcelFileDescriptor profileFd,
        boolean autoStopProfiler, WaitResult outResult, Configuration config) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

	//通过Intent获取用来启动Activity时所需的component对象
    boolean componentSpecified = intent.getComponent() != null;

    // Don't modify the client's object!
    intent = new Intent(intent);

    //Collect information about the target of the Intent.
    //解析当前要启动的Activity信息,
	//跟入代码后resolveActivity还调用了resolveIntent，queryIntentActivities等函数，这些都是很显而易见的函数功能，显然是根据intent中提供的信息，检索出最匹配的结果
	ActivityInfo aInfo = resolveActivity(intent, resolvedType, debug,
            profileFile, profileFd, autoStopProfiler);
        ......
		......
		......
        
        int res = startActivityLocked(caller, intent, resolvedType,
                grantedUriPermissions, grantedMode, aInfo,
                resultTo, resultWho, requestCode, callingPid, callingUid,
                onlyIfNeeded, componentSpecified, null);
        
        if (mConfigWillChange && mMainStack) {
            // If the caller also wants to switch to a new configuration,
            // do so now.  This allows a clean switch, as we are waiting
            // for the current activity to pause (so we will not destroy
            // it), and have not yet started the next activity.
            mService.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                    "updateConfiguration()");
            mConfigWillChange = false;
            if (DEBUG_CONFIGURATION) Slog.v(TAG,
                    "Updating to new configuration after starting activity.");
            mService.updateConfigurationLocked(config, null, false, false);
        }
        
        Binder.restoreCallingIdentity(origId);
        
        if (outResult != null) {
            outResult.result = res;
            if (res == IActivityManager.START_SUCCESS) {
                mWaitingActivityLaunched.add(outResult);
                do {
                    try {
                        mService.wait();
                    } catch (InterruptedException e) {
                    }
                } while (!outResult.timeout && outResult.who == null);
            } else if (res == IActivityManager.START_TASK_TO_FRONT) {
                ActivityRecord r = this.topRunningActivityLocked(null);
                if (r.nowVisible) {
                    outResult.timeout = false;
                    outResult.who = new ComponentName(r.info.packageName, r.info.name);
                    outResult.totalTime = 0;
                    outResult.thisTime = 0;
                } else {
                    outResult.thisTime = SystemClock.uptimeMillis();
                    mWaitingActivityVisible.add(outResult);
                    do {
                        try {
                            mService.wait();
                        } catch (InterruptedException e) {
                        }
                    } while (!outResult.timeout && outResult.who == null);
                }
            }
        }
        
        return res;
    }
}
    
```
之后进入到startActivityLocked函数中：

```java
final int startActivityLocked(IApplicationThread caller,
        Intent intent, String resolvedType,
        Uri[] grantedUriPermissions,
        int grantedMode, ActivityInfo aInfo, IBinder resultTo,
        String resultWho, int requestCode,
        int callingPid, int callingUid, boolean onlyIfNeeded,
        boolean componentSpecified, ActivityRecord[] outActivity) {

    int err = START_SUCCESS;

    ProcessRecord callerApp = null;
    if (caller != null) {
        callerApp = mService.getRecordForAppLocked(caller);
		//这个参数caller来自于execStartActivity函数中的whoThread,代表了调用者的进/
		//程，getRecordForAppLocked就是从LRU的进程记录表中查找到这个caller对应的
		//ProcessRecord
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else {
            Slog.w(TAG, "Unable to find app for caller " + caller
                  + " (pid=" + callingPid + ") when starting: "
                  + intent.toString());
            err = START_PERMISSION_DENIED;
        }
    }

    if (err == START_SUCCESS) {
        Slog.i(TAG, "START {" + intent.toShortString(true, true, true) + "} from pid "
                + (callerApp != null ? callerApp.pid : callingPid));
    }

    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    if (resultTo != null) {
        int index = indexOfTokenLocked(resultTo);
        if (DEBUG_RESULTS) Slog.v(
            TAG, "Will send result to " + resultTo + " (index " + index + ")");
        if (index >= 0) {
            sourceRecord = mHistory.get(index);
            if (requestCode >= 0 && !sourceRecord.finishing) {
                resultRecord = sourceRecord;
            }
        }
    }

    int launchFlags = intent.getFlags();

    if ((launchFlags&Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0
            && sourceRecord != null) {
        // Transfer the result target from the source activity to the new
        // one being started, including any failures.
        if (requestCode >= 0) {
            return START_FORWARD_AND_REQUEST_CONFLICT;
        }
        resultRecord = sourceRecord.resultTo;
        resultWho = sourceRecord.resultWho;
        requestCode = sourceRecord.requestCode;
        sourceRecord.resultTo = null;
        if (resultRecord != null) {
            resultRecord.removeResultsLocked(
                sourceRecord, resultWho, requestCode);
        }
    }

    if (err == START_SUCCESS && intent.getComponent() == null) {
        // We couldn't find a class that can handle the given Intent.
        // That's the end of that!
        err = START_INTENT_NOT_RESOLVED;
    }

    if (err == START_SUCCESS && aInfo == null) {
        // We couldn't find the specific class specified in the Intent.
        // Also the end of the line.
        err = START_CLASS_NOT_FOUND;
    }

    if (err != START_SUCCESS) {
        if (resultRecord != null) {
            sendActivityResultLocked(-1,
                resultRecord, resultWho, requestCode,
                Activity.RESULT_CANCELED, null);
        }
        mDismissKeyguardOnNextActivity = false;
        return err;
    }

    final int perm = mService.checkComponentPermission(aInfo.permission, callingPid,
            callingUid, aInfo.applicationInfo.uid, aInfo.exported);
    if (perm != PackageManager.PERMISSION_GRANTED) {
        if (resultRecord != null) {
            sendActivityResultLocked(-1,
                resultRecord, resultWho, requestCode,
                Activity.RESULT_CANCELED, null);
        }
        mDismissKeyguardOnNextActivity = false;
        String msg;
        if (!aInfo.exported) {
            msg = "Permission Denial: starting " + intent.toString()
                    + " from " + callerApp + " (pid=" + callingPid
                    + ", uid=" + callingUid + ")"
                    + " not exported from uid " + aInfo.applicationInfo.uid;
        } else {
            msg = "Permission Denial: starting " + intent.toString()
                    + " from " + callerApp + " (pid=" + callingPid
                    + ", uid=" + callingUid + ")"
                    + " requires " + aInfo.permission;
        }
        Slog.w(TAG, msg);
        throw new SecurityException(msg);
    }

    if (mMainStack) {
        if (mService.mController != null) {
            boolean abort = false;
            try {
                // The Intent we give to the watcher has the extra data
                // stripped off, since it can contain private information.
                Intent watchIntent = intent.cloneFilter();
                abort = !mService.mController.activityStarting(watchIntent,
                        aInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                mService.mController = null;
            }

            if (abort) {
                if (resultRecord != null) {
                    sendActivityResultLocked(-1,
                        resultRecord, resultWho, requestCode,
                        Activity.RESULT_CANCELED, null);
                }
                // We pretend to the caller that it was really started, but
                // they will just get a cancel result.
                mDismissKeyguardOnNextActivity = false;
                return START_SUCCESS;
            }
        }
    }
    
    ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
            intent, resolvedType, aInfo, mService.mConfiguration,
            resultRecord, resultWho, requestCode, componentSpecified);
    if (outActivity != null) {
        outActivity[0] = r;
    }

    if (mMainStack) {
        if (mResumedActivity == null
                || mResumedActivity.info.applicationInfo.uid != callingUid) {
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid, "Activity start")) {
                PendingActivityLaunch pal = new PendingActivityLaunch();
                pal.r = r;
                pal.sourceRecord = sourceRecord;
                pal.grantedUriPermissions = grantedUriPermissions;
                pal.grantedMode = grantedMode;
                pal.onlyIfNeeded = onlyIfNeeded;
                mService.mPendingActivityLaunches.add(pal);
                mDismissKeyguardOnNextActivity = false;
                return START_SWITCHES_CANCELED;
            }
        }
    
        if (mService.mDidAppSwitch) {
            // This is the second allowed switch since we stopped switches,
            // so now just generally allow switches.  Use case: user presses
            // home (switches disabled, switch to home, mDidAppSwitch now true);
            // user taps a home icon (coming from home so allowed, we hit here
            // and now allow anyone to switch again).
            mService.mAppSwitchesAllowedTime = 0;
        } else {
            mService.mDidAppSwitch = true;
        }
     
        mService.doPendingActivityLaunchesLocked(false);
    }
    
    err = startActivityUncheckedLocked(r, sourceRecord,
            grantedUriPermissions, grantedMode, onlyIfNeeded, true);
    if (mDismissKeyguardOnNextActivity && mPausingActivity == null) {
        // Someone asked to have the keyguard dismissed on the next
        // activity start, but we are not actually doing an activity
        // switch...  just dismiss the keyguard now, because we
        // probably want to see whatever is behind it.
        mDismissKeyguardOnNextActivity = false;
        mService.mWindowManager.dismissKeyguard();
    }
    return err;
}
```
　　通过上面的代码可以看出来全部都是用来解析ActivityInfo和Intent等启动信息的，
AMS如何知道要启动的activity是谁呢？就是通过intent，先解析Intent得到一些基本信息。然后根据这些结果生成Activityrecord，存放在活动栈里面。最终会生成一个ActivityRecord.在调用startActivityUncheckedLocked使用这个对象


　　注意两个类：ApplicationThreadNative和ActivityManagerNative。。。
**上面这段代码会调用ActivityThread内部类ApplicationThread里的scheduleLaunchActivity**


```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
        IVoiceInteractor voiceInteractor, int procState, Bundle state,
        PersistableBundle persistentState, List<ResultInfo> pendingResults,
        List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
        ProfilerInfo profilerInfo) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.voiceInteractor = voiceInteractor;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;
    r.persistentState = persistentState;

    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;

    r.startsNotResumed = notResumed;
    r.isForward = isForward;

    r.profilerInfo = profilerInfo;

    updatePendingConfiguration(curConfig);

    sendMessage(H.LAUNCH_ACTIVITY, r);
	/*
	H是一个Handler对象,经过一个简单的封装,
	H.LAUNCH_ACTIVITY就成了msg.what,而类型为ActivityClientRecord的r这个变量就成了
	msg.obj
	*/
}
```
进入sendMessage函数中：
```java
private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
    synchronized (this) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        mH.sendMessage(msg);//这个mH是一个hook点
    }
}
```
对应的消息处理函数handlerMessage如下：

```java
public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
			//处理LAUNCH_ACTIVITY消息
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null);//调用handleLaunchActivity
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            case RELAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                handleRelaunchActivity(r);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            case PAUSE_ACTIVITY:.......

```

　　ActivityThread中的performLaunchActivity，**此函数是被handleLaunchActivity调用的**(具体的跟进去就知道了)，而handleLaunchActivity是由scheduleLaunchActivity这个函数中通过Handler机制调用的一个消息处理函数。

　　ActivityClientRecord对象表示一个Acticity以及他的相关信息， 比如activityInfo字段包括了启动模式等， 还有loadedApk， 顾名思义指的是加载过了的APK， 他会被放在一个Map中， 应用包名到LoadedApk的键值对的映射， 包含了一个应用的相关信息。

```Java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
		//packageInfo是一个LoadedApk类型的对象，他从一个HashMap中取出来，这个HashMap维护了LoadedApk和包名的关系一个包名对应一个Loadedapk(表示这个apk已经加载过了)
    }


    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
		//将要启动的Activity所在的包名和目标Activity
    }

    Activity activity = null;
    try {
		//这个ClassLoader保存于LoadedApk对象中， 它是用来加载我们写的activity的加载器
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
		//用加载器来加载activity类， 这个会根据不同的intent加载匹配的activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
        if (localLOGV) Slog.v(
                TAG, r + ": app=" + app
                + ", appName=" + app.getPackageName()
                + ", pkg=" + r.packageInfo.getPackageName()
                + ", comp=" + r.intent.getComponent().toShortString()
                + ", dir=" + r.packageInfo.getAppDir());

        if (activity != null) {
			//Activity对应的Context对象就在这里
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            //这里Context和Activity绑定了起来
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.voiceInteractor);//此处调用Activity的attach函数 
			
            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
			//从这里就会执行到我们通常看到的activity的生命周期的onCreate里面
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                            r.persistentState);
                } else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPostCreate()");
                }
            }
        }
        r.paused = true;

        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component
                + ": " + e.toString(), e);
        }
    }

    return activity;
}
```



```java
//这个ClassLoader保存于LoadedApk对象中， 它是用来加载我们写的activity的加载器
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
```

```java
public ClassLoader getClassLoader() {
    synchronized (this) {
        if (mClassLoader != null) {
            return mClassLoader;
        }
			String zip = mAppDir;
            String instrumentationAppDir =
                    mActivityThread.mInstrumentationAppDir;
            String instrumentationAppPackage =
                    mActivityThread.mInstrumentationAppPackage;
            String instrumentedAppDir =
                    mActivityThread.mInstrumentedAppDir;
            String[] instrumentationLibs = null;
			......
            mClassLoader =
                ApplicationLoaders.getDefault().getClassLoader(
                    zip, mLibDir, mBaseClassLoader);
			/*zip为Apk路径,libraryPath是jni路径，此处无论如何都可以拿到一个
			ClassLoader并存在了mClassLoader当中，
			当下一次再ava.lang.ClassLoader cl = r.packageInfo.getClassLoader();
			获取LoadedApk的classLoader的时候就一定返回了mClassLoader*/
            initializeJavaContextClassLoader();
            StrictMode.setThreadPolicy(oldPolicy);
        } else {
            if (mBaseClassLoader == null) {
                mClassLoader = ClassLoader.getSystemClassLoader();
            } else {
                mClassLoader = mBaseClassLoader;
            }
        }
        return mClassLoader;
    }
}
```
　　ApplicationLoaders使用单例它的getClassLoader方法根据传入的zip路径，事实上也就是Apk的路径来创建加载器， 返回的是一个PathClassLoader。 并且PathClassLoader只能加载安装过的APK。 这个加载器创建的时候传入的是当前应用APK的路径， 理所应当的，想加载其他的APK就构造一个传递其他APK的类加载器。

现在我们切入354行，进入getClassLoader中看

```java
    public ClassLoader getClassLoader(String zip, String libPath, ClassLoader parent)
    {
        /*
         * This is the parent we use if they pass "null" in.  In theory
         * this should be the "system" class loader; in practice we
         * don't use that and can happily (and more efficiently) use the
         * bootstrap class loader.
         */
        ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();

        synchronized (mLoaders) {
            if (parent == null) {
                parent = baseParent;
            }

            /*
             * If we're one step up from the base class loader, find
             * something in our cache.  Otherwise, we create a whole
             * new ClassLoader for the zip archive.
             */
            if (parent == baseParent) {
                ClassLoader loader = mLoaders.get(zip);
                if (loader != null) {
                    return loader;
                }
    
                PathClassLoader pathClassloader =
                    new PathClassLoader(zip, libPath, parent);
                
                mLoaders.put(zip, pathClassloader);
                return pathClassloader;
            }

            return new PathClassLoader(zip, parent);
        }
    }

```

到此，完事具备了，开始调用如下代码：

```java
activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
```

通过使用上面我们拿到的classLoader加载我们要启动的Activity,并反射创建一个Activity的实例
```java
    /**
     * Perform instantiation of the process's {@link Activity} object.  The
     * default implementation provides the normal system behavior.
     * 
     * @param cl The ClassLoader with which to instantiate the object.
     * @param className The name of the class implementing the Activity
     *                  object.
     * @param intent The Intent object that specified the activity class being
     *               instantiated.
     * 
     * @return The newly instantiated Activity object.
     */
    public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```


Activity的attach是一个隐藏的函数，这个函数是由ActivityThread中的performLaunchActivity调用

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);

    mFragments.attachActivity(this, mContainer, null);

    mWindow = PolicyManager.makeNewWindow(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    if (voiceInteractor != null) {
        if (lastNonConfigurationInstances != null) {
            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
        } else {
            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                    Looper.myLooper());
        }
    }
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
}
```

```java
activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.voiceInteractor);
```

切到这段代码中的getInstrumentation()可以看到

```java
public Instrumentation getInstrumentation()
{
    return mInstrumentation;
}
```

**可以看到mInstrumentation是ActivityThread的一个成员变量，不过，既然找到ActvityThread了，所有的问题都解决了，为什么，因为每个应用在运行时都只有一个ActivityThread，它用来代表主线程，它是单例的.**
　　


　　这个ApplicationThread实际上是一个Binder对象，是App所在的进程与AMS所在进程system_server通信的桥梁；在Activity启动的过程中，App进程会频繁地与AMS进程进行通信：
App进程会委托AMS进程完成Activity生命周期的管理以及任务栈的管理；这个通信过程AMS是Server端，App进程通过持有AMS的client代理ActivityManagerNative完成通信过程；
AMS进程完成生命周期管理以及任务栈管理后，会把控制权交给App进程，让App进程完成Activity类对象的创建，以及生命周期回调；这个通信过程也是通过Binder完成的，App所在server端的Binder对象存在于ActivityThread的内部类ApplicationThread；AMS所在client通过持有IApplicationThread的代理对象完成对于App进程的通信。