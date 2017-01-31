```java
@Override
public FragmentTransaction beginTransaction() {
    return new BackStackRecord(this);//beginTransaction实则返回的是个BackStackRecord(FragmentTransaction的子类),
}
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
　　BackStackRecord是个啥：BackStackRecord是FragmentTransaction的子类，实现了对Fragment操作的接口。