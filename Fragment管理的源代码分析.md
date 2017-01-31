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
　　BackStackRecord是个啥：BackStackRecord是FragmentTransaction的子类，实现了对Fragment操作的接口并且还有一些自己独特的东西。
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
从next， prev 两个指针可以看出，Op是作为一个链表的节点而存在的，因此FragmentTransaction肯定是在一个链表中存储了一次事务中的所有需要执行的操作。
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
    mManager.enqueueAction(this, allowStateLoss);
    return mIndex;
}
```