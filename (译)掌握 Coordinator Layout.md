(译)掌握 Coordinator Layout
发表于 2015-11-12   |   分类于 android   |   3条评论
在今年的 Google I/O 15上Google 发布了 新的支持库 ，其中有好几个组件与Material Design设计密切相关,在这些新组件中，你可以找到有几个类似于ViewGroup 的控件，如 AppbarLayout,CollapsingToolbarLayout 和 CoordinatorLayout.
这些ViewGroups 控件提供了非常强大的功能，我决定写一篇文章来介绍相关的配置和技巧。

CoordinatorLayout
顾名思义，这个控件的目的就是协调它里面View的行为。

请看下面的图片：



在这个例子中我们可以看到View之间是如何相互配合的，View会根据其他View的变动做相应的变化。

以下是CoordinatorLayout的简单使用例子：

<?xml version="1.0" encoding="utf-8"?>

<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/background_light"
    android:fitsSystemWindows="true"
    >

    <android.support.design.widget.AppBarLayout
        android:id="@+id/main.appbar"
        android:layout_width="match_parent"
        android:layout_height="300dp"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
        android:fitsSystemWindows="true"
        >

        <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/main.collapsing"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_scrollFlags="scroll|exitUntilCollapsed"
            android:fitsSystemWindows="true"
            app:contentScrim="?attr/colorPrimary"
            app:expandedTitleMarginStart="48dp"
            app:expandedTitleMarginEnd="64dp"
            >

            <ImageView
                android:id="@+id/main.backdrop"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:scaleType="centerCrop"
                android:fitsSystemWindows="true"
                android:src="@drawable/material_flat"
                app:layout_collapseMode="parallax"
                />

            <android.support.v7.widget.Toolbar
                android:id="@+id/main.toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
                app:layout_collapseMode="pin"
                />
        </android.support.design.widget.CollapsingToolbarLayout>
    </android.support.design.widget.AppBarLayout>

    <android.support.v4.widget.NestedScrollView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"
        >

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:textSize="20sp"
            android:lineSpacingExtra="8dp"
            android:text="@string/lorem"
            android:padding="@dimen/activity_horizontal_margin"
            />
    </android.support.v4.widget.NestedScrollView>

    <android.support.design.widget.FloatingActionButton
        android:layout_height="wrap_content"
        android:layout_width="wrap_content"
        android:layout_margin="@dimen/activity_horizontal_margin"
        android:src="@drawable/ic_comment_24dp"
        app:layout_anchor="@id/main.appbar"
        app:layout_anchorGravity="bottom|right|end"
        />
</android.support.design.widget.CoordinatorLayout>
我们看一下这个layout结构，CoordinatorLayout包含3个子控件：
AppbarLayout， scrolleable view 和 anchoredFloatingActionBar。

<CoordinatorLayout>
    <AppbarLayout/>
    <scrollableView/>
    <FloatingActionButton/>
</CoordinatorLayout>
AppBarLayout
AppBarLayout 是继承LinerLayout实现的一个ViewGroup容器组件,
默认的AppBarLayout是垂直方向的, 可以管理其中的控件在内容滚动时的行为。

这听起来可能有点令人困惑，我想一张图片可以胜过千言万语，特别时GIF图片：


AppBarLayout在这个例子中时蓝色的View，在其下放置了一个可以缩放的图片，其中包含一个Toolbar，
一个LinearLayout（包含标题和副标题），以及一个TabLayout。

我们可以通过设置layout_scrollFlags参数，来控制AppBarLayout中的控件行为。
在我们的这个例子中，大部分View的layout_scrollFlags都设置为scroll，如果没有设置的话，
当可滚动的View进行滚动时，这些没设置为scroll的View位置会保持不变；

layout_scrollFlags设置上snap值则可以避免进入动画中间状态（ mid-animation-states），
这意味着动画会一直持续到View完全显示或完全隐藏为止。

LinearLayout其中包含了一个标题和一个副标题，当用户向上移动时LinearLayout是一直显示的，直到移出屏幕（enterAlways）;

TabLayout会一直是可见的，因为我们没有在TabLayout上设置任何flag。

正如你所见，AppbarLayout的强大管理能力是通过在View上设置不同scroll flags实现的。

<AppBarLayout>
    <CollapsingToolbarLayout
        app:layout_scrollFlags="scroll|snap"
        />

    <Toolbar
        app:layout_scrollFlags="scroll|snap"
        />

    <LinearLayout
        android:id="+id/title_container"
        app:layout_scrollFlags="scroll|enterAlways"
        />

    <TabLayout /> <!-- no flags -->
</AppBarLayout>
这些参数的设置请参考 Google Developers docs。
不过我建议还是通过代码练习来掌握它。我在文章的末尾提供了几个Github上的例子。

AppbarLayout flags

SCROLL_FLAG_ENTER_ALWAYS: 当任何向下滚动事件发生时, View都会移入 , 不管scrolling view 是否正在滚动。

SCROLL_FLAG_ENTER_ALWAYS_COLLAPSED: ‘enterAlways’的附加标识，它使得returning view恢复到指定的最小高度后才开始显示，然后再慢慢展开。

SCROLL_FLAG_EXIT_UNTIL_COLLAPSED: 但向上移出屏幕时，View会一直收缩到最小高度后，再移出屏幕。

SCROLL_FLAG_SCROLL: View 会根据滚动事件进行移动。

SCROLL_FLAG_SNAP: 但滚动结束时，如果View只有部分可见，它将会自动滑动到最近的边界（完全可见或完全隐藏）

CoordinatorLayout Behaviors
让我们做一些测试，打开Android Studio（>= 1.4），根据模板Scrolling Activity创建一个项目，
不需要修改任何代码，以下就是运行后的界面：


如果我们查看生成的代码，不管layouts或java类中我们都不能找到Fab在滚动时变化的动画，为什么呢？

答案在FloatingActionButton的源代码里，自动 Android Studio v1.2 加入了java反编译功能，
我们使用ctrl/cmd + click可以查看源码，看看到底发生了什么：

/*
 * Copyright (C) 2015 The Android Open Source Project
 *
 *  Floating action buttons are used for a
 *  special type of promoted action.
 *  They are distinguished by a circled icon
 *  floating above the UI and have special motion behaviors
 *  related to morphing, launching, and the transferring anchor point.
 *
 *  blah.. blah..
 */
@CoordinatorLayout.DefaultBehavior(
    FloatingActionButton.Behavior.class)
public class FloatingActionButton extends ImageButton {
    ...

    public static class Behavior
        extends CoordinatorLayout.Behavior<FloatingActionButton> {

        private boolean updateFabVisibility(
           CoordinatorLayout parent, AppBarLayout appBarLayout,
           FloatingActionButton child {

           if (a long condition) {
                // If the anchor's bottom is below the seam,
                // we'll animate our FAB out
                child.hide();
            } else {
                // Else, we'll animate our FAB back in
                child.show();
            }
        }
    }

    ...
}
负责缩放动画的是design library新引入的元素叫做Behavior, 在这里是CoordinatorLayout.Behavior<FloatingAcctionButton>, 它根据一些滚动条件，判断是否显示FAB。

SwipeDismissBehavior

深入design support library的代码，我们会发现一个新的类：SwipeDismissBehavior，使用这个Behavior，
我们可以很容易的使用CoordinatorLayout实现滑动删除功能:


@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_swipe_behavior);
    mCardView = (CardView) findViewById(R.id.swype_card);

    final SwipeDismissBehavior<CardView> swipe
        = new SwipeDismissBehavior();

        swipe.setSwipeDirection(
            SwipeDismissBehavior.SWIPE_DIRECTION_ANY);

        swipe.setListener(
            new SwipeDismissBehavior.OnDismissListener() {
            @Override public void onDismiss(View view) {
                Toast.makeText(SwipeBehaviorExampleActivity.this,
                    "Card swiped !!", Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onDragStateChanged(int state) {}
        });

        LayoutParams coordinatorParams =
            (LayoutParams) mCardView.getLayoutParams();

        coordinatorParams.setBehavior(swipe);
    }
Custom Behaviors
创建自定义Behaviors，并没有想象的那么难，首先我们得搞清楚两个核心元素 child 和 dependency.


Childs and dependencies

child 是指需要应用behavior的View ，dependency 担任触发behavior的角色，并与child进行互动。
在这个例子中， child 是ImageView， dependency 是Toolbar，如果Toolbar发生移动，ImageView也会做相应的移动。



现在我们已经知道概念了，接着我们看看怎么实现，
第一步我们需要继承CoordinatorLayout.Behavior，T是指某一个View，
在我们的例子中是ImageView， 继承后，我们必须实现以下2个方法:

layoutDependsOn
onDependentViewChanged
layoutDependsOn方法在每次layout发生变化时都会调用，我们需要在dependency控件发生变化时返回True，在我们的例子中是用户在屏幕上滑动时（因为Toolbar发生了移动），然后我们需要让child做出相应的反应。

@Override
  public boolean layoutDependsOn(     
     CoordinatorLayout parent,
     CircleImageView, child,
     View dependency) {

     return dependency instanceof Toolbar;
 }
一旦layoutDependsOn返回了True，第二个方法onDependentViewChanged就会被调用，
在这个方法里我们需要实现动画，转场等效果。

public boolean onDependentViewChanged(
      CoordinatorLayout parent,
      CircleImageView avatar,
      View dependency) {

      modifyAvatarDependingDependencyState(avatar, dependency);
   }

   private void modifyAvatarDependingDependencyState(
    CircleImageView avatar, View dependency) {
        //  avatar.setY(dependency.getY());
        //  avatar.setBlahBlat(dependency.blah / blah);
    }
整合后的代码：

public static class AvatarImageBehavior
   extends CoordinatorLayout.Behavior<CircleImageView> {

   @Override
   public boolean layoutDependsOn(
    CoordinatorLayout parent,
    CircleImageView, child,
    View dependency) {

    return dependency instanceof Toolbar;
  }

  public boolean onDependentViewChanged(   
    CoordinatorLayout parent,
    CircleImageView avatar,
    View dependency) {
      modifyAvatarDependingDependencyState(avatar, dependency);
   }

  private void modifyAvatarDependingDependencyState(
    CircleImageView avatar, View dependency) {
        //  avatar.setY(dependency.getY());
        //  avatar.setBlahBlah(dependency.blah / blah);
    }    
}
Resources
Coordinator Behavior Example - Github
Coordinator Examples - Github
Introduction to coordinator layout on Android - Grzesiek Gajewski
本文译自：http://saulmm.github.io/mastering-coordinator/

#Material Design #android