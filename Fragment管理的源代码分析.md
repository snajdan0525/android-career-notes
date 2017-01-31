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
    while (op != null) {
        switch (op.cmd) {
            case OP_ADD: {
                Fragment f = op.fragment;
                f.mNextAnim = op.enterAnim;
                mManager.addFragment(f, false);
            } break;
            case OP_REPLACE: {
                Fragment f = op.fragment;
                if (mManager.mAdded != null) {
                    for (int i=0; i<mManager.mAdded.size(); i++) {
                        Fragment old = mManager.mAdded.get(i);
                        if (FragmentManagerImpl.DEBUG) Log.v(TAG,
                                "OP_REPLACE: adding=" + f + " old=" + old);
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
                mManager.addFragment(f, false);
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

    mManager.moveToState(mManager.mCurState, mTransition,
            mTransitionStyle, true);

    if (mAddToBackStack) {
        mManager.addBackStackState(this);
    }
}
```