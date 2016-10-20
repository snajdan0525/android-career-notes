
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
			//mMainThread是ActivityThread类型，代表了应用程序的主线程
			//getApplication获取到ApplicationThread成员变量，这是一个binder对象
			//ActivityManagerService会使用它来和ActivityThread来进行进程间通信。
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

参数caller为ApplicationThread类型的Binder实体；参
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

```java
final int startActivityUncheckedLocked(ActivityRecord r,
        ActivityRecord sourceRecord, Uri[] grantedUriPermissions,
        int grantedMode, boolean onlyIfNeeded, boolean doResume) {
    final Intent intent = r.intent;
    final int callingUid = r.launchedFromUid;
    
    int launchFlags = intent.getFlags();
    
    // We'll invoke onUserLeaving before onPause only if the launching
    // activity did not explicitly state that this is an automated launch.
    mUserLeaving = (launchFlags&Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;
    if (DEBUG_USER_LEAVING) Slog.v(TAG,
            "startActivity() => mUserLeaving=" + mUserLeaving);
    
    // If the caller has asked not to resume at this point, we make note
    // of this in the record so that we can skip it when trying to find
    // the top running activity.
    if (!doResume) {
        r.delayedResume = true;
    }
    
    ActivityRecord notTop = (launchFlags&Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP)
            != 0 ? r : null;

    // If the onlyIfNeeded flag is set, then we can do this if the activity
    // being launched is the same as the one making the call...  or, as
    // a special case, if we do not know the caller then we count the
    // current top activity as the caller.
    if (onlyIfNeeded) {
        ActivityRecord checkedCaller = sourceRecord;
        if (checkedCaller == null) {
            checkedCaller = topRunningNonDelayedActivityLocked(notTop);
        }
        if (!checkedCaller.realActivity.equals(r.realActivity)) {
            // Caller is not the same as launcher, so always needed.
            onlyIfNeeded = false;
        }
    }

    if (sourceRecord == null) {
        // This activity is not being started from another...  in this
        // case we -always- start a new task.
        if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            Slog.w(TAG, "startActivity called from non-Activity context; forcing Intent.FLAG_ACTIVITY_NEW_TASK for: "
                  + intent);
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        }
    } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
        // The original activity who is starting us is running as a single
        // instance...  this new activity it is starting must go on its
        // own task.
        launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
    } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
        // The activity being started is a single instance...  it always
        // gets launched into its own task.
        launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
    }

    if (r.resultTo != null && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
        // For whatever reason this activity is being launched into a new
        // task...  yet the caller has requested a result back.  Well, that
        // is pretty messed up, so instead immediately send back a cancel
        // and let the new task continue launched as normal without a
        // dependency on its originator.
        Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
        sendActivityResultLocked(-1,
                r.resultTo, r.resultWho, r.requestCode,
            Activity.RESULT_CANCELED, null);
        r.resultTo = null;
    }

    boolean addingToTask = false;
    TaskRecord reuseTask = null;
    if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
            (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
        // If bring to front is requested, and no result is requested, and
        // we can find a task that was started with this same
        // component, then instead of launching bring that one to the front.
        if (r.resultTo == null) {
            // See if there is a task to bring to the front.  If this is
            // a SINGLE_INSTANCE activity, there can be one and only one
            // instance of it in the history, and it is always in its own
            // unique task, so we do a special search.
			/*
			*这段代码的逻辑是查看一下，当前有没有Task可以用来执行这个Activity.
			*由于r.launchMode的值不为ActivityInfo.LAUNCH_SINGLE_INSTANCE,因
			*此,它通过findTaskLocked函数来查找存不存这样的Task,这里返回的结果是
			*null,即taskTop为null.因此，需要创建一个新的Task来启动这个
			*Activity.
			*/
            ActivityRecord taskTop = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE
                    ? findTaskLocked(intent, r.info)
                    : findActivityLocked(intent, r.info);
            if (taskTop != null) {
                if (taskTop.task.intent == null) {
                    // This task was started because of movement of
                    // the activity based on affinity...  now that we
                    // are actually launching it, we can assign the
                    // base intent.
                    taskTop.task.setIntent(intent, r.info);
                }
                // If the target task is not in the front, then we need
                // to bring it to the front...  except...  well, with
                // SINGLE_TASK_LAUNCH it's not entirely clear.  We'd like
                // to have the same behavior as if a new instance was
                // being started, which means not bringing it to the front
                // if the caller is not itself in the front.
                ActivityRecord curTop = topRunningNonDelayedActivityLocked(notTop);
                if (curTop != null && curTop.task != taskTop.task) {
                    r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                    boolean callerAtFront = sourceRecord == null
                            || curTop.task == sourceRecord.task;
                    if (callerAtFront) {
                        // We really do want to push this one into the
                        // user's face, right now.
                        moveHomeToFrontFromLaunchLocked(launchFlags);
                        moveTaskToFrontLocked(taskTop.task, r);
                    }
                }
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    taskTop = resetTaskIfNeededLocked(taskTop, r);
                }
                if (onlyIfNeeded) {
                    // We don't need to start a new activity, and
                    // the client said not to do anything if that
                    // is the case, so this is it!  And for paranoia, make
                    // sure we have correctly resumed the top activity.
                    if (doResume) {
                        resumeTopActivityLocked(null);
                    }
                    return START_RETURN_INTENT_TO_CALLER;
                }
                if ((launchFlags &
                        (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK))
                        == (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) {
                    // The caller has requested to completely replace any
                    // existing task with its new activity.  Well that should
                    // not be too hard...
                    reuseTask = taskTop.task;
                    performClearTaskLocked(taskTop.task.taskId);
                    reuseTask.setIntent(r.intent, r.info);
                } else if ((launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0
                        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                    // In this situation we want to remove all activities
                    // from the task up to the one being started.  In most
                    // cases this means we are resetting the task to its
                    // initial state.
                    ActivityRecord top = performClearTaskLocked(
                            taskTop.task.taskId, r, launchFlags);
                    if (top != null) {
                        if (top.frontOfTask) {
                            // Activity aliases may mean we use different
                            // intents for the top activity, so make sure
                            // the task now has the identity of the new
                            // intent.
                            top.task.setIntent(r.intent, r.info);
                        }
                        logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                        top.deliverNewIntentLocked(callingUid, r.intent);
                    } else {
                        // A special case: we need to
                        // start the activity because it is not currently
                        // running, and the caller has asked to clear the
                        // current task to have this activity at the top.
                        addingToTask = true;
                        // Now pretend like this activity is being started
                        // by the top of its task, so it is put in the
                        // right place.
                        sourceRecord = taskTop;
                    }
                } else if (r.realActivity.equals(taskTop.task.realActivity)) {
                    // In this case the top activity on the task is the
                    // same as the one being launched, so we take that
                    // as a request to bring the task to the foreground.
                    // If the top activity in the task is the root
                    // activity, deliver this new intent to it if it
                    // desires.
                    if ((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                            && taskTop.realActivity.equals(r.realActivity)) {
                        logStartActivity(EventLogTags.AM_NEW_INTENT, r, taskTop.task);
                        if (taskTop.frontOfTask) {
                            taskTop.task.setIntent(r.intent, r.info);
                        }
                        taskTop.deliverNewIntentLocked(callingUid, r.intent);
                    } else if (!r.intent.filterEquals(taskTop.task.intent)) {
                        // In this case we are launching the root activity
                        // of the task, but with a different intent.  We
                        // should start a new instance on top.
                        addingToTask = true;
                        sourceRecord = taskTop;
                    }
                } else if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
                    // In this case an activity is being launched in to an
                    // existing task, without resetting that task.  This
                    // is typically the situation of launching an activity
                    // from a notification or shortcut.  We want to place
                    // the new activity on top of the current task.
                    addingToTask = true;
                    sourceRecord = taskTop;
                } else if (!taskTop.task.rootWasReset) {
                    // In this case we are launching in to an existing task
                    // that has not yet been started from its front door.
                    // The current task has been brought to the front.
                    // Ideally, we'd probably like to place this new task
                    // at the bottom of its stack, but that's a little hard
                    // to do with the current organization of the code so
                    // for now we'll just drop it.
                    taskTop.task.setIntent(r.intent, r.info);
                }
                if (!addingToTask && reuseTask == null) {
                    // We didn't do anything...  but it was needed (a.k.a., client
                    // don't use that intent!)  And for paranoia, make
                    // sure we have correctly resumed the top activity.
                    if (doResume) {
                        resumeTopActivityLocked(null);
                    }
                    return START_TASK_TO_FRONT;
                }
            }
        }
    }

    //String uri = r.intent.toURI();
    //Intent intent2 = new Intent(uri);
    //Slog.i(TAG, "Given intent: " + r.intent);
    //Slog.i(TAG, "URI is: " + uri);
    //Slog.i(TAG, "To intent: " + intent2);

    if (r.packageName != null) {
        // If the activity being launched is the same as the one currently
        // at the top, then we need to check if it should only be launched
        // once.
        ActivityRecord top = topRunningNonDelayedActivityLocked(notTop);
        if (top != null && r.resultTo == null) {
            if (top.realActivity.equals(r.realActivity)) {
                if (top.app != null && top.app.thread != null) {
                    if ((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP
                        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
                        logStartActivity(EventLogTags.AM_NEW_INTENT, top, top.task);
                        // For paranoia, make sure we have correctly
                        // resumed the top activity.
                        if (doResume) {
                            resumeTopActivityLocked(null);
                        }
                        if (onlyIfNeeded) {
                            // We don't need to start a new activity, and
                            // the client said not to do anything if that
                            // is the case, so this is it!
                            return START_RETURN_INTENT_TO_CALLER;
                        }
                        top.deliverNewIntentLocked(callingUid, r.intent);
                        return START_DELIVERED_TO_TOP;
                    }
                }
            }
        }

    } else {
        if (r.resultTo != null) {
            sendActivityResultLocked(-1,
                    r.resultTo, r.resultWho, r.requestCode,
                Activity.RESULT_CANCELED, null);
        }
        return START_CLASS_NOT_FOUND;
    }

    boolean newTask = false;
    boolean keepCurTransition = false;

    // Should this be considered a new task?
    if (r.resultTo == null && !addingToTask
            && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
        if (reuseTask == null) {
            // todo: should do better management of integers.
            mService.mCurTask++;
            if (mService.mCurTask <= 0) {
                mService.mCurTask = 1;
            }
            r.setTask(new TaskRecord(mService.mCurTask, r.info, intent), null, true);
            if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                    + " in new task " + r.task);
        } else {
            r.setTask(reuseTask, reuseTask, true);
        }
        newTask = true;
        moveHomeToFrontFromLaunchLocked(launchFlags);
        
    } else if (sourceRecord != null) {
        if (!addingToTask &&
                (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
            // In this case, we are adding the activity to an existing
            // task, but the caller has asked to clear that task if the
            // activity is already running.
            ActivityRecord top = performClearTaskLocked(
                    sourceRecord.task.taskId, r, launchFlags);
            keepCurTransition = true;
            if (top != null) {
                logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                top.deliverNewIntentLocked(callingUid, r.intent);
                // For paranoia, make sure we have correctly
                // resumed the top activity.
                if (doResume) {
                    resumeTopActivityLocked(null);
                }
                return START_DELIVERED_TO_TOP;
            }
        } else if (!addingToTask &&
                (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
            // In this case, we are launching an activity in our own task
            // that may already be running somewhere in the history, and
            // we want to shuffle it to the front of the stack if so.
            int where = findActivityInHistoryLocked(r, sourceRecord.task.taskId);
            if (where >= 0) {
                ActivityRecord top = moveActivityToFrontLocked(where);
                logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                top.deliverNewIntentLocked(callingUid, r.intent);
                if (doResume) {
                    resumeTopActivityLocked(null);
                }
                return START_DELIVERED_TO_TOP;
            }
        }
        // An existing activity is starting this new activity, so we want
        // to keep the new one in the same task as the one that is starting
        // it.
        r.setTask(sourceRecord.task, sourceRecord.thumbHolder, false);
        if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                + " in existing task " + r.task);

    } else {
        // This not being started from an existing activity, and not part
        // of a new task...  just put it in the top task, though these days
        // this case should never happen.
        final int N = mHistory.size();
        ActivityRecord prev =
            N > 0 ? mHistory.get(N-1) : null;
        r.setTask(prev != null
                ? prev.task
                : new TaskRecord(mService.mCurTask, r.info, intent), null, true);
        if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                + " in new guessed " + r.task);
    }

    if (grantedUriPermissions != null && callingUid > 0) {
        for (int i=0; i<grantedUriPermissions.length; i++) {
            mService.grantUriPermissionLocked(callingUid, r.packageName,
                    grantedUriPermissions[i], grantedMode, r.getUriPermissionsLocked());
        }
    }

    mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
            intent, r.getUriPermissionsLocked());

    if (newTask) {
        EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, r.task.taskId);
    }
    logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
    startActivityLocked(r, newTask, doResume, keepCurTransition);
    return START_SUCCESS;
}
```


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