```java
@Override
public FragmentTransaction beginTransaction() {
    return new BackStackRecord(this);//beginTransaction实则返回的是个BackStackRecord(FragmentTransaction的子类),
}
这里的this是一个Fragment引用
```
　　FragmentTransaction是个啥：FragmentTransaction实际上是一个抽象类，里面定义了一些关于Fragment操作的函数接口。
```java
public abstract class FragmentTransaction {
    add
    replace
    remove
    hide 
    show
    detach
    attach
    addToBackStack
    ...
}
```
　　BackStackRecord是个啥：BackStackRecord是FragmentTransaction的子类，实现了对Fragment操作的接口并且还有一些自己独特的东西,而且他还实现了Runnable接口。
```java
static final int OP_NULL = 0;
static final int OP_ADD = 1;
static final int OP_REPLACE = 2;
static final int OP_REMOVE = 3;
static final int OP_HIDE = 4;
static final int OP_SHOW = 5;
static final int OP_DETACH = 6;
static final int OP_ATTACH = 7;

static final class Op {
    Op next;
    Op prev;
    int cmd;
    Fragment fragment;
    int enterAnim;
    int exitAnim;
    int popEnterAnim;
    int popExitAnim;
    ArrayList<Fragment> removed;
}
```
　　从next， prev 两个指针可以看出，Op是作为一个链表的节点而存在的，因此BackStackRecord肯定是在一个链表中存储了一次事务中的所有需要执行的操作,而且是以双向链表的形式来存储这些操作序列的。
```java
//addOp执行了一个典型的添加节点到链表末尾的数据结构操作
void addOp(Op op) {
    if (mHead == null) {
        mHead = mTail = op;
    } else {
        op.prev = mTail;
        mTail.next = op;
        mTail = op;
    }
    op.enterAnim = mEnterAnim;
    op.exitAnim = mExitAnim;
    op.popEnterAnim = mPopEnterAnim;
    op.popExitAnim = mPopExitAnim;
    mNumOp++;
}
```
　　接下来将执行commit操作，最后会调用commitInternal函数：
```java
int commitInternal(boolean allowStateLoss) {
    if (mCommitted) throw new IllegalStateException("commit already called");
    if (FragmentManagerImpl.DEBUG) Log.v(TAG, "Commit: " + this);
    mCommitted = true;
    if (mAddToBackStack) {
        mIndex = mManager.allocBackStackIndex(this);
    } else {
        mIndex = -1;
    }
    mManager.enqueueAction(this, allowStateLoss);//把BackStackRecord添加到队列，这里的this其实就是BackStackRecord,BackStackRecord相当于记录了一次操作（也可能是一些列的，譬如add,replace....)
    return mIndex;
}
```
　　下来我们进入到FragmentManager的enqueueAction函数中看：
```java
public void enqueueAction(Runnable action, boolean allowStateLoss) {
    if (!allowStateLoss) {
        checkStateLoss();
    }
    synchronized (this) {
        if (mActivity == null) {
            throw new IllegalStateException("Activity has been destroyed");
        }
        if (mPendingActions == null) {
            mPendingActions = new ArrayList<Runnable>();
        }
        mPendingActions.add(action);
        if (mPendingActions.size() == 1) {
            mActivity.mHandler.removeCallbacks(mExecCommit);
            mActivity.mHandler.post(mExecCommit);
        }
    }
}
```
　　接下来看mExecCommit：
```java
Runnable mExecCommit = new Runnable() {
    @Override
    public void run() {
        execPendingActions();
    }
};
```
```java
/**
 * Only call from main thread!
 */
public boolean execPendingActions() {
    if (mExecutingActions) {
        throw new IllegalStateException("Recursive entry to executePendingTransactions");
    }
    
    if (Looper.myLooper() != mActivity.mHandler.getLooper()) {
        throw new IllegalStateException("Must be called from main thread of process");
    }

    boolean didSomething = false;

    while (true) {
        int numActions;
        
        synchronized (this) {
            if (mPendingActions == null || mPendingActions.size() == 0) {
                break;
            }
            
            numActions = mPendingActions.size();
            if (mTmpActions == null || mTmpActions.length < numActions) {
                mTmpActions = new Runnable[numActions];
            }
            mPendingActions.toArray(mTmpActions);
            mPendingActions.clear();
            mActivity.mHandler.removeCallbacks(mExecCommit);
        }
        
        mExecutingActions = true;
        for (int i=0; i<numActions; i++) {
            mTmpActions[i].run();
            mTmpActions[i] = null;
        }
        mExecutingActions = false;
        didSomething = true;
    }

    if (mHavePendingDeferredStart) {
        boolean loadersRunning = false;
        for (int i=0; i<mActive.size(); i++) {
            Fragment f = mActive.get(i);
            if (f != null && f.mLoaderManager != null) {
                loadersRunning |= f.mLoaderManager.hasRunningLoaders();
            }
        }
        if (!loadersRunning) {
            mHavePendingDeferredStart = false;
            startPendingDeferredFragments();
        }
    }
    return didSomething;
}
```
　　这段代码绕来绕去，最后调用的BackStackRecord本身的run方法，而且是直接调用没有涉及到线程的切换，这也符合了commit方法名字本身的定义，transaction只是被批量提交到了主线程的任务队列里，并不是马上执行，等待主线程的looper去安排这些任务的执行。
　　现在来看看BackStackRecord的run函数干了啥：
```java
public void run() {
    if (FragmentManagerImpl.DEBUG) Log.v(TAG, "Run: " + this);

    if (mAddToBackStack) {
        if (mIndex < 0) {
            throw new IllegalStateException("addToBackStack() called after commit()");
        }
    }

    bumpBackStackNesting(1);

    Op op = mHead;
	//循环遍历存储在BackStackRecord中的记录了操作序列的双向链表。
    while (op != null) {
        switch (op.cmd) {
//根据不同的cmd类型进行不同的处理
            case OP_ADD: {
                Fragment f = op.fragment;
                f.mNextAnim = op.enterAnim;
                mManager.addFragment(f, false);//这里是个重点
            } break;
            case OP_REPLACE: {
                Fragment f = op.fragment;
                if (mManager.mAdded != null) {
					//循环遍历所有已经添加过的Fragment
                    for (int i=0; i<mManager.mAdded.size(); i++) {
                        Fragment old = mManager.mAdded.get(i);
                        if (FragmentManagerImpl.DEBUG) Log.v(TAG,
                                "OP_REPLACE: adding=" + f + " old=" + old);
						//如果ContainerId 是一样的
                        if (old.mContainerId == f.mContainerId) {
                            if (op.removed == null) {
                                op.removed = new ArrayList<Fragment>();
                            }
                            op.removed.add(old);
                            old.mNextAnim = op.exitAnim;
                            if (mAddToBackStack) {
                                old.mBackStackNesting += 1;
                                if (FragmentManagerImpl.DEBUG) Log.v(TAG, "Bump nesting of "
                                        + old + " to " + old.mBackStackNesting);
                            }
                            mManager.removeFragment(old, mTransition, mTransitionStyle);
                        }
                    }
                }
                f.mNextAnim = op.enterAnim;
                mManager.addFragment(f, false);//将Fragment添加到mManager中进行管理
            } break;
            case OP_REMOVE: {
                Fragment f = op.fragment;
                f.mNextAnim = op.exitAnim;
                mManager.removeFragment(f, mTransition, mTransitionStyle);
            } break;
            case OP_HIDE: {
                Fragment f = op.fragment;
                f.mNextAnim = op.exitAnim;
                mManager.hideFragment(f, mTransition, mTransitionStyle);
            } break;
            case OP_SHOW: {
                Fragment f = op.fragment;
                f.mNextAnim = op.enterAnim;
                mManager.showFragment(f, mTransition, mTransitionStyle);
            } break;
            case OP_DETACH: {
                Fragment f = op.fragment;
                f.mNextAnim = op.exitAnim;
                mManager.detachFragment(f, mTransition, mTransitionStyle);
            } break;
            case OP_ATTACH: {
                Fragment f = op.fragment;
                f.mNextAnim = op.enterAnim;
                mManager.attachFragment(f, mTransition, mTransitionStyle);
            } break;
            default: {
                throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
            }
        }

        op = op.next;
    }
	
	//循环操作完所有的操作序列后调用这个函数
    mManager.moveToState(mManager.mCurState, mTransition,
            mTransitionStyle, true);

    if (mAddToBackStack) {
        mManager.addBackStackState(this);
    }
}
```

　　接下来就是Fragment的状态迁移函数moveToState，我们先看状态迁移后看addFragment removeFragment这些函数
```java
void moveToState(Fragment f, int newState, int transit, int transitionStyle) {
    if (!f.mAdded && newState > Fragment.CREATED) {
        newState = Fragment.CREATED;//如果这个Fragment还没有被add,则修改newState为Created即onCreate()状态。
    }
    if (f.mRemoving && newState > f.mState) {
        // While removing a fragment, we can't change it to a higher state.
        newState = f.mState;//这个不难理解
    }
    // Defer start if requested; don't allow it to move to STARTED or higher
    // if it's not already started.
    if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.STOPPED) {
        newState = Fragment.STOPPED;
    }
    if (f.mState < newState) {
        // For fragments that are created from a layout, when restoring from
        // state we don't want to allow them to be created until they are
        // being reloaded from the layout.
        if (f.mFromLayout && !f.mInLayout) {
            return;
        }
        if (f.mAnimatingAway != null) {
            // The fragment is currently being animated...  but!  Now we
            // want to move our state back up.  Give up on waiting for the
            // animation, move to whatever the final state should be once
            // the animation is done, and then we can proceed from there.
            f.mAnimatingAway = null;
            moveToState(f, f.mStateAfterAnimating, 0, 0);
        }
        switch (f.mState) {
            case Fragment.INITIALIZING:
                if (DEBUG) Log.v(TAG, "moveto CREATED: " + f);
                if (f.mSavedFragmentState != null) {
                    f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                            FragmentManagerImpl.VIEW_STATE_TAG);
                    f.mTarget = getFragment(f.mSavedFragmentState,
                            FragmentManagerImpl.TARGET_STATE_TAG);
                    if (f.mTarget != null) {
                        f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                    }
                    f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                            FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                    if (!f.mUserVisibleHint) {
                        f.mDeferStart = true;
                        if (newState > Fragment.STOPPED) {
                            newState = Fragment.STOPPED;
                        }
                    }
                }
                f.mActivity = mActivity;
                f.mFragmentManager = mActivity.mFragments;
                f.mCalled = false;
                f.onAttach(mActivity);
                if (!f.mCalled) {
                    throw new SuperNotCalledException("Fragment " + f
                            + " did not call through to super.onAttach()");
                }
                mActivity.onAttachFragment(f);
                
                if (!f.mRetaining) {
                    f.mCalled = false;
                    f.onCreate(f.mSavedFragmentState);
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onCreate()");
                    }
                }
                f.mRetaining = false;
                if (f.mFromLayout) {
                    // For fragments that are part of the content view
                    // layout, we need to instantiate the view immediately
                    // and the inflater will take care of adding it.
                    f.mView = f.onCreateView(f.getLayoutInflater(f.mSavedFragmentState),
                            null, f.mSavedFragmentState);
                    if (f.mView != null) {
                        f.mView.setSaveFromParentEnabled(false);
                        if (f.mHidden) f.mView.setVisibility(View.GONE);
                        f.onViewCreated(f.mView, f.mSavedFragmentState);
                    }
                }
            case Fragment.CREATED:
                if (newState > Fragment.CREATED) {
                    if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                    if (!f.mFromLayout) {
                        ViewGroup container = null;
                        if (f.mContainerId != 0) {
                            container = (ViewGroup)mActivity.findViewById(f.mContainerId);
                            if (container == null && !f.mRestored) {
                                throw new IllegalArgumentException("No view found for id 0x"
                                        + Integer.toHexString(f.mContainerId)
                                        + " for fragment " + f);
                            }
                        }
                        f.mContainer = container;
                        f.mView = f.onCreateView(f.getLayoutInflater(f.mSavedFragmentState),
                                container, f.mSavedFragmentState);
                        if (f.mView != null) {
                            f.mView.setSaveFromParentEnabled(false);
                            if (container != null) {
                                Animator anim = loadAnimator(f, transit, true,
                                        transitionStyle);
                                if (anim != null) {
                                    anim.setTarget(f.mView);
                                    anim.start();
                                }
                                container.addView(f.mView);
                            }
                            if (f.mHidden) f.mView.setVisibility(View.GONE);
                            f.onViewCreated(f.mView, f.mSavedFragmentState);
                        }
                    }
                    
                    f.mCalled = false;
                    f.onActivityCreated(f.mSavedFragmentState);
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onActivityCreated()");
                    }
                    if (f.mView != null) {
                        f.restoreViewState();
                    }
                    f.mSavedFragmentState = null;
                }
            case Fragment.ACTIVITY_CREATED:
            case Fragment.STOPPED:
                if (newState > Fragment.STOPPED) {
                    if (DEBUG) Log.v(TAG, "moveto STARTED: " + f);
                    f.mCalled = false;
                    f.performStart();
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onStart()");
                    }
                }
            case Fragment.STARTED:
                if (newState > Fragment.STARTED) {
                    if (DEBUG) Log.v(TAG, "moveto RESUMED: " + f);
                    f.mCalled = false;
                    f.mResumed = true;
                    f.onResume();
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onResume()");
                    }
                    // Get rid of this in case we saved it and never needed it.
                    f.mSavedFragmentState = null;
                    f.mSavedViewState = null;
                }
        }
    } else if (f.mState > newState) {
        switch (f.mState) {
            case Fragment.RESUMED:
                if (newState < Fragment.RESUMED) {
                    if (DEBUG) Log.v(TAG, "movefrom RESUMED: " + f);
                    f.mCalled = false;
                    f.onPause();
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onPause()");
                    }
                    f.mResumed = false;
                }
            case Fragment.STARTED:
                if (newState < Fragment.STARTED) {
                    if (DEBUG) Log.v(TAG, "movefrom STARTED: " + f);
                    f.mCalled = false;
                    f.performStop();
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onStop()");
                    }
                }
            case Fragment.STOPPED:
            case Fragment.ACTIVITY_CREATED:
                if (newState < Fragment.ACTIVITY_CREATED) {
                    if (DEBUG) Log.v(TAG, "movefrom ACTIVITY_CREATED: " + f);
                    if (f.mView != null) {
                        // Need to save the current view state if not
                        // done already.
                        if (!mActivity.isFinishing() && f.mSavedViewState == null) {
                            saveFragmentViewState(f);
                        }
                    }
                    f.mCalled = false;
                    f.performDestroyView();
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onDestroyView()");
                    }
                    if (f.mView != null && f.mContainer != null) {
                        Animator anim = null;
                        if (mCurState > Fragment.INITIALIZING && !mDestroyed) {
                            anim = loadAnimator(f, transit, false,
                                    transitionStyle);
                        }
                        if (anim != null) {
                            final ViewGroup container = f.mContainer;
                            final View view = f.mView;
                            final Fragment fragment = f;
                            container.startViewTransition(view);
                            f.mAnimatingAway = anim;
                            f.mStateAfterAnimating = newState;
                            anim.addListener(new AnimatorListenerAdapter() {
                                @Override
                                public void onAnimationEnd(Animator anim) {
                                    container.endViewTransition(view);
                                    if (fragment.mAnimatingAway != null) {
                                        fragment.mAnimatingAway = null;
                                        moveToState(fragment, fragment.mStateAfterAnimating,
                                                0, 0);
                                    }
                                }
                            });
                            anim.setTarget(f.mView);
                            anim.start();

                        }
                        f.mContainer.removeView(f.mView);
                    }
                    f.mContainer = null;
                    f.mView = null;
                }
            case Fragment.CREATED:
                if (newState < Fragment.CREATED) {
                    if (mDestroyed) {
                        if (f.mAnimatingAway != null) {
                            // The fragment's containing activity is
                            // being destroyed, but this fragment is
                            // currently animating away.  Stop the
                            // animation right now -- it is not needed,
                            // and we can't wait any more on destroying
                            // the fragment.
                            Animator anim = f.mAnimatingAway;
                            f.mAnimatingAway = null;
                            anim.cancel();
                        }
                    }
                    if (f.mAnimatingAway != null) {
                        // We are waiting for the fragment's view to finish
                        // animating away.  Just make a note of the state
                        // the fragment now should move to once the animation
                        // is done.
                        f.mStateAfterAnimating = newState;
                        newState = Fragment.CREATED;
                    } else {
                        if (DEBUG) Log.v(TAG, "movefrom CREATED: " + f);
                        if (!f.mRetaining) {
                            f.mCalled = false;
                            f.onDestroy();
                            if (!f.mCalled) {
                                throw new SuperNotCalledException("Fragment " + f
                                        + " did not call through to super.onDestroy()");
                            }
                        }

                        f.mCalled = false;
                        f.onDetach();
                        if (!f.mCalled) {
                            throw new SuperNotCalledException("Fragment " + f
                                    + " did not call through to super.onDetach()");
                        }
                        if (!f.mRetaining) {
                            makeInactive(f);
                        } else {
                            f.mActivity = null;
                            f.mFragmentManager = null;
                        }
                    }
                }
        }
    }
    
    f.mState = newState;
}
```
本文这里不多说了，这个地方有点多，反正就是状态跃迁，我准备重新开一篇文章重新来说这个moveState....
![](http://i.imgur.com/pRWsWOI.png)

现在来叔叔addFragment这个玩意儿：