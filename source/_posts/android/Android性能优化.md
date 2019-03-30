---
title: Android 性能优化
tags:
  - Android优化
categories:
  - Android
  - 高级
abbrlink: 19777
date: 2019-02-21 21:16:17
---

## Android 性能优化

### 一、布局优化

布局优化的思想很简单，就是尽量减少布局文件的层级，这个道理是很浅显的，布局中的层级少了，这就意味着Android绘制时的工作量少了，那么程序的性能自然就高了。

如何进行布局优化呢？首先删除布局中无用的控件和层级，其次有选择地使用性能较低的ViewGroup，比如RelativeLayout。如果布局中既可以使用LinearLayout 也可以使用RelativeLayout，那么就采用LinearLayout，这是因为RelativeLayout的功能比较复杂，它的布局过程需要花费更多的CPU时间。FrameLayout和LinearLayout一样都是一种简单高效的ViewGroup，因此可以考虑使用它们，但是很多时候单纯通过一个LinearLayout 或者FrameLayout无法实现产品效果，需要通过嵌套的方式来完成。这种情况下还是建议采用RelativeLayout，因为ViewGroup的嵌套就相当于增加了布局的层级，同样会降低程序的性能。

布局优化的另外一种手段是采用<include\>标签、<merge\>标签和ViewStub。<include\>标签主要用于布局重用，<merge\>标签一般和<include\>配合使用，它可以降低减少布局的层级，而ViewStub则提供了按需加载的功能，当需要时才会将ViewStub中的布局加载到内存，这提高了程序的初始化效率，下面分别介绍它们的使用方法。

####   <include\> 标签
<include\>标签可以将一个指定的布局文件加载到当前的布局文件中，如下所示。

``` xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <include layout="@layout/titlebar"></include>
</LinearLayout>
```

上面的代码中，@layout/titlebar指定了另外一个布局文件，通过这种方式就不用把titlebar这个布局文件的内容再重复写一遍了，这就是<include\>的好处。<include\>标签只支持以android:layout\_开头的属性，比如 android:layout\_width、android:layout\_height，其他属性是不支持的，比如android:background。当然，android:id这个属性是个特例，如果<include\>指定了这个id属性，同时被包含的布局文件的根元素也指定了id属性，那么以<include\>指定的id属性为准。需要注意的是，如果<include\>标签指定了android:layout\_*这种属性，那么要求android:layout\_width和android:layout\_height必须存在，否则其他android:layout\_*
形式的属性无法生效，下面是一个指定了android:layout\_*属性的示例。

``` xml
    <include
        android:id="@+id/test"
        layout="@layout/titlebar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"></include>
```
#### <merge\>标签

<merge\>标签一般和<font color=red><include\></font>标签一起使用从而减少布局的层级。在上面的示例中，由于当前布局是一个竖直方向的LinearLayout，这个时候如果被包含的布局文件中也采用了竖直方向的LinearLayout，那么显然被包含的布局文件中的LinearLayout是多余的，通过
<merge\>标签就可以去掉多余的那一层LinearLayout，如下所示。

``` xml
<merge xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</merge>
```
#### ViewStub

ViewStub继承了View，它非常轻量级且宽/高都是0，因此它本身不参与任何的布局和绘制过程。ViewStub的意义在于按需加载所需的布局文件，在实际开发中，有很多布局文件在正常情况下不会显示，比如网络异常时的界面，这个时候就没有必要在整个界面初始化的时候将其加载进来，通过ViewStub就可以做到在使用的时候再加载，提高了程序初始化时的性能。下面是一个ViewStub的示例：
``` xml
    <ViewStub 
        android:id="@+id/stub_import"
        android:inflatedId="@+id/panel_import"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom"
        android:layout="@layout/fly_distance_time_widget"
        />
```

其中stubimport是ViewStub的id，而 panel_import是layout/layout network error这个布局的根元素的id。如何做到按需加载呢？在需要加载ViewStub中的布局时，可以按照如下两种方式进行：
``` java
((ViewStub) findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
```
或者

``` java
View importPanel=((Viewstub) findViewById(R.id.stub_import)).inflate();
```
当ViewStub通过setVisibility或者inflate方法加载后，ViewStub就会被它内部的布局<font color=blue>替换掉</font>，这个时候ViewStub就不再是整个布局结构中的一部分了。另外，目前ViewStub还不支持<merge\>标签。

### 二、绘制优化

绘制优化是指View的onDraw方法要避免执行大量的操作，这主要体现在两个方面。

首先，onDraw中不要创建新的局部对象，这是因为onDraw方法可能会被频繁调用，这样就会在一瞬间产生大量的临时对象，这不仅占用了过多的内存而且还会导致系统更加频繁gc，降低了程序的执行效率。

另外一方面，onDraw方法中不要做耗时的任务，也不能执行成千上万次的循环操作，尽管每次循环都很轻量级，但是大量的循环仍然十分抢占CPU的时间片，这会造成View的绘制过程不流畅。按照Google官方给出的性能优化典范中的标准，View的绘制帧率保证60fps是最佳的，这就要求每帧的绘制时间不超过16ms（16ms=1000/60），虽然程序很难保证16ms这个时间，但是尽量降低onDraw方法的复杂度总是切实有效的。

### 三、内存泄漏优化

#### 场景1：静态变量导致的内存泄露

下面这种情形是一种最简单的内存泄露，相信读者都不会这么干，下面的代码将导致Activity无法正常销毁，因此静态变量sContext引用了它。
``` java
public class Leack2Activity extends AppCompatActivity {
    private static Context sContext=null;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leack2);
        sContext=this;
    }
}
```
上面的代码也可以改造一下，如下所示。sView是一个静态变量，它内部持有了当前Activity，所以Activity仍然无法释放，估计读者也都明白。

``` java
public class Leack2Activity extends AppCompatActivity {
    private static View sView=null;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leack2);
        sView=new View(this);
    }
}
```

#### 场景2：单例模式导致的内存泄漏

静态变量导致的内存泄露都太过于明显，相信读者都不会犯这种错误，而单例模式所带来的内存泄露是我们容易忽视的，如下所示。首先提供一个单例模式的TestManager，TestManager可以接收外部的注册并将外部的监听器存储起来。

``` java
public class TestManager {
    private List<OnDataArrivedListener> mOndataArrivedListners = new ArrayList<>();
    private static class SingletonHolder {
        public static final TestManager INSTANCE = new TestManager();
    }
    private TestManager() {
    }
    public static TestManager getInstance() {
        return SingletonHolder.INSTANCE;
    }
    public synchronized void registerListener(OnDataArrivedListener listener) {
        if (!mOndataArrivedListners.contains(listener)) {
            mOndataArrivedListners.add(listener);
        }
    }
    public synchronized void unregisterListener(OnDataArrivedListener listener) {
        mOndataArrivedListners.remove(listener);
    }
    public interface OnDataArrivedListener {
        public void onDataArrived(Object data);
    }
}
```
接着再让Activity实现OnDataArrivedListener 接口并向TestManager注册监听，如下所示。下面的代码由于缺少解注册的操作所以会引起内存泄露，泄露的原因是Activity的对象被单例模式的TestManager所持有，而单例模式的特点是其生命周期和Application保持一致，因此Activity对象无法被及时释放。

``` java
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leack2);
       TestManager.getInstance().registerListener(this);
    }
```

#### 场景3：属性动画导致的内存泄露

从Android3.0开始，Google提供了属性动画，属性动画中有一类无限循环的动画，如果在Activity中播放此类动画且没有在onDestroy中去停止动画，那么动画会一直播放下去，尽管已经无法在界面上看到动画效果了，并且这个时候Activity的View会被动画持有，而View又持有了Activity，最终Activity无法释放。下面的动画是无限动画，会泄露当前Activity，解决方法是在Activity的onDestroy 中调用animator.cancel()来停止动画。

``` java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leack2);
        View view = new View(this);
        ObjectAnimator animator = ObjectAnimator.ofFloat(view, "rotation", 0, 360).setDuration(2000);
        animator.setRepeatCount(ValueAnimator.INFINITE);
        animator.start();
        // animator.cancel();
    }
```

### 四、响应速度优化和ANR日志分析

响应速度优化的核心思想是避免在主线程中做耗时操作，但是有时候的确有很多耗时操作，怎么办呢？可以将这些耗时操作放在线程中去执行，即采用异步的方式执行耗时操作。响应速度过慢更多地体现在Activity的启动速度上面，如果在主线程中做太多事情，会导致Activity启动时出现黑屏现象，甚至出现ANR。Android规定，Activity如果<font color=green>5秒钟</font>之内无法响应屏幕触摸事件或者键盘输入事件就会出现ANR，而BroadcastReceiver如果<font color=green>10秒钟</font>之内还未执行完操作也会出现ANR。在实际开发中，ANR是很难从代码上发现的，如果在开发过程中遇到了ANR，那么怎么定位问题呢？其实当一个进程发生ANR了以后，系统会在<font color=green>/data/anr</font>目录下创建一个文件traces.txt，通过分析这个文件就能定位出ANR的原因，下面模拟一个ANR的场景。下面的代码在Activity的onClick中休眠30s，程序运行后持续点击屏幕，应用一定会出现ANR：

``` java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.jump).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //导致ANR的原因
                SystemClock.sleep(30 * 1000);
                startActivity(new Intent(MainActivity.this, LeackActivity.class));
            }
        });
        findViewById(R.id.jump2).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startActivity(new Intent(MainActivity.this, Leack2Activity.class));
            }
        });
    }
```

**traces.txt 文件内容**

``` java
DALVIK THREADS (19):
"main" prio=5 tid=1 Sleeping //关键字
  | group="main" sCount=1 dsCount=0 obj=0x73a7f970 self=0xf4406800
  | sysTid=2921 nice=0 cgrp=default sched=0/0 handle=0xf77c1160
  | state=S schedstat=( 424227458 15422192 277 ) utm=24 stm=18 core=3 HZ=100
  | stack=0xff44d000-0xff44f000 stackSize=8MB
  | held mutexes=
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x120b1730> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1031)
  - locked <0x120b1730> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:985)
  at android.os.SystemClock.sleep(SystemClock.java:120)
  at com.mk.analyzememoryleak.MainActivity$1.onClick(MainActivity.java:18)
  at android.view.View.performClick(View.java:4756)
  at android.view.View$PerformClick.run(View.java:19749)
  at android.os.Handler.handleCallback(Handler.java:739)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:135)
  at android.app.ActivityThread.main(ActivityThread.java:5221)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:899)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:694)

"Heap thread pool worker thread 0" prio=5 tid=2 Native (still starting up)
  | group="" sCount=1 dsCount=0 obj=0x0 self=0xf000d400
  | sysTid=2924 nice=0 cgrp=default sched=0/0 handle=0xf4456800
  | state=S schedstat=( 774337 292432 9 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xf39c8000-0xf39ca000 stackSize=1020KB
  | held mutexes=
  kernel: futex_wait_queue_me+0xc0/0x110
  kernel: futex_wait+0x112/0x250
  kernel: do_futex+0xdc/0xb30
  kernel: compat_SyS_futex+0x75/0x150
  kernel: do_syscall_32_irqs_off+0x5f/0x180
  kernel: entry_INT80_compat+0x36/0x50
  native: #00 pc 00012d80  /system/lib/libc.so (syscall+32)
  native: #01 pc 000fdc97  /dev/ashmem/dalvik-Heap thread pool worker thread 0 (deleted) (???)
  (no managed stack frames)

```



下面的代码也会导致ANR，原因是这样的，在Activity的onCreate中开启了一个线程，在线程中执行 testANR()，而testANR()和initView()都被加了同一个锁，为了百分之百让testANR()先获得锁，特意在执行initView()之前让主线程休眠了10ms，这样一来initView()肯定会因为等待testANR()所持有的锁而被同步住，这样就产生了一个稍微复杂些的ANR。
这个ANR是很参考意义的，这样的代码很容易在实际开发中出现，尤其是当调用关系比较复杂时，这个时候分析ANR日志就显得异常重要了。下面的代码中虽然已经将耗时操作放在线程中了，按道理就不会出现ANR了，但是仍然要注意子线程和主线程抢占同步锁的情况。

```java

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.jump).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        testANR();
                    }
                }).start();

                SystemClock.sleep(10);
                intView();
            }
        });
    }
    
    private synchronized void testANR(){
        SystemClock.sleep(30*1000);
    }

    private synchronized void intView() {
    }
```





**traces.txt 文件内容**

```java
DALVIK THREADS (20):
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x73a7f970 self=0xf4406800
  | sysTid=3340 nice=0 cgrp=default sched=0/0 handle=0xf77c1160
  | state=S schedstat=( 392067727 9754393 244 ) utm=23 stm=15 core=3 HZ=100
  | stack=0xff44d000-0xff44f000 stackSize=8MB
  | held mutexes=
  at com.mk.analyzememoryleak.MainActivity.intView(MainActivity.java:-1)
  //看这里在等待锁
  - waiting to lock <0x12bc211b> (a com.mk.analyzememoryleak.MainActivity) held by thread 20
  at com.mk.analyzememoryleak.MainActivity.access$100(MainActivity.java:9)
  at com.mk.analyzememoryleak.MainActivity$1.onClick(MainActivity.java:26)
  at android.view.View.performClick(View.java:4756)
  at android.view.View$PerformClick.run(View.java:19749)
  at android.os.Handler.handleCallback(Handler.java:739)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:135)
  at android.app.ActivityThread.main(ActivityThread.java:5221)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:899)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:694)
```

上面的情况稍微复杂一些，需要逐步分析。首先看主线程，如下所示。可以看得出主线程在initView方法中正在等待一个锁<0x12bc211b>，这个锁的类型是一个MainActivity对象，并且这个锁已经被线程id为20（即tid=20）的线程持有了，因此需要再看一下线程20的情况。

```java
"Thread-245" prio=5 tid=20 Sleeping
  | group="main" sCount=1 dsCount=0 obj=0x12c27be0 self=0xed233c00
  | sysTid=3386 nice=0 cgrp=default sched=0/0 handle=0xf445c980
  | state=S schedstat=( 601859 223899 2 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xe15fe000-0xe1600000 stackSize=1036KB
  | held mutexes=
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x0e5966eb> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1031)
  - locked <0x0e5966eb> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:985)
  at android.os.SystemClock.sleep(SystemClock.java:120)
  at com.mk.analyzememoryleak.MainActivity.testANR(MainActivity.java:36)
  - locked <0x12bc211b> (a com.mk.analyzememoryleak.MainActivity)
  at com.mk.analyzememoryleak.MainActivity.access$000(MainActivity.java:9)
  at com.mk.analyzememoryleak.MainActivity$1$1.run(MainActivity.java:21)
  at java.lang.Thread.run(Thread.java:818)
```

tid是20的线程就是“Thread-245”，就是它持有了主线程所需的锁，可以看出“Thread-245”正在sleep，sleep的原因是MainActivity的21行，即testANR方法。这个时候可以发现testANR方法和主线程的initView方法都加了synchronized关键字，表明它们在竞争同一个锁，即当前Activity的对象锁，这样一来ANR的原因就明确了，接着就可以修改代码了。
上面分析了两个ANR的实例，尤其是第二个ANR在实际开发中很容易出现，我们首先要有意识地避免出现ANR，其次出现ANR了也不要着急，通过分析traces文件即可定位问题。

