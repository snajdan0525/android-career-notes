# Android 绘制和Activity生命周期的关系 #



**getViewTreeObserver().addOnGlobalLayoutListener()**
----------

　　通常情况下,在onCreate函数中通过View.getWidth()和View.getHeight()是无法获取到View的高度和宽度的，这是因为View组件的布局是在onResume回调完成后才进行的。一般情况下,我们采用getViewTreeObserver().addOnGlobalLayoutListener()来获取View的宽和高。
　　OnGlobalLayoutListener 是ViewTreeObserver的内部类，当一个视图树的布局发生改变时，可以被ViewTreeObserver监听到，这是一个注册监听视图树的观察者(observer)，在视图树的全局事件改变时得到通知,onGlobalLayout将在控件完成绘制后被调用。ViewTreeObserver不能直接实例化，而是通过getViewTreeObserver()获得。
```java
private int mHeaderViewHeight;
private View mHeaderView;

mHeaderView.getViewTreeObserver().addOnGlobalLayoutListener(
    new OnGlobalLayoutListener() {
        @Override
        public void onGlobalLayout() {                                                                                                                                                                                                                   
            mHeaderViewHeight = mHeaderView.getHeight();
 			final int version = VERSION.SDK_INT;
                /*
                 * 移除监听器，避免重复调用
                 */
                // 判断sdk版本，removeGlobalOnLayoutListener在API 16以后不再使用
                if (version >= 16) {
                    mHeaderView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
                } else {
                    mHeaderView.getViewTreeObserver().removeGlobalOnLayoutListener(this);
                }
        }
});
```
　　但是需要注意的是OnGlobalLayoutListener可能会被多次触发，因此在得到了高度之后，要将OnGlobalLayoutListener注销掉。另外mHeaderViewHeight和mHeaderView都需要写在当前java文件类（比如Activity）的成员变量中。不能直接在onCreate中定义否则会编译不通过,报错：
> Cannot refer to a non-final variable sHeight inside an inner class defined in a different method


**onWindowFocusChanged**
----------
　　上面提到了View组件的布局是在onResume回调完成后才进行的。通过打印Log发现如下的一个顺序。
```java
OnCreate=>>>>>>>>>width==0,height==0
onStart=>>>>>>>>>width==0,height==0
onPostCreate=>>>>>>>>>width==0,height==0
onResume=>>>>>>>>>width==0,height==0
onPostResume=>>>>>>>>>width==0,height==0
onAttachedToWindow=>>>>>>>>>width==0,height==0
onWindowFocusChanged=>>>>>>>>>width==222,height==58
```
　　通过打印可看出，在onResume回调完成后,会继续调用onWindowFocusChanged,这个函数调用时完成了控件的绘制，所以通过onWindowFocusChanged也可以得到View的宽和高，但是这个函数获取宽高有一个缺点即这个函数只在Activity中有效，在Fragment中是无法通过这个方法获取宽高的。


**使用 View 的 post() 方法获取宽高**
----------

```java
@Override
protected void onCreate(final Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    hello_world_tv = (TextView) this.findViewById(R.id.hello_world_tv);
    hello_world_tv.post(new Runnable() {

        @Override
        public void run() {
            getWidthAndHeight("post");
        }
    });
}
```