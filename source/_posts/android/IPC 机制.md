---
title: Android IPC 机制
tags:
  - Android
  - Android 进程间通信
categories:
  - Android
  - 高级
abbrlink: 20135
date: 2019-03-22 21:16:17
---



[TOC]

> 本文内容是看Android 开发者艺术 得来，如果侵权，请联系本人。

## 一、Android IPC 简介

IPC是Inter-Process Communication的缩写，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。说起进程间通信，我们首先要理解什么是进程，什么是线程，进程和线程是截然不同的概念。按照操作系统中的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。一个进程可以包含多个线程，因此进程和线程是包含与被包含的关系。最简单的情况下，一个进程中可以只有一个线程，即主线程，在Android里面主线程也叫UI线程，在UI线程里才能操作界面元素。很多时候，一个进程中需要执行大量耗时的任务，如果这些任务放在主线程中去执行就会造成界面无法响应，严重影响用户体验，这种情况在PC系统和移动系统中都存在，在Android中有一个特殊的名字叫做ANR（Application Not Responding），即应用无响应。解决这个问题就需要用到线程，把一些耗时的任务放在线程中即可。

IPC不是Android中所独有的，任何一个操作系统都需要有相应的IPC机制，比如Windows上可以通过剪贴板、管道和邮槽等来进行进程间通信；Linux上可以通过命名管道、共享内容、信号量等来进行进程间通信。可以看到不同的操作系统平台有着不同的进程间通信方式，对于Android来说，它是一种基于Linux内核的移动操作系统，它的进程间通信方式并不能完全继承自Linux，相反，它有自己的进程间通信方式。在Android中最有特色的进程间通信方式就是Binder了，通过Binder可以轻松地实现进程间通信。除了Binder，Android还支持Socket，通过Socket也可以实现任意两个终端之间的通信，当然同一个设备上的两个进程通过Socket通信自然也是可以的。

说到IPC的使用场景就必须提到多进程，只有面对多进程这种场景下，才需要考虑进程间通信。这个是很好理解的，如果只有一个进程在运行，又何谈多进程呢？多进程的情况分为两种。第一种情况是一个应用因为某些原因自身需要采用多进程模式来实现，至于原因，可能有很多，比如有些模块由于特殊原因需要运行在单独的进程中，又或者为了加大一个应用可使用的内存所以需要通过多进程来获取多份内存空间。Android对单个应用所使用的最大内存做了限制，早期的一些版本可能是16MB，不同设备有不同的大小。另一种情况是当前应用需要向其他应用获取数据，由于是两个应用，所以必须采用跨进程的方式来获取所需的数据，甚至我们通过系统提供的ContentProvider去查询数据的时候，其实也是一种进程间通信，只不过通信细节被系统内部屏蔽了，我们无法感知而已。后续章节会详细介绍ContentProvider的底层实现，这里就先不做详细介绍了。总之，不管由于何种原因，我们采用了多进程的设计方法，那么应用中就必须妥善地处理进程间通信的各种问题。

## 二、Android 中的多进程模式

在正式介绍进程间通信之前，我们必须先要理解Android中的多进程模式。通过给四大组件指定
<font color=red>android:process</font>属性，我们可以轻易地开启多进程模式，这看起来很简单，但是实际使用过程中却暗藏杀机，多进程远远没有我们想的那么简单，有时候我们通过多进程得到的好处甚至都不足以弥补使用多进程所带来的代码层面的负面影响。下面会详细分析这些问题。

### 1、开启多进程模式

正常情况下，在Android中多进程是指一个应用中存在多个进程的情况，因此这里不讨论两个应用之间的多进程情况。首先，在Android中使用多进程只有一种方法，那就是给四大组件（Activity、Service、Receiver、ContentProvider）在AndroidMenifest 中指定android:process属性，除此之外没有其他办法，也就是说我们无法给一个线程或者一个实体类指定其运行时所在的进程。其实还有另一种非常规的多进程方法，那就是通过JNI在native层去fork一个新的进程，但是这种方法属于特殊情况，也不是常用的创建多进程的方式，因此我们暂时不考虑这种方式。下面是一个示例，描述了如何在Android中创建多进程：
``` xml
        <activity
            android:name=".MainActivity"
            android:configChanges="orientation|screenSize"
            android:label="@string/app_name"
            android:launchMode="standard">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity
            android:name=".SecondActivity"
            android:configChanges="screenLayout"
            android:label="@string/app_name"
            android:process=":remote" />
        <activity
            android:name=".ThirdActivity"
            android:configChanges="screenLayout"
            android:label="@string/app_name"
            android:process="com.ryg.chapter_2.remote" />
```

上面的示例分别为SecondActivity和ThirdActivity指定了process属性，并且它们的属性值不同，这意味着当前应用又增加了两个新进程。假设当前应用的包名为“com.ryg.chapter2”，当SecondActivity启动时，系统会为它创建一个单独的进程，进程名为
“com.ryg.chapter 2：remote”；当ThirdActivity启动时，系统也会为它创建一个单独的进程，进程名为“com.ryg.chapter_2.remote”。同时入口Activity是MainActivity，没有为它指定process属性，那么它运行在默认进程中，默认进程的进程名是包名。下面我们运行一下看看效果，如下图。进程列表末尾存在3个进程，进程id分别为3039、3056、3098，这说明我们的应用成功地使用了多进程技术，是不是很简单呢？这只是开始，实际使用中多进程是有很多问题需要处理的。

![](https://sweetm-1256061026.cos.ap-beijing.myqcloud.com/blog/20190321165817.png)

使用 adb shell 来查看进程信息

``` shell
vbox86p:/ # ps | grep com.ryg.chapter_2
u0_a65    3039  249   856408 102248    ep_poll e8b89bb9 S com.ryg.chapter_2
u0_a65    3056  249   837576 85156    ep_poll e8b89bb9 S com.ryg.chapter_2:remote
u0_a65    3098  249   836072 82460    ep_poll e8b89bb9 S com.ryg.chapter_2.remote
```
不知道读者朋友有没有注意到，SecondActivity和ThirdActivity的android:process属性分别为“：remote”和“com.ryg.chapter_2.remote”，那么这两种方式有区别吗？其实是有区别的，区别有两方面：首先，<font color=red>“：”</font>的含义是指要在当前的进程名前面附加上当前的包名，这是一种简写的方法，对于SecondActivity来说，它完整的进程名为com.ryg.chapter_2：remote，这可以从上面图片的进程信息也能看出来，而对于ThirdActivity中的声明方式，它是一种完整的命名方式，不会附加包名信息；其次，<font red=blue>进程名以“：”开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中</font>，而进程名不以“：”开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

我们知道Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。这里要说明的是，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，比如data目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程中，那么除了能共享data目录、组件信息，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

### 2、多进程模式的运行机制

如果用一句话来形容多进程，那笔者只能这样说：“当应用开启了多进程以后，各种奇怪的现象都出现了”。为什么这么说呢？这是有原因的。大部分人都认为开启多进程是很简单的事情，只需要给四大组件指定androidprocess属性即可。比如说在实际的产品开发中，可能会有多进程的需求，需要把某些组件放在单独的进程中去运行，很多人都会觉得这不很简单吗？
然后迅速地给那些组件指定了android:process属性，然后编译运行，发现“正常地运行起来了”。
这里笔者想说的是，那是真的正常地运行起来了吗？现在先不置可否，下面先给举个例子，然后引入本节的话题。还是本章刚开始说的那个例子，其中SecondActivity通过指定android:process属性从而使其运行在一个独立的进程中，这里做了一些改动，我们新建了一个类，叫做UserManager，这个类中有一个public的静态成员变量，如下所示。

``` java
public class UserManager {
    public static int sUserId = 1;
}
```

然后在MainActivity的onCreate中我们把这个sUserld重新赋值为2，打印出这个静态变量的值后再启动SecondActivity，在SecondActivity中我们再打印一下sUserld的值。按照正常的逻辑，静态变量是可以在所有的地方共享的，并且一处有修改处处都会同步，下面是运行时所打印的日志，我们看一下结果如何。

```java
2019-03-21 20:53:47.451 1682-1682/com.ryg.chapter_2 D/MainActivity: UserManage.sUserId=2
2019-03-21 20:56:03.063 1852-1852/com.ryg.chapter_2:remote D/SecondActivity: UserManage.sUserId=1
```



上述问题出现的原因是SecondActivity运行在一个单独的进程中，我们知道Android为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。拿我们这个例子来说，在进程com.ryg.chapter_2和进程com.ryg.chapter_2：remote中都存在一个UserManager类，并且这两个类是互不干扰的，在一个进程中修改sUserld的值只会影响当前进程，对其他进程不会造成任何影响，这样我们就可以理解为什么在MainActivity中修改了sUserld的值，但是在SecondActivity中sUserld的值却没有发生改变这个现象。

所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败，这也是多进程所带来的主要影响。正常情况下，四大组件中间不可能不通过一些中间层来共享数据，那么通过简单地指定进程名来开启多进程都会无法正确运行。当然，特殊情况下，某些组件之间不需要共享数据，这个时候可以直接指定android:process属性来开启多进程，但是这种场景是不常见的，几乎所有情况都需要共享数据。

**一般来说，使用多进程会造成如下几方面的问题：**
（1）静态成员和单例模式完全失效。
（2）线程同步机制完全失效。
（3）SharedPreferences的可靠性下降。
（4）Application会多次创建。

**第1个问题**在上面已经进行了分析。

**第2个问题**本质上和第一个问题是类似的，既然都不是一块内存了，那么不管是锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象。

**第3个问题**是因为SharedPreferences不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这是因为SharedPreferences底层是通过读/写XML文件来实现的，并发写显然是可能出问题的，甚至并发读/写都有可能出问题。

**第4个问题**也是显而易见的，当一个组件跑在一个新的进程中的时候，由于系统要在创建新的进程同时分配独立的虚拟机，所以这个过程其实就是启动一个应用的过程。因此，相当于系统又把这个应用重新启动了一遍，既然重新启动了，那么自然会创建新的Application。这个问题其实可以这么理解，运行在同一个进程中的组件是属于同一个虚拟机和同一个Application的，同理，运行在不同进程中的组件是属于两个不同的虚拟机和Application的。为了更加清晰地展示这一点，下面我们来做一个测试，首先在Application的onCreate方法中打印出当前进程的名字，然后连续启动三个同一个应用内但属于不同进程的Activity，按照期望，Application的onCreate应该执行三次并打印出三次进程名不同的log，代码如下所示。

```java
2019-03-21 21:06:47.581 2024-2024/com.ryg.chapter_2 D/MyApplication: application start, process name:com.ryg.chapter_2
2019-03-21 21:06:47.702 2041-2041/com.ryg.chapter_2:remote D/MyApplication: application start, process name:com.ryg.chapter_2:remote
2019-03-21 21:07:01.798 2079-2079/? D/MyApplication: application start, process name:com.ryg.chapter_2.remote
```



## 三、IPC基本概念介绍

主要介绍IPC中的一些基础概念，主要包含三方面内容：<u>Serializable接口、Parcelable 接口以及Binder</u>，只有熟悉这三方面的内容后，我们才能更好地理解跨进程通信的各种方式。Serializable和Parcelable接口可以完成对象的序列化过程，当我们需要通过Intent和Binder传输数据时就需要使用Parcelable 或者Serializable。还有的时候我们需要把对象持久化到存储设备上或者通过网络传输给其他客户端，这个时候也需要使用Serializable 来完成对象的持久化，下面先介绍如何使用Serializable来完成对象的序列化。

### 1、Serializable 接口

Serializable是Java所提供的一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。使用Serializable来实现序列化相当简单，只需要在类的声明中指定一个类似下面的标识即可自动实现默认的序列化过程。

``` java
    private static final long serialVersionUID = 519067123721295773L;
```
在Android中也提供了新的序列化方式，那就是Parcelable接口，使用Parcelable来实现对象的序列号，其过程要稍微复杂一些，本节先介绍Serializable接口。上面提到，想让一个对象实现序列化，只需要这个类实现Serializable接口并声明一个serial VersionUID即可，实际上，甚至这个serialVersionUID也不是必需的，我们不声明这个serialVersionUID同样也可以实现序列化，但是这将会对反序列化过程产生影响，具体什么影响后面再介绍。
User类就是一个实现了Serializable接口的类，它是可以被序列化和反序列化的，如下所示。

``` java
public class User implements Serializable {
    private static final long serialVersionUID = 519067123721295773L;
    public int userId;
    public String userName;
    public boolean isMale;
    public Book book;
    ...
}
```
通过Serializable方式来实现对象的序列化，实现起来非常简单，几乎所有工作都被系统自动完成了。如何进行对象的序列化和反序列化也非常简单，只需要采用ObjectOutputStream和ObjectinputStream即可轻松实现。下面举个简单的例子。

``` java
        //序列化
        User user = new User(0, "jake", true);
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cache.txt"));
        out.writeObject(user);
        out.close();

        //反序列化
        ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt"));
        User newUser = (User) in.readObject();
        in.close();
```

上述代码演示了采用Serializable方式序列化对象的典型过程，很简单，只需要把实现了Serializable接口的User对象写到文件中就可以快速恢复了，恢复后的对象newUser和user的内容完全一样，但是两者并不是同一个对象。

刚开始提到，即使不指定serialVersionUID也可以实现序列化，那到底要不要指定呢？如果指定的话，serialVersionUID后面那一长串数字又是什么含义呢？我们要明白，系统既然提供了这个serialVersionUID，那么它必须是有用的。这个serialVersionUID是用来辅助序列化和反序列化过程的，<font color=blue>原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID相同才能够正常地被反序列化。</font>。serialVersionUID的详细工作机制是这样的：序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也可能是其他中介），当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的类的版本和当前类的版本是相同的，这个时候可以成功反序列化；否则就说明当前类和序列化的类相比发生了某些变换，比如成员变量的数量、类型可能发生了改变，这个时候是无法正常反序列化的，因此会报如下错误：

```java
java.io.InvalidClassException: Main; local class incompatible: streamclassdesc serialVersionUID=8711368828010083044, local class serial-VersionUID=8711368828010083043.
```
一般来说，我们应该手动指定serialVersionUID的值，比如1L，也可以让Eclipse根据当前类的结构自动去生成它的hash值，这样序列化和反序列化时两者的serialVersionUID是相同的，因此可以正常进行反序列化。如果不手动指定serialVersionUID的值，反序列化时当前类有所改变，比如增加或者删除了某些成员变量，那么系统就会重新计算当前类的hash 值并把它赋值给serialVersionUID，这个时候当前类的serialVersionUID就和序列化的数据中的serialVersionUID不一致，于是反序列化失败，程序就会出现crash。所以，我们可以明显感觉到serialVersionUID的作用，当我们手动指定了它以后，就可以在很大程度上避免反序列化过程的失败。比如当版本升级后，我们可能删除了某个成员变量也可能增加了一些新的成员变量，这个时候我们的反向序列化过程仍然能够成功，程序仍然能够最大限度地恢复数据，相反，如果不指定serialVersionUID的话，程序则会挂掉。当然我们还要考虑另外一种情况，如果类结构发生了非常规性改变，比如修改了类名，修改了成员变量的类型，这个时候尽管serialVersionUID验证通过了，但是反序列化过程还是会失败，因为类结构有了毁灭性的改变，根本无法从老版本的数据中还原出一个新的类结构的对象。

根据上面的分析，我们可以知道，给serialVersionUID指定为1L或者采用Eclipse根据当前类结构去生成的hash值，这两者并没有本质区别，效果完全一样。以下两点需要特别提一下，首先静态成员变量属于类不属于对象，所以不会参与序列化过程；其次用**transient**关键字标记的成员变量不参与序列化过程。

### 2、Parcelable接口

通过Serializable方式来实现序列化的方法，本节接着介绍另一种序列化方式：Parcelable。Parcelable也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过Intent和Binder传递。下面的示例是一个典型的用法。

``` java
package com.ryg.chapter_2.model;

import java.io.Serializable;

import com.ryg.chapter_2.aidl.Book;

import android.os.Parcel;
import android.os.Parcelable;

public class User implements Parcelable, Serializable {
    private static final long serialVersionUID = 519067123721295773L;

    public int userId;
    public String userName;
    public boolean isMale;
    public Book book;
    public User() { }
    public User(int userId, String userName, boolean isMale) {
        this.userId = userId;
        this.userName = userName;
        this.isMale = isMale;
    }
    @Override
    public int describeContents() {
        return 0;
    }
    @Override
    public void writeToParcel(Parcel out, int flags) {
        out.writeInt(userId);
        out.writeString(userName);
        out.writeInt(isMale ? 1 : 0);
        out.writeParcelable(book, 0);
    }
    public static final Parcelable.Creator<User> CREATOR = new Parcelable.Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
    private User(Parcel in) {
        userId = in.readInt();
        userName = in.readString();
        isMale = in.readInt() == 1;
        book = in.readParcelable(Thread.currentThread().getContextClassLoader());
    }

    @Override
    public String toString() {
        return String.format(
                "User:{userId:%s, userName:%s, isMale:%s}, with child:{%s}",
                userId, userName, isMale, book);
    }
}
```

这里先说一下Parcel，Parcel内部包装了可序列化的数据，可以在Binder中自由传输。从上述代码中可以看出，在序列化过程中需要实现的功能有序列化、反序列化和内容描述。序列化功能由**writeToParcel**方法来完成，最终是通过Parcel中的一系列write方法来完成的；反序列化功能由CREATOR来完成，其内部标明了如何创建序列化对象和数组，并通过Parcel的一系列read方法来完成反序列化过程；内容描述功能由**describeContents**方法来完成，几乎在所有情况下这个方法都应该返回0，仅当当前对象中存在文件描述符时，此方法返回1。需要注意的是，在User（Parcelin）方法中，由于book是另一个可序列化对象，所以它的反序列化过程需要传递当前线程的上下文类加载器，否则会报无法找到类的错误。
详细的方法说明请参看表2-1。

|                 方法                 |                             功能                             |            标记位             |
| :----------------------------------: | :----------------------------------------------------------: | :---------------------------: |
|     createFromParcel(Parcel in)      |                从序列化后的对象中创建原始对象                |                               |
|          newArray(int size)          |                  创建指定长度的原始对象数组                  |                               |
|           User(Parcel in)            |                从序列化后的对象中创建原始对象                |                               |
| writeToParcel(Parcel  out,int flags) | 将当前对象写入序列化结构中，其中flags标识有两种值：0或者1（参见右侧标记位）。为l时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况都为0 | PARCELABLE_WRITE_RETURN_VALUE |
|           describeContents           | 返回当前对象的内容描述。如果含有文件描述符，返回 1 （参见右侧标记位），否则返回0，几乎所有情况都返回0 |  CONTENTS _FILE _DESCRIPTOR   |

系统已经为我们提供了许多实现了Parcelable接口的类，它们都是可以直接序列化的，比如Intent、Bundle、Bitmap等，同时List和Map也可以序列化，前提是它们里面的每个元素都是可序列化的。

既然Parcelable和Serializable都能实现序列化并且都可用于Intent间的数据传递，那么二者该如何选取呢？Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列化过程需要大量I/O操作。而Parcelable是Android中的序列化方式，因此更适合用在Android平台上，它的缺点就是使用起来稍微麻烦点，但是它的效率很高，这是Android推荐的序列化方式，因此我们要首选Parcelable。Parcelable主要用在内存序列化上，通过Parcelable将对象序列化到存储设备中或者将对象序列化后通过网络传输也都是可以的，但是这个过程会稍显复杂，因此在这两种情况下建议大家使用Serializable。以上就是Parcelable和Serializable的区别。

https://blog.csdn.net/javazejian/article/details/52665164



### 3、Binder

Binder是一个很深入的话题，笔者也看过一些别人写的Binder相关的文章，发现很少有人能把它介绍清楚，不是深入代码细节不能自拔，就是长篇大论不知所云，看完后都是晕晕的感觉。所以，本节笔者不打算深入探讨Binder的底层细节，因为Binder太复杂了。

直观来说，Binder是Android中的一个类，它实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是**/dev/binder**，该通信方式在Linux中没有；从Android Framework角度来说，Binder 是ServiceManager连接各种Manager（ActivityManager、WindowManager，等等）和相应ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

Android 开发中，Binder主要用在Service中，包括AIDL和Messenger，其中普通 Service中的Binder不涉及进程间通信，所以较为简单，无法触及Binder的核心，而Messenger的底层其实是AIDL，所以这里选择用AIDL来分析Binder的工作机制。为了分析Binder的工作机制，我们需要新建一个AIDL示例，SDK会自动为我们生产AIDL所对应的Binder类，然后我们就可以分析Binder的工作过程。还是采用本章开始时用的例子，新建Java包com.ryg.chapter_2.aidl，然后新建三个文件Book.java、Book.aidl和IBookManager.aidl，代码如下所示。

``` java
public class Book implements Parcelable {

    public int bookId;
    public String bookName;
    public Book() {}
    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }
    public int describeContents() {
        return 0;
    }
    public void writeToParcel(Parcel out, int flags) {
        out.writeInt(bookId);
        out.writeString(bookName);
    }
    public static final Parcelable.Creator<Book> CREATOR = new Parcelable.Creator<Book>() {
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
    private Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }
    @Override
    public String toString() {
        return String.format("[bookId:%s, bookName:%s]", bookId, bookName);
    }
}
```
#### AIDL 文件

``` java
//Book.aidl
package com.ryg.chapter_2.aidl;
parcelable Book;

//IBookManager.aidl
package com.ryg.chapter_2.aidl;
import com.ryg.chapter_2.aidl.Book;
interface IBookManager {
     List<Book> getBookList();
     void addBook(in Book book);
}
```



上面三个文件中，Book.java是一个表示图书信息的类，它实现了Parcelable接口。Book.aidl 是Book类在AIDL中的声明。IBookManageraidl是我们定义的一个接口，里面有两个方法：getBookList和addBook，其getBookList用于从远程服务端获取图书列表，而addBook用于往图书列表中添加一本书，当然这两个方法主要是示例用，不一定要有实际意义。我们可以看到，尽管Book类已经和IBookManager位于相同的包中，但是在IBookManager 中仍然要导入Book类，这就是AIDL的特殊之处。下面我们先看一下系统为IBookManageraidl生产的Binder类，在gen目录下的com.ryg.chapter_2.aidl包中有一个IBookManager.java的类，这就是我们要找的类。接下来我们需要根据这个系统生成的Binder类来分析Binder的工作原理，代码如下：

aidl 编译生成的代码：

```java
package com.ryg.chapter_2.aidl;
public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.ryg.chapter_2.aidl.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.ryg.chapter_2.aidl.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.ryg.chapter_2.aidl.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.ryg.chapter_2.aidl.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.ryg.chapter_2.aidl.IBookManager))) {
                return ((com.ryg.chapter_2.aidl.IBookManager) iin);
            }
            return new com.ryg.chapter_2.aidl.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    java.util.List<com.ryg.chapter_2.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(descriptor);
                    com.ryg.chapter_2.aidl.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.ryg.chapter_2.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements com.ryg.chapter_2.aidl.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.util.List<com.ryg.chapter_2.aidl.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.ryg.chapter_2.aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.ryg.chapter_2.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.ryg.chapter_2.aidl.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }
		//标识方法的 整型变量
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.util.List<com.ryg.chapter_2.aidl.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.ryg.chapter_2.aidl.Book book) throws android.os.RemoteException;
}

```



上述代码是系统生成的，为了方便查看笔者稍微做了一下格式上的调整。在gen目录下，可以看到根据IBookManager.aidl系统为我们生成了IBookManager.java这个类，它继承了IInterface这个接口，同时它自己也还是个接口，所有可以在Binder中传输的接口都需要继承Interface接口。这个类刚开始看起来逻辑混乱，但是实际上还是很清晰的，通过它我们可以清楚地了解到Binder的工作机制。这个类的结构其实很简单，首先，它声明了两个方法getBookList和addBook，显然这就是我们在IBookManageraidl中所声明的方法，同时它还声明了两个整型的id分别用于标识这两个方法，这两个id用于标识在**transact**过程中客户端所请求的到底是哪个方法。接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub的内部代理类Proxy来完成。这么来看，IBookManager这个接口的确很简单，但是我们也应该认识到，这个接口的核心实现就是它的内部类Stub和Stub的内部代理类Proxy，下面详细介绍针对这两个类的每个方法的含义。

##### DESCRIPTOR

Binder的唯一标识，一般用当前Binder的类名表示，比如本例中的 “com.ryg.chapter_2.aidl.IBookManager”。

asInterface(android.os.IBinder obj)

用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象。

##### asBinder

此方法用于返回当前Binder对象。

##### onTransact

这个方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。该方法的原型为public Boolean onTransact（int code，android.os.Parcel data，android.os.Parcel reply，int flags）。服务端通过code可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需的参数（如果目标方法有参数的话），然后执行目标方法。当目标方法执行完毕后，就向reply中写入返回值（如果目标方法有返回值的话），onTransact方法的执行过程就是这样的。需要注意的是，如果此方法返回false，那么客户端的请求会失败，因此我们可以利用这个**特性来做权限验证**，毕竟我们也不希望随便一个进程都能远程调用我们的服务。

##### Proxy#getBookList

这个方法运行在客户端，当客户端远程调用此方法时，它的内部实现是这样的：首先创建该方法所需要的输入型Parcel对象_data、输出型Parcel对象\_reply和返回值对象List；然后把该方法的参数信息写入\_data中（如果有参数的话）；接着调用**transact**方法来发起RPC（远程过程调用）请求，同时当前线程挂起；然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从reply中取出RPC过程的返回结果；最后返回\_reply中的数据。

##### Proxy#addBook
这个方法运行在客户端，它的执行过程和getBookList是一样的，addBook没有返回值，所以它不需要从\_reply中取出返回值。

通过上面的分析，读者应该已经了解了Binder的工作机制，但是有两点还是需要额外说明一下：首先，当客户端发起远程请求时，由于当前线程会**被挂起直至服务端进程返回数据**，所以如果一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求；其次，由于服务端的Binder方法运行在Binder的线程池中，**所以Binder方法不管是否耗时都应该采用同步的方式去实现，因为它已经运行在一个线程中了**。为了更好地说明Binder，下面给出一个Binder的工作机制图，如图。

![](https://sweetm-1256061026.cos.ap-beijing.myqcloud.com/blog/20190322000702.png)

#### 手写Binder

从上述分析过程来看，我们完全可以不提供AIDL文件即可实现Binder，之所以提供AIDL文件，是为了方便系统为我们生成代码。系统根据AIDL文件生成Java文件的格式是固定的，我们可以抛开AIDL文件直接写一个Binder出来，接下来我们就介绍如何手动写一个Binder。还是上面的例子，但是这次我们不提供AIDL文件。参考上面系统自动生成的IBookManager.java这个类的代码，可以发现这个类是相当有规律的，根据它的特点，我们完全可以自己写一个和它一模一样的类出来，然后这个不借助AIDL文件的Binder就完成了。但是我们发现系统生成的类看起来结构不清晰，我们想试着对它进行结构上的调整，可以发现这个类主要由两部分组成，首先它本身是一个**Binder的接口**（继承了IInterface），其次它的内部由个Stub类，这个类就是个Binder。还记得我们怎么写一个Binder的服务端吗？代码如下所示。

```java
    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };

```

首先我们会实现一个创建了一个Stub对象并在内部实现IBookManager的接口方法，然后在Service的onBind中返回这个Stub对象。因此，从这一点来看，我们完全可以把Stub类提取出来直接作为一个独立的Binder类来实现，这样IBookManager中就只剩接口本身了，通过这种分离的方式可以让它的结构变得清晰点。

根据上面的思想，手动实现一个Binder可以通过如下步骤来完成：

（1）声明一个AIDL性质的接口，只需要继承IInterface 接口即可，IInterface接口中只有一个asBinder方法。这个接口的实现如下：

```java
public interface IBookManager extends IInterface {
    static final String DESCRIPTOR = "com.ryg.chapter_2.manualbinder.IBookManager";
    static final int TRANSACTION_getBookList = (IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_addBook = (IBinder.FIRST_CALL_TRANSACTION + 1);
    public List<Book> getBookList() throws RemoteException;
    public void addBook(Book book) throws RemoteException;
}
```

可以看到，在接口中声明了一个Binder描述符和另外两个id，这两个id分别表示的是getBookList和addBook方法，这段代码原本也是系统生成的，我们仿照系统生成的规则去手动书写这部分代码。如果我们有三个方法，应该怎么做呢？很显然，我们要再声明一个id，然后按照固定模式声明这个新方法即可，这个比较好理解，不再多说。
（2）实现Stub类和Stub类中的Proxy代理类，这段代码我们可以自己写，但是写出来后会发现和系统自动生成的代码是一样的，因此这个Stub类我们只需要参考系统生成的代码即可，只是结构上需要做一下调整，调整后的代码如下所示。

```java
public class BookManagerImpl extends Binder implements IBookManager {

    /** Construct the stub at attach it to the interface. */
    public BookManagerImpl() {
        this.attachInterface(this, DESCRIPTOR);
    }

    /**
     * Cast an IBinder object into an IBookManager interface, generating a proxy
     * if needed.
     */
    public static IBookManager asInterface(IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IBookManager))) {
            return ((IBookManager) iin);
        }
        return new BookManagerImpl.Proxy(obj);
    }

    @Override
    public IBinder asBinder() {
        return this;
    }

    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case INTERFACE_TRANSACTION: {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_getBookList: {
            data.enforceInterface(DESCRIPTOR);
            List<Book> result = this.getBookList();
            reply.writeNoException();
            reply.writeTypedList(result);
            return true;
        }
        case TRANSACTION_addBook: {
            data.enforceInterface(DESCRIPTOR);
            Book arg0;
            if ((0 != data.readInt())) {
                arg0 = Book.CREATOR.createFromParcel(data);
            } else {
                arg0 = null;
            }
            this.addBook(arg0);
            reply.writeNoException();
            return true;
        }
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public List<Book> getBookList() throws RemoteException {
        // TODO 待实现
        return null;
    }

    @Override
    public void addBook(Book book) throws RemoteException {
        // TODO 待实现
    }

    private static class Proxy implements IBookManager {
        private IBinder mRemote;

        Proxy(IBinder remote) {
            mRemote = remote;
        }

        @Override
        public IBinder asBinder() {
            return mRemote;
        }

        public java.lang.String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }

        @Override
        public List<Book> getBookList() throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            List<Book> result;
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                mRemote.transact(TRANSACTION_getBookList, data, reply, 0);
                reply.readException();
                result = reply.createTypedArrayList(Book.CREATOR);
            } finally {
                reply.recycle();
                data.recycle();
            }
            return result;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                if ((book != null)) {
                    data.writeInt(1);
                    book.writeToParcel(data, 0);
                } else {
                    data.writeInt(0);
                }
                mRemote.transact(TRANSACTION_addBook, data, reply, 0);
                reply.readException();
            } finally {
                reply.recycle();
                data.recycle();
            }
        }
    }

}
```

通过将上述代码和系统生成的代码对比，可以发现简直是一模一样的。也许有人会问：既然和系统生成的一模一样，那我们为什么要手动去写呢？我们在实际开发中完全可以通过AIDL文件让系统去自动生成，手动去写的意义在于可以让我们更加理解Binder的工作原理，同时也提供了一种不通过AIDL文件来实现Binder的新方式。也就是说，AIDL文件并不是实现Binder的必需品。**如果是我们手写的Binder，那么在服务端只需要创建一个BookManagerlmpl的对象并在Service的onBind方法中返回即可**。最后，是否手动实现Binder没有本质区别，二者的工作原理完全一样，AIDL文件的本质是系统为我们提供了一种快速实现Binder的工具，仅此而已。



#### 设置死亡代理

接下来，我们介绍Binder的两个很重要的方法**linkToDeath**和**unlinkToDeath**。我们知道，Binder运行在服务端进程，如果服务端进程由于某种原因异常终止，这个时候我们到服务端的Binder连接断裂（称之为Binder死亡），会导致我们的远程调用失败。更为关键的是，如果我们不知道Binder连接已经断裂，那么客户端的功能就会受到影响。为了解决这个问题，Binder中提供了两个配对的方法linkToDeath和unlinkToDeath，通过linkToDeath我们可以给Binder**设置一个死亡代理**，当Binder死亡时，我们就会收到通知，这个时候我们就可以重新发起连接请求从而**恢复连接**。那么到底如何给Binder设置死亡代理呢？也很简单。

首先，声明一个DeathRecipient对象。DeathRecipient是一个接口，其内部只有一个方法binderDied，我们需要实现这个方法，当Binder死亡的时候，系统就会回调binderDied方法，然后我们就可以移出之前绑定的binder代理并重新绑定远程服务：

```java
    private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            if (mBookManager == null)
                return;
            mBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
            mBookManager = null;
            // TODO:这里重新绑定远程Service
        }
    };
```

其次，在客户端绑定远程服务成功后，给binder设置死亡代理：

```java
     IBookManager bookManager = IBookManager.Stub.asInterface(service);
	 //绑定链接断开的对象
     bookManager.asBinder().linkToDeath(mDeathRecipient, 0);
```

其中linkToDeath的第二个参数是个标记位，我们直接设为0即可。经过上面两个步骤，就给我们的Binder设置了死亡代理，当Binder死亡的时候我们就可以收到通知了。另外，通过Binder的方法isBinderAlive也可以判断Binder是否死亡。
到这里，IPC的基础知识就介绍完毕了，下面开始进入正题，直面形形色色的进程间通信方式。

## 四、Android中的IPC方式

### 4.1 使用Bundle

我们知道，四大组件中的三大组件（Activity、Service、Receiver）都是支持在Intent中传递Bundle数据的，由于Bundle实现了Parcelable接口，所以它可以方便地在不同的进程间传输。基于这一点，当我们在一个进程中启动了另一个进程的Activity、Service和Receiver，我们就可以在Bundle中附加我们需要传输给远程进程的信息并通过Intent发送出去。当然，我们传输的数据必须能够被序列化，比如基本类型、实现了Parcellable接口的对象、实现了Serializable接口的对象以及一些Android支持的特殊对象，具体内容可以看Bundle这个类，就可以看到所有它支持的类型。Bundle不支持的类型我们无法通过它在进程间传递数据，这个很简单，就不再详细介绍了。这是一种最简单的进程间通信方式。

除了直接传递数据这种典型的使用场景，它还有一种特殊的使用场景。比如A进程正在进行一个计算，计算完成后它要启动B进程的一个组件并把计算结果传递给B进程，可是遗憾的是这个计算结果不支持放入Bundle中，因此无法通过Intent来传输，这个时候如果我们用其他IPC方式就会略显复杂。可以考虑如下方式：我们通过Intent启动进程B的一个Service组件（比如IntentService），让Service在后台进行计算，计算完毕后再启动B进程中真正要启动的目标组件，由于Service也运行在B进程中，所以目标组件就可以直接获取计算结果，这样一来就轻松解决了跨进程的问题。这种方式的核心思想在于将原本需要在A进程的计算任务转移到B进程的后台Service中去执行，这样就成功地避免了进程间通信问题，而且只用了很小的代价。

### 4.2 使用文件共享
共享文件也是一种不错的进程间通信方式，两个进程通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。我们知道，在Windows上，一个文件如果被加了排斥锁将会导致其他线程无法对其进行访问，包括读和写，而由于Android系统基于Linux，使得其并发读/写文件可以没有限制地进行，甚至两个线程同时对同一个文件进行写操作都是允许的，尽管这可能出问题。通过文件交换数据很好使用，除了可以交换一些文本信息外，我们还可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象，下面就展示这种使用方法。

这次我们在MainActivity的onResume中序列化一个User对象到sd卡上的一个文件里，然后在SecondActivity的onResume中去反序列化，我们期望在SecondActivity中能够正确地恢复User对象的值。关键代码如下：
``` java
// MainActivity代码
    private void persistToFile() {
        new Thread(new Runnable() {

            @Override
            public void run() {
                User user = new User(1, "hello world", false);
                File dir = new File(MyConstants.CHAPTER_2_PATH);
                if (!dir.exists()) {
                    dir.mkdirs();
                }
                File cachedFile = new File(MyConstants.CACHE_FILE_PATH);
                ObjectOutputStream objectOutputStream = null;
                try {
                    objectOutputStream = new ObjectOutputStream(
                            new FileOutputStream(cachedFile));
                    objectOutputStream.writeObject(user);
                    Log.d(TAG, "persist user:" + user);
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    MyUtils.close(objectOutputStream);
                }
            }
        }).start();
    }

 //SecondActivity代码
    private void recoverFromFile() {
        new Thread(new Runnable() {

            @Override
            public void run() {
                User user = null;
                File cachedFile = new File(MyConstants.CACHE_FILE_PATH);
                if (cachedFile.exists()) {
                    ObjectInputStream objectInputStream = null;
                    try {
                        objectInputStream = new ObjectInputStream(
                                new FileInputStream(cachedFile));
                        user = (User) objectInputStream.readObject();
                        Log.d(TAG, "recover user:" + user);
                    } catch (IOException e) {
                        e.printStackTrace();
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    } finally {
                        MyUtils.close(objectInputStream);
                    }
                }
            }
        }).start();
    }

```

通过文件共享这种方式来共享数据对文件格式是没有具体要求的，比如可以是文本文件，也可以是XML文件，只要读/写双方约定数据格式即可。通过文件共享的方式也是有局限性的，比如并发读/写的问题，像上面的那个例子，如果**并发读/写**，那么我们读出的内容就有可能不是最新的，如果是并发写的话那就更严重了。因此我们要尽量避免并发写这种情况的发生或者考虑使用线程同步来限制多个线程的写操作。通过上面的分析，我们可以知道，文件共享方式适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读/写的问题。

当然，SharedPreferences是个特例，众所周知，SharedPreferences是Android中提供的轻量级存储方案，它通过键值对的方式来存储数据，在底层实现上它采用XML文件来存储键值对，每个应用的SharedPreferences文件都可以在当前包所在的data目录下查看到。一般来说，它的目录位于/data/data/package name/sharedprefs 目录下，其中package name表示的是当前应用的包名。从本质上来说，SharedPreferences也属于文件的一种，但是由于系统对它的读/写有一定的缓存策略，**即在内存中会有一份SharedPreferences文件的缓存**，因此在多进程模式下，系统对它的读/写就变得不可靠，当面对高并发的读/写访问，Sharedpreferences有很大几率会丢失数据，因此，不建议在进程间通信中使用SharedPreferences。

### 4.3 使用Messenger

Messenger可以翻译为信使，顾名思义，通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它的底层实现是AIDL，为什么这么说呢，我们大致看一下Messenger这个类的构造方法就明白了。下面是Messenger的两个构造方法，从构造方法的实现上我们可以明显看出AIDL的痕迹，不管是IMessenger还是Stub.aslnterface，这种使用方法都表明它的底层是AIDL。

``` java
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
```
Messenger的使用方法很简单，它对AIDL做了封装，使得我们可以更简便地进行进程间通信。同时，由于它一次处理一个请求，因此在服务端我们不用考虑线程同步的问题，这是因为服务端中不存在并发执行的情形。实现一个Messenger有如下几个步骤，分为服务端和客户端。

#### 1.服务端进程

首先，我们需要在服务端创建一个Service来处理客户端的连接请求，同时创建一个 Handler 并通过它来创建一个Messenger对象，然后在Service的onBind中返回这个Messenger对象底层的Binder即可。

#### 2.客户端进程

客户端进程中，首先要绑定服务端的Service，绑定成功后用服务端返回的IBinder对象创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，发消息类型为Message对象。如果需要服务端能够回应客户端，就和服务端一样，我们还需要创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。这听起来可能还是有点抽象，不过看了下面的两个例子，读者肯定就都明白了。首先，我们来看一个简单点的例子，在这个例子中服务端无法回应客户端。

首先看服务端的代码，这是服务端的典型代码，可以看到MessengerHandler用来处理客户端发送的消息，并从消息中取出客户端发来的文本信息。而mMessenger是一个Messenger对象，它和MessengerHandler相关联，并在onBind方法中返回它里面的Binder对象，可以看出，这里Mesenger的作用是将客户端发送的消息传递给MessengerHandler处理。

``` java
public class MessengerService extends Service {
    private static final String TAG = "MessengerService";
    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case MyConstants.MSG_FROM_CLIENT:
                Log.i(TAG, "receive msg from Client:" + msg.getData().getString("msg"));
            default:
                super.handleMessage(msg);
            }
        }
    }
    private final Messenger mMessenger = new Messenger(new MessengerHandler());
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```
然后，注册 service，让其运行在单独的进程中：
``` xml
        <service
            android:name=".messenger.MessengerService"
            android:process=":remote">
        </service>
```
接下来再看看客户端的实现，客户端的实现也比较简单，首先需要绑定远程进程的MessengerService，绑定成功后，根据服务端返回的binder对象创建Messenger对象并使用此对象向服务端发送消息。下面的代码在Bundle中向服务端发送了一句话，在上面的服务端代码中会打印出这句话。

``` java
public class MessengerActivity extends Activity {
    private static final String TAG = "MessengerActivity";
    private Messenger mService;
    private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            mService = new Messenger(service);
            Log.d(TAG, "bind service");
            Message msg = Message.obtain(null, MyConstants.MSG_FROM_CLIENT);
            Bundle data = new Bundle();
            data.putString("msg", "hello, this is client.");
            msg.setData(data);
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        public void onServiceDisconnected(ComponentName className) {
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        Intent intent = new Intent("com.ryg.MessengerService.launch");
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();
    }
}

```
可以从log中可以收到 client发送的消息
```
2019-03-24 23:35:52.489 3856-3856/com.ryg.chapter_2:remote I/MessengerService: receive msg from Client:hello, this is client.
```
通过上面的例子可以看出，在Mesenger中进行数据传递必须将数据放入Message中，而Messenger和Message都实现了Parcelable接口，因此可以跨进程传输。简单来说，Message中所支持的数据类型就是Messenger所支持的传输类型。实际上，通过Messenger来传输Message，Message中能使用的载体只有what、arg1、arg2、Bundle以及replyTo。Message中的另一个字段object在同一个进程中是很实用的，但是在进程间通信的时候，在Android2.2以前object字段不支持跨进程传输，即便是2.2以后，也仅仅是系统提供的实现了Parcelable接口的对象才能通过它来传输。**这就意味着我们自定义的Parcelable对象是无法通过object字段来传输的**，读者可以试一下。非系统的Parcelable对象的确无法通过object字段来传输，这也导致了object字段的实用性大大降低，所幸我们还有Bundle，Bundle中可以支持大量的数据类型。



上面的例子演示了如何在服务端接收客户端中发送的消息，但是有时候我们还需要能回应客户端，下面就介绍如何实现这种效果。还是采用上面的例子，但是稍微做一下修改，每当客户端发来一条消息，服务端就会自动回复一条“嗯，你的消息我已经收到，稍后会回复你。”，这很类似邮箱的自动回复功能。

首先看服务端的修改，服务端只需要修改MessengerHandler，当收到消息后，会立即回复一条消息给客户端。

```java
   private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case MyConstants.MSG_FROM_CLIENT:
                Log.i(TAG, "receive msg from Client:" + msg.getData().getString("msg"));
                Messenger client = msg.replyTo;
                Message relpyMessage = Message.obtain(null, MyConstants.MSG_FROM_SERVICE);
                Bundle bundle = new Bundle();
                bundle.putString("reply", "嗯，你的消息我已经收到，稍后会回复你。");
                relpyMessage.setData(bundle);
                try {
                    client.send(relpyMessage);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            default:
                super.handleMessage(msg);
            }
        }
    }
```

接着再看客户端的修改，为了接收服务端的回复，客户端也需要准备一个接收消息的Messenger和Handler，.如下所示。

```java
    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case MyConstants.MSG_FROM_SERVICE:
                Log.i(TAG, "receive msg from Service:" + msg.getData().getString("reply"));
                break;
            default:
                super.handleMessage(msg);
            }
        }
    }
```

修改面的程序后可以看到log变成了

```
2019-03-24 23:35:52.489 3856-3856/com.ryg.chapter_2:remote I/MessengerService: receive msg from Client:hello, this is client.
2019-03-24 23:35:52.618 3824-3824/com.ryg.chapter_2 I/MessengerActivity: receive msg from Service:嗯，你的消息我已经收到，稍后会回复你。
```
到这里，我们已经把采用Messenger进行进程间通信的方法都介绍完了，读者可以试着通过Messenger来实现更复杂的跨进程通信功能。下面给出一张Messenger的工作原理图以方便读者更好地理解Messenger。
![](https://sweetm-1256061026.cos.ap-beijing.myqcloud.com/blog/20190324235219.png)

### 4.4 使用AIDL

上面我们介绍了使用Messenger来进行进程间通信的方法，可以发现，Messenger是以**串行的方式**处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适了。同时，Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形用Messenger就无法做到了，但是我们可以使用AIDL来实现跨进程的方法调用。AIDL也是Messenger的底层实现，因此Messenger本质上也是AIDL，只不过系统为我们做了封装从而方便上层的调用而已。在上一节中，我们介绍了Binder的概念，大家对Binder也有了一定的了解，在Binder的基础上我们可以更加容易地理解AIDL。这里先介绍使用AIDL来进行进程间通信的流程，分为服务端和客户端两个方面。

#### 1.服务端
服务端首先要创建一个Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。
#### 2.客户端
客户端所要做事情就稍微简单一些，首先需要绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。

上面描述的只是一个感性的过程，AIDL的实现过程远不止这么简单，接下来会对其中的细节和难点进行详细介绍，并完善我们在Binder那一节所提供的的实例。
#### 3.AlDL接口的创建
首先看AIDL接口的创建，如下所示，我们创建了一个后缀为AIDL的文件，在里面声明了一个接口和两个接口方法。

```java
interface IBookManager {
    public List<Book> getBookList() throws RemoteException;
    public void addBook(Book book) throws RemoteException;
}
```

在AIDL文件中，并不是所有的数据类型都是可以使用的，那么到底AIDL文件支持哪些数据类型呢？如下所示。

* 基本数据类型（int、long、char、boolean、double等）；
* String和CharSequence；
* List：只支持ArrayList，里面每个元素都必须能够被AIDL支持；
* Map：只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value；
* Parcelable；所有实现了Parcelable接口的对象；
* AIDL：所有的AIDL接口本身也可以在AIDL文件中使用。

以上6种数据类型就是AIDL所支持的所有类型，其中自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内。比如IBookManager.aidl这个文件，里面用到了Book这个类，这个类实现了Parcelable接口并且和IBookManager.aidl位于同一个包中，但是遵守AIDL的规范，我们仍然需要显式地import进来：import com.ryg.chapter_2.aidl.Book。AIDL中会大量使用到Parcelable，至于如何使用Parcelable接口来序列化对象，在本章的前面已经介绍过，这里就不再赘述。

另外一个需要注意的地方是，如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。在上面的IBookManager.aidl中，我们用到了Book这个类，所以，我们必须要创建Book.aidl，然后在里面添加如下内容：

``` java
  parcelable Book;
```

我们需要注意，AIDL中每个实现了Parcelable接口的类都需要按照上面那种方式去创建相应的AIDL文件并声明那个类为parcelable。除此之外，AIDL中除了基本数据类型，其他类型的参数必须标上方向：in、out或者inout，in表示输入型参数，out表示输出型参数，inout表示输入输出型参数，至于它们具体的区别，这个就不说了。我们要根据实际需要去指定参数类型，不能一概使用out或者inout，因为这在底层实现是有开销的。最后，AIDL接口中只支持方法，不支持声明静态常量，这一点区别于传统的接口。

为了方便AIDL的开发，建议把所有和AIDL相关的类和文件全部放入同一个包中，这样做的好处是，当客户端是另外一个应用时，我们可以直接把整个包复制到客户端工程中，对于本例来说，就是要把com.ryg.chapter 2.aidl这个包和包中的文件原封不动地复制到客户端中。如果AIDL相关的文件位于不同的包中时，那么就需要把这些包一一复制到客户端工程中，这样操作起来比较麻烦而且也容易出错。需要注意的是，AIDL的包结构在服务端和客户端要保持一致，否则运行会出错，这是因为客户端需要反序列化服务端中和AIDL接口相关的所有类，如果类的完整路径不一样的话，就无法成功反序列化，程序也就无法正常运行。为了方便演示，本章的所有示例都是在同一个工程中进行的，但是读者要理解，一个工程和两个工程的多进程本质是一样的，两个工程的情况，除了需要复制AIDL接口所相关的包到客户端，其他完全一样，读者可以自行试验。

#### 4.服务端的实现

``` java
public class BookManagerService extends Service {
    private static final String TAG = "BMS";
    private AtomicBoolean mIsServiceDestoryed = new AtomicBoolean(false);
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<Book>();
    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }
        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };
    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "Ios"));
    }
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

上面是一个服务端Service的典型实现，首先在onCreate中初始化添加了两本图书的信息，然后创建了一个Binder对象并在onBind中返回它，这个对象继承自IBookManager.Stub并实现了它内部的AIDL方法，这个过程在Binder那一节已经介绍过了，这里就不多说了。
这里主要看getBookList和addBook这两个AIDL方法的实现，实现过程也比较简单，注意这里采用了CopyOnWriteArrayList，这个CopyOnWriteArrayList支持并发读/写。在前面我们提到，AIDL方法是在服务端的Binder线程池中执行的，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以我们要在AIDL方法中处理线程同步，而我们这里直接使用CopyOnWriteArrayList来进行自动的线程同步。

前面我们提到，AIDL中能够使用的List只有ArrayList，但是我们这里却使用了CopyOnWriteArrayList （注意它不是继承自ArrayList），为什么能够正常工作呢？这是因为AIDL中所支持的是抽象的List，而List只是一个接口，因此虽然服务端返回的是CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端。所以，我们在服务端采用CopyOnWriteArrayList是完全可以的。和此类似的还有ConcurrentHashMap，读者可以体会一下这种转换情形。然后我们需要在XML中注册这个Service，如下所示。注意BookManagerService是运行在独立进程中的，它和客户端的Activity不在同一个进程中，这样就构成了进程间通信的场景。

``` xml
        <service
            android:name=".aidl.BookManagerService"
            android:process=":remote"></service>
```
#### 5.客户端的实现

##### 查询书籍

客户端的实现就比较简单了，首先要绑定远程服务，绑定成功后将服务端返回的Binder对象转换成AIDL接口，然后就可以通过这个接口去调用服务端的远程方法了，代码如下所示。

``` java
    private ServiceConnection mConnection = new ServiceConnection() {
        //bind Service 后获取到对象
        public void onServiceConnected(ComponentName className, IBinder service) {
        //转化对象
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            mRemoteBookManager = bookManager;
            try {
              mRemoteBookManager.asBinder().linkToDeath(mDeathRecipient, 0);
                List<Book> list = bookManager.getBookList();
                Log.i(TAG, "query book list, list type:"
                        + list.getClass().getCanonicalName());
                Log.i(TAG, "query book list:" + list.toString());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        public void onServiceDisconnected(ComponentName className) {
            mRemoteBookManager = null;
            Log.d(TAG, "onServiceDisconnected. tname:" + Thread.currentThread().getName());
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book_manager);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

```

绑定成功以后，会通过bookManager 去调用getBookList方法，然后打印出所获取的图书信息。需要注意的是，服务端的方法有可能需要很久才能执行完毕，这个时候下面的代码就会导致ANR，这一点是需要注意的，后面会再介绍这种情况，之所以先这么写是为了让读者更好地了解AIDL的实现步骤。

接着在XML中注册此Activity，运行程序，log如下所示。
```
03-25 16:44:35.622 15908-15908/com.ryg.chapter_2 I/BookManagerActivity: query book list, list type:java.util.ArrayList
03-25 16:44:35.622 15908-15908/com.ryg.chapter_2 I/BookManagerActivity: query book list:[[bookId:1, bookName:Android], [bookId:2, bookName:Ios], [bookId:3, bookName:new book#3]]
```

可以发现，虽然我们在服务端返回的是CopyOnWriteArrayList类型，但是客户端收到的仍然是**ArrayList**类型，这也证实了我们在前面所做的分析。第二行log表明客户端成功地得到了服务端的图书列表信息。
这就是一次完完整整的使用AIDL进行IPC的过程，到这里相信读者对AIDL应该有了一个整体的认识了，但是还没完，AIDL的复杂性远不止这些，下面继续介绍AIDL中常见的一些难点。

##### 添加书籍

我们接着再调用一下另外一个接口addBook，我们在客户端给服务端添加一本书，然后再获取一次，看程序是否能够正常工作。还是上面的代码，客户端在服务连接后，在onServiceConnected中做如下改动：

```java
        public void onServiceConnected(ComponentName className, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            mRemoteBookManager = bookManager;
            try {
                mRemoteBookManager.asBinder().linkToDeath(mDeathRecipient, 0);
                List<Book> list = bookManager.getBookList();
                Log.i(TAG, "query book list, list type:"
                        + list.getClass().getCanonicalName());
                Log.i(TAG, "query book list:" + list.toString());
                Book newBook = new Book(3, "Android进阶");
                bookManager.addBook(newBook);
                Log.i(TAG, "add book:" + newBook);
                List<Book> newList = bookManager.getBookList();
                Log.i(TAG, "query book list:" + newList.toString());
                bookManager.registerListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
```

可以看到 log 日志

```logs
03-25 16:48:08.522 15908-15908/com.ryg.chapter_2 I/BookManagerActivity: query book list, list type:java.util.ArrayList
03-25 16:48:08.522 15908-15908/com.ryg.chapter_2 I/BookManagerActivity: query book list:[[bookId:1, bookName:Android], [bookId:2, bookName:Ios], [bookId:3, bookName:new book#3]]
03-25 16:48:08.522 15908-15908/com.ryg.chapter_2 I/BookManagerActivity: add book:[bookId:3, bookName:Android进阶]
03-25 16:48:13.532 15908-15908/com.ryg.chapter_2 I/BookManagerActivity: query book list:[[bookId:1, bookName:Android], [bookId:2, bookName:Ios], [bookId:3, bookName:new book#3], [bookId:3, bookName:Android进阶]]]
```



#####  动态监听添加的书籍

现在我们考虑一种情况，假设有一种需求：用户不想时不时地去查询图书列表了，太累了，于是，他去问图书馆，“当有新书时能不能把书的信息告诉我呢？”。大家应该明白了，这就是一种典型的观察者模式，每个感兴趣的用户都观察新书，当新书到的时候，图书馆就通知每一个对这本书感兴趣的用户，这种模式在实际开发中用得很多，下面我们就来模拟这种情形。首先，我们需要提供一个AIDL接口，每个用户都需要实现这个接口并且向图书馆申请新书的提醒功能，当然用户也可以随时取消这种提醒。之所以选择AIDL接口而不是普通接口，是因为AIDL中无法使用普通接口。这里我们创建一个IOnNewBookArrivedListener.aidl 文件，我们所期望的情况是：当服务端有新书到来时，就会通知每一个已经申请提醒功能的用户。从程序上来说就是调用所有IOnNew BookArrivedListener对象中的onNewBookArrived方法，并把新书的对象通过参数传递给客户端，内容如下所示。

```java
interface IOnNewBookArrivedListener {
    void onNewBookArrived(in Book newBook);
}
```

除了要新加一个AIDL接口，还需要在原有的接口中添加两个新方法，代码如下所示。

```java
 interface IBookManager {
      List<Book> getBookList();
      void addBook(in Book book);
      void registerListener(IOnNewBookArrivedListener listener);
      void unregisterListener(IOnNewBookArrivedListener listener);
 }
```

####

接着，服务端中Service的实现也要稍微修改一下，主要是Service中IBookManager.Stub的实现，因为我们在IBookManager新加了两个方法，所以在IBookManager.Stub中也要实现这两个方法。同时，在BookManagerService中还开启了一个线程，每隔5s就向书库中增加一本新书并通知所有感兴趣的用户，整个代码如下所示。

```java
public class BookManagerService extends Service {

    private static final String TAG = "BMS";

    private AtomicBoolean mIsServiceDestoryed = new AtomicBoolean(false);

    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<Book>();
    private CopyOnWriteArrayList<IOnNewBookArrivedListener> mListenerList = new CopyOnWriteArrayList<IOnNewBookArrivedListener>();
    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            SystemClock.sleep(5000);
            return mBookList;
        }
        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
        @Override
        public void registerListener(IOnNewBookArrivedListener listener)
                throws RemoteException {
            if (!mListenerList.contains(listener)) {
                mListenerList.add(listener);
            } else {
                Log.d(TAG, "already exists");
            }

            Log.d(TAG, "registerListener,  size:" + mListenerList.size());
        }
        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener)
                throws RemoteException {
            if (mListenerList.contains(listener)) {
                mListenerList.remove(listener);
                Log.d(TAG, "unregister listener succeed");
            } else {
                Log.d(TAG, "not found,can not unregister");
            }
            Log.d(TAG, "unregisterListener,current size:" + mListenerList.size());
        }
    };
    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "Ios"));
        new Thread(new ServiceWorker()).start();
    }
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    @Override
    public void onDestroy() {
        mIsServiceDestoryed.set(true);
        super.onDestroy();
    }

    private void onNewBookArrived(Book book) throws RemoteException {
        mBookList.add(book);
        Log.d(TAG, "onNewBookArrived,notify listeners:" + mListenerList.size());
        //遍历集合通知书籍到达
        for (int i = 0; i < mListenerList.size(); i++) {
            IOnNewBookArrivedListener listener = mListenerList.get(i);
            Log.d(TAG, "onNewBookArrived,notify listener:" + listener);
            listener.onNewBookArrived(book);
        }
    }
    private class ServiceWorker implements Runnable {
        @Override
        public void run() {
            // do background processing here.....
            while (!mIsServiceDestoryed.get()) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                int bookId = mBookList.size() + 1;
                Book newBook = new Book(bookId, "new book#" + bookId);
                try {
                    onNewBookArrived(newBook);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    }

}

```



最后，我们还需要修改一下客户端的代码，主要有两方面：首先客户端要注册IOnNewBookArrivedListener 到远程服务端，这样当有新书时服务端才能通知当前客户端，同时我们要在Activity 退出时解除这个注册；另一方面，当有新书时，服务端会回调客户端的IOnNewBookArrivedListener对象中的onNewBookArrived方法，但是这个方法是在客户端的Binder线程池中执行的，因此，为了便于进行UI操作，我们需要有一个Handler可以将其切换到客户端的主线程中去执行，这个原理在Binder中已经做了分析，这里就不多说了。客户端的代码修改如下：

```java
public class BookManagerActivity extends Activity {

    private static final String TAG = "BookManagerActivity";
    private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;
    private IBookManager mRemoteBookManager;
    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case MESSAGE_NEW_BOOK_ARRIVED:
                Log.d(TAG, "receive new book :" + msg.obj);
                break;
            default:
                super.handleMessage(msg);
            }
        }
    };
    private ServiceConnection mConnection = new ServiceConnection() {
        //bind Service 后获取到对象
        public void onServiceConnected(ComponentName className, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            mRemoteBookManager = bookManager;
            try {
             mRemoteBookManager.asBinder().linkToDeath(mDeathRecipient, 0);
                List<Book> list = bookManager.getBookList();
                Log.i(TAG, "query book list, list type:"
                        + list.getClass().getCanonicalName());
                Log.i(TAG, "query book list:" + list.toString());
                Book newBook = new Book(3, "Android进阶");
                bookManager.addBook(newBook);
                Log.i(TAG, "add book:" + newBook);
                List<Book> newList = bookManager.getBookList();
                Log.i(TAG, "query book list:" + newList.toString());
                bookManager.registerListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        public void onServiceDisconnected(ComponentName className) {
            mRemoteBookManager = null;
            Log.d(TAG, "onServiceDisconnected. tname:" + Thread.currentThread().getName());
        }
    };
    private IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {

        @Override
        public void onNewBookArrived(Book newBook) throws RemoteException {
            mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED, newBook)
                    .sendToTarget();
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book_manager);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        if (mRemoteBookManager != null
                && mRemoteBookManager.asBinder().isBinderAlive()) {
            try {
                Log.i(TAG, "unregister listener:" + mOnNewBookArrivedListener);
                mRemoteBookManager
                        .unregisterListener(mOnNewBookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        unbindService(mConnection);
        super.onDestroy();
    }

}


```

打印的log日志

```
2019-03-25 22:44:58.927 5594-5815/com.ryg.chapter_2:remote D/BMS: onNewBookArrived,notify listeners:1
2019-03-25 22:44:58.927 5594-5815/com.ryg.chapter_2:remote D/BMS: onNewBookArrived,notify listener:com.ryg.chapter_2.aidl.IOnNewBookArrivedListener$Stub$Proxy@dc2912b
2019-03-25 22:44:58.928 5560-5560/com.ryg.chapter_2 D/BookManagerActivity: receive new book :[bookId:290, bookName:new book#290]
2019-03-25 22:45:00.020 1425-1445/? D/hwcomposer: hw_composer sent 66 syncs in 60s
```



##### 回调事件的销毁 错误的方式

如果你以为到这里AIDL的介绍就结束了，那你就错了，之前就说过，AIDL远不止这么简单，目前还有一些难点是我们还没有涉及的，接下来将继续为读者介绍。
从上面的代码可以看出，当BookManagerActivity关闭时，我们会在onDestroy中去解除已经注册到服务端的listener，这就相当于我们不想再接收图书馆的新书提醒了，所以我们可以随时取消这个提醒服务。按back键退出BookManagerActivity，下面是打印出的log。

```
2019-03-25 22:52:43.777 5594-5834/com.ryg.chapter_2:remote D/BMS: onTransact: com.ryg.chapter_2
2019-03-25 22:52:43.777 5594-5834/com.ryg.chapter_2:remote D/BMS: not found,can not unregister
2019-03-25 22:52:43.778 5594-5834/com.ryg.chapter_2:remote D/BMS: unregisterListener,current size:1
```

从上面的log可以看出，程序没有像我们所预期的那样执行。在解注册的过程中，服务端竟然无法找到我们之前注册的那个listener，在客户端我们注册和解注册时明明传递的是同一个listener啊！最终，服务端由于无法找到要解除的listener而宣告解注册失败！这当然不是我们想要的结果，但是仔细想想，好像这种方式的确无法完成解注册。其实，这是必然的，这种解注册的处理方式在日常开发过程中时常使用到，但是放到多进程中却无法奏效，因为Binder会把客户端传递过来的对象重新转化并生成一个新的对象。虽然我们在注册和解注册过程中使用的是同一个客户端对象，但是通过Binder传递到服务端后，却会产生两个全新的对象。别忘了对象是不能跨进程直接传输的，对象的跨进程传输本质上都是反序列化的过程，这就是为什么AIDL中的自定义对象都必须要实现Parcelable接口的原因。那么到底我们该怎么做才能实现解注册功能呢？答案是使用RemoteCallbackList，这看起来很抽象，不过没关系，请看接下来的详细分析。

##### 回调事件销毁 正确的方式

RemoteCallbackList 是系统专门提供的用于删除跨进程listener的接口。Remote-
CalbackList是一个泛型，支持管理任意的AIDL接口，这点从它的声明就可以看出，因为所有的AIDL接口都继承自lInterface接口，读者还有印象吗？

```java
public class RemoteCallbackList<E extends IInterface>
```

它的工作原理很简单，在它的内部有一个Map结构专门用来保存所有的AIDL回调，这个Map的key是IBinder类型，value是Callback类型，如下所示。

```java
ArrayMap<IBinder, Callback> mCallbacks= new ArrayMap<IBinder, Callback>();
```

其中Callback中封装了真正的远程listener。当客户端注册listener的时候，它会把这个listener的信息存入mCalbacks中，其中key和value分别通过下面的方式获得：

```java
IBinder key=listener. asBinder()
Callback value=new Callback(listener, cookie)
```

到这里，读者应该都明白了，虽然说多次跨进程传输客户端的同一个对象会在服务端生成不同的对象，但是这些新生成的对象有一个共同点，那就是它们底层的Binder对象是同一个，利用这个特性，就可以实现上面我们无法实现的功能。当客户端解注册的时候，我们只要遍历服务端所有的listener，找出那个和解注册listener具有相同Binder对象的服务端listener 并把它删掉即可，这就是RemoteCallbackList为我们做的事情。同时RemoteCallbackList还有一个很有用的功能，那就是当客户端进程终止后，它能够自动移除客户端所注册的listener。另外，RemoteCallbackList内部自动实现了线程同步的功能，所以我们使用它来注册和解注册时，不需要做额外的线程同步工作。由此可见，RemoteCallbackList的确是个很有价值的类，下面就演示如何使用它来完成解注册。

RemoteCallbackList 使用起来很简单，我们要对BookManagerService做一些修改，首先要创建一个RemoteCallbackList对象来替代之前的CopyOnWriteArrayList，如下所示。

```java
private RemoteCallbackList<IOnNewBookArrivedListener> mListenerList = new RemoteCallbackList<IOnNewBookArrivedListener>();
```

然后修改registerListener和unregisterListener这两个接口的实现，如下所示。

```java
        @Override
        public void registerListener(IOnNewBookArrivedListener listener)
                throws RemoteException {
            mListenerList.register(listener);
        }


        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener)
                throws RemoteException {
            boolean success = mListenerList.unregister(listener);
        }
```



怎么样？是不是用起来很简单，接着要修改onNewBookArrived方法，当有新书时，我们就要通知所有已注册的listener，如下所示。

```java
    private void onNewBookArrived(Book book) throws RemoteException {
        mBookList.add(book);
        final int N = mListenerList.beginBroadcast();
        for (int i = 0; i < N; i++) {
            IOnNewBookArrivedListener l = mListenerList.getBroadcastItem(i);
            if (l != null) {
                try {
                    l.onNewBookArrived(book);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
        mListenerList.finishBroadcast();
    }
```

BookManagerService的修改已经完毕了，为了方便我们验证程序的功能，我们还需要添加一些log，在注册和解注册后我们分别打印出所有listener的数量。如果程序正常工作的话，那么注册之后listener总数量是1解注册之后总数量应该是0，我们再次运行一下程序，看是否如此。从下面的log来看，很显然，使用RemoteCallbackList的确可以完成跨进程的解注册功能。

```java
2019-03-26 00:04:26.056 11421-11616/com.ryg.chapter_2:remote D/BMS: unregister success.
2019-03-26 00:04:26.056 11421-11616/com.ryg.chapter_2:remote D/BMS: unregisterListener, current size:0
```

使用RemoteCallbackList，有一点需要注意，我们无法像操作List一样去操作它，尽管它的名字中也带个List，但是它并不是一个List。遍历RemoteCalbackList，必须要按照下面的方式进行，其中beginBroadcast和finishBroadcast必须要配对使用，哪怕我们仅仅是想要获取RemoteCallbackList中的元素个数，这是必须要注意的地方。

```java
        final int N = mListenerList.beginBroadcast();
        for (int i = 0; i < N; i++) {
            IOnNewBookArrivedListener l = mListenerList.getBroadcastItem(i);
            if (l != null) {
              //todo
            }
        }
        mListenerList.finishBroadcast();
```



到这里，AIDL的基本使用方法已经介绍完了，但是有几点还需要再次说明一下。我们知道，客户端调用远程服务的方法，被调用的方法运行在服务端的Binder线程池中，同时客户端线程会被挂起，这个时候如果服务端方法执行比较耗时，就会导致客户端线程长时间地阻塞在这里，而如果这个客户端线程是UI线程的话，就会导致客户端ANR，这当然不是我们想要看到的。因此，如果我们明确知道某个远程方法是耗时的，那么就要避免在客户端的UI线程中去访问远程方法。由于客户端的onServiceConnected和onService Disconnected方法都运行在UI线程中，所以也不可以在它们里面直接调用服务端的耗时方法，这点要尤其注意。另外，由于服务端的方法本身就运行在服务端的Binder线程池中，所以服务端方法本身就可以执行大量耗时操作，这个时候切记不要在服务端方法中开线程去进行异步任务，除非你明确知道自己在干什么，否则不建议这么做。下面我们稍微改造一下服务端的getBookList方法，我们假定这个方法是耗时的，那么服务端可以这么实现：

```java
        @Override
        public List<Book> getBookList() throws RemoteException {
            SystemClock.sleep(5000);
            return mBookList;
        }
```

然后在客户端中放一个按钮，单击它的时候就会调用服务端的getBookList方法，可以预知，连续单击几次，客户端就ANR了，如下图，感兴趣读者可以自行试一下。

![](https://sweetm-1256061026.cos.ap-beijing.myqcloud.com/blog/20190326001328.png)

避免出现上述这种ANR其实很简单，我们只需要把调用放在非UI线程即可，如下所示。

```java
    public void onButton1Click(View view) {
        Toast.makeText(this, "click button1", Toast.LENGTH_SHORT).show();
        new Thread(new Runnable() {
            @Override
            public void run() {
                if (mRemoteBookManager != null) {
                    try {
                        List<Book> newList = mRemoteBookManager.getBookList();
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
```

同理，当远程服务端需要调用客户端的listener中的方法时，被调用的方法也运行在Binder线程池中，只不过是客户端的线程池。所以，我们同样不可以在服务端中调用客户端的耗时方法。比如针对BookManagerService的onNewBookArrived方法，如下所示。在它内部调用了客户端的IOnNewBookArrivedListener中的onNewBookArrived方法，如果客户端的这个onNewBookArrived方法比较耗时的话，那么请确保BookManagerService中的onNewBookArrived运行在非UI线程中，否则将导致服务端无法响应。

```java
    private void onNewBookArrived(Book book) throws RemoteException {
        mBookList.add(book);
        Log.d(TAG, "onNewBookArrived,notify listeners:" + mListenerList.size());
        //遍历集合通知书籍到达
        for (int i = 0; i < mListenerList.size(); i++) {
            IOnNewBookArrivedListener listener = mListenerList.get(i);
            Log.d(TAG, "onNewBookArrived,notify listener:" + listener);
            listener.onNewBookArrived(book);
        }
    }
```



另外，由于客户端的IOnNewBookArrivedListener中的onNewBookArrived方法运行在客户端的Binder线程池中，所以不能在它里面去访问UI相关的内容，如果要访问UI，请使用Handler切换到UI线程，这一点在前面的代码实例中已经有所体现，这里就不再详细描述了。

为了程序的健壮性，我们还需要做一件事。**Binder是可能意外死亡的**，这往往是由于服务端进程意外停止了，这时我们需要重新连接服务。有两种方法，第一种方法是给Binder设置**DeathRecipient**监听，当Binder死亡时，我们会收到binderDied方法的回调，在binderDied方法中我们可以重连远程服务，具体方法在Binder那一节已经介绍过了，这里就不再详细描述了。另一种方法是在onServiceDisconnected中重连远程服务。这两种方法我们可以随便选择一种来使用，它们的区别在于：**onServiceDisconnected在客户端的UI线程中被回调**，而**binderDied在客户端的Binder 线程池中被回调**。也就是说，在binderDied方法中我们不能访问UI，这就是它们的区别。

#### 6.AIDL 权限验证

到此为止，我们已经对AIDL有了一个系统性的认识，但是还差最后一步：如何在AIDL中使用权限验证功能。默认情况下，我们的远程服务任何人都可以连接，但这应该不是我们愿意看到的，所以我们必须给服务加入权限验证功能，权限验证失败则无法调用服务中的方法。在AIDL中进行权限验证，这里介绍两种常用的方法。

##### 第一种方法
我们可以在onBind中进行验证，验证不通过就直接返回null，这样验证失败的客户端直接无法绑定服务，至于验证方式可以有多种，比如使用permission验证。
使用这种验证方式，我们要先在AndroidMenifest中声明所需的权限，比如：

```xml
    <permission
        android:name="com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE"
        android:protectionLevel="normal" />
```

关于permission的定义方式请读者查看相关资料，这里就不详细展开了，毕竟本节的主要内容是介绍AIDL。定义了权限以后，就可以在BookManagerService的onBind方法中做权限验证了，如下所示。

```
    @Override
    public IBinder onBind(Intent intent) {
        int check = checkCallingOrSelfPermission("com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE");
        Log.d(TAG, "onbind check=" + check);
        if (check == PackageManager.PERMISSION_DENIED) {
            return null;
        }
        return mBinder;
    }
```

一个应用来绑定我们的服务时，会验证这个应用的权限，如果它没有使用这个权限，onBind方法就会直接返回null，最终结果是这个应用无法绑定到我们的服务，这样就达到了权限验证的效果，这种方法同样适用于Messenger中，读者可以自行扩展。
如果我们自己内部的应用想绑定到我们的服务中，只需要在它的AndroidMenifest文件中采用如下方式使用permission即可。

```xml
<uses-permission android:name="com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE" />
```

##### 第二种方法
我们可以在服务端的onTransact方法中进行权限验证，如果验证失败就直接返回false，这样服务端就不会终止执行AIDL中的方法从而达到保护服务端的效果。
至于具体的验证方式有很多，可以采用permission验证，具体实现方式和第一种方法一样。还可以采用Uid和Pid来做验证，通过getCallingUid和getCallingPid可以拿到客户端所属应用的Uid和Pid，通过这两个参数我们可以做一些验证工作，比如验证包名。在下面的代码中，既验证了permission，又验证了包名。一个应用如果想远程调用服务中的方法，首先要使用我们的自定义权限“com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE”，其次包名必须以“com.ryg”开始，否则调用服务端的方法会失败。

```java
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
                throws RemoteException {
            int check = checkCallingOrSelfPermission("com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE");
            Log.d(TAG, "check=" + check);
            if (check == PackageManager.PERMISSION_DENIED) {
                return false;
            }

            String packageName = null;
            String[] packages = getPackageManager().getPackagesForUid(
                    getCallingUid());
            if (packages != null && packages.length > 0) {
                packageName = packages[0];
            }
            Log.d(TAG, "onTransact: " + packageName);
            if (!packageName.startsWith("com.ryg")) {
                return false;
            }
            return super.onTransact(code, data, reply, flags);
        }
```

上面介绍了两种AIDL中常用的权限验证方法，但是肯定还有其他方法可以做权限验证，比如为Service指定android:permission属性等，这里就不一一进行介绍了。

## 五、使用ContentProvider

ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，从这一点来看，它天生就适合进程间通信。和Messenger一样，ContentProvider的底层实现同样也是Binder，由此可见，Binder在Android系统中是何等的重要。虽然ContentProvider的底层实现是Binder，但是它的使用过程要比AIDL简单许多，这是因为系统已经为我们做了封装，使得我们无须关心底层细节即可轻松实现IPC。ContentProvider虽然使用起来很简单，包括自己创建一个ContentProvider也不是什么难事，尽管如此，它的细节还是相当多，比如CRUD操作、防止SQL注入和权限控制等。由于章节主题限制，在本节中，笔者暂时不对ContentProvider的使用细节以及工作机制进行详细分析，而是为读者介绍采用ContentProvider进行跨进程通信的主要流程，至于使用细节和内部工作机制会在后续章节进行详细分析。

系统预置了许多ContentProvider，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。在本节中，我们来实现一个自定义的ContentProvider，并演示如何在其他应用中获取ContentProvider中的数据从而实现进程间通信这一目的。首先，我们创建一个ContentProvider，名字就叫BookProvider。创建一个自定义的ContentProvider很简单，只需要继承ContentProvider类并实现六个抽象方法即可：onCreate、query、update、insert、delete和getlype。这六个抽象方法都很好理解，onCreate代表ContentProvider的创建，一般来说我们需要做一些初始化工作；getType用来返回一个Uri请求所对应的MIME类型（媒体类型），比如图片、视频等，这个媒体类型还是有点复杂的，如果我们的应用不关注这个选项，可以直接在这个方法中返回null或者“\*\*”；剩下的四个方法对应于CRUD操作，即实现对数据表的增删改查功能。根据Binder的工作原理，我们知道这六个方法均运行在ContentProvider的进程中，除了onCreate由系统回调并运行在主线程里，其他五个方法均由外界回调并运行在Binder线程池中，这一点在接下来的例子中可以再次证明。

ContentProvider主要以表格的形式来组织数据，并且可以包含多个表，对于每个表格来说，它们都具有行和列的层次性，行往往对应一条记录，而列对应一条记录中的一个字段，这点和数据库很类似。除了表格的形式，ContentProvider还支持文件数据，比如图片、视频等。文件数据和表格数据的结构不同，因此处理这类数据时可以在ContentProvider中返回文件的句柄给外界从而让文件来访问ContentProvider中的文件信息。Android系统所提供的MediaStore功能就是文件类型的ContentProvider，详细实现可以参考MediaStore。另外，虽然ContentProvider的底层数据看起来很像一个SQLite数据库，但是ContentProvider对底层的数据存储方式没有任何要求，我们既可以使用SOLite数据库，也可以使用普通的文件，甚至可以采用内存中的一个对象来进行数据的存储，这一点在后续的章节中会再次介绍，所以这里不再深入了。


下面看一个最简单的示例，它演示了ContentProvider的工作工程。首先创建一个BookProvider类，它继承自ContentProvider并实现了ContentProvider的六个必须需要实现的抽象方法。在下面的代码中，我们什么都没干，尽管如此，这个BookProvider也是可以工作的，只是它无法向外界提供有效的数据而已。

``` java
public class BookProvider extends ContentProvider {
    private static final String TAG = "BookProvider";
    @Override
    public boolean onCreate() {
        Log.d(TAG, "onCreate, current thread:"
                + Thread.currentThread().getName());
        return false;
    }
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        Log.d(TAG, "query, current thread:" + Thread.currentThread().getName());
        return null;
    }
    @Override
    public String getType(Uri uri) {
        Log.d(TAG, "getType");
        return null;
    }
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        Log.d(TAG, "insert");
        return null;
    }
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        Log.d(TAG, "delete");
        return 0;
    }
    @Override
    public int update(Uri uri, ContentValues values, String selection,
            String[] selectionArgs) {
        Log.d(TAG, "update");
        return 0;
    }
}

```

接着我们需要注册这个BookProvider，如下所示。其中android:authorities是Content Provider的唯一标识，通过这个属性外部应用就可以访问我们的BookProvider，因此，android:authorities必须是唯一的，这里建议读者在命名的时候加上包名前缀。为了演示进程间通信，我们让BookProvider运行在独立的进程中并给它添加了权限，这样外界应用如果想访问BookProvider，就必须声明“com.ry.PROVIDER”这个权限。ContentProvider的权限还可以细分为读权限和写权限，分别对应android:readPermission和android:writePermission 属性，如果分别声明了读权限和写权限，那么外界应用也必须依沙声明相应的权限才可以进行读/写操作，否则外界应用会异常终止。关于权限这一块，请读者自行查阅相关资料，本章不进行详细介绍。

``` xml
        <provider
            android:name=".provider.BookProvider"
            android:authorities="com.ryg.chapter_2.book.provider"
            android:permission="com.ryg.PROVIDER"
            android:process=":provider"></provider>
```

注册了ContentProvider以后，我们就可以在外部应用中访问它了。为了方便演示，这里仍然选择在同一个应用的其他进程中去访问这个BookProvider，至于在单独的应用中去访问这个BookProvider，和同一个应用中访问的效果是一样的，读者可以自行试一下（注意要声明对应权限）。

``` java
public class ProviderActivity extends Activity {
    private static final String TAG = "ProviderActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_provider);
         Uri uri = Uri.parse("content://com.ryg.chapter_2.book.provider");
         getContentResolver().query(uri, null, null, null, null);
         getContentResolver().query(uri, null, null, null, null);
         getContentResolver().query(uri, null, null, null, null);
    }
}

```
在上面的代码中，我们通过ContentResolver对象的query方法去查询BookProvider中的数据，其中“content://com.ryg.chapter_2.boo.provider”唯一标识了BookProvider，而这个标识正是我们前面为BookProvider的android:authorities属性所指定的值。我们运行后看一下log。从下面log可以看出，BookProvider中的query方法被调用了三次，并且这三次调用不在同一个线程中。可以看出，它们运行在一个Binder 线程中，前面提update、insert和delete方法同样也运行在Binder线程中。另外，onCreate运行在main线程中，也就是UI线程，所以我们不能在onCreate中做耗时操作。


``` java
2019-03-26 11:38:01.381 6215-6215/com.ryg.chapter_2:provider D/BookProvider: onCreate, current thread:main
2019-03-26 11:38:01.395 6215-6228/com.ryg.chapter_2:provider D/BookProvider: query, current thread:Binder:6215_2
2019-03-26 11:38:01.397 6215-6215/com.ryg.chapter_2:provider D/MyApplication: application start, process name:com.ryg.chapter_2:provider
2019-03-26 11:38:01.399 6215-6227/com.ryg.chapter_2:provider D/BookProvider: query, current thread:Binder:6215_1
2019-03-26 11:38:01.401 6215-6228/com.ryg.chapter_2:provider D/BookProvider: query, current thread:Binder:6215_2

```

到这里，整个ContentProvider的流程我们已经跑通了，虽然ContentProvider中没有返回任何数据。接下来，在上面的基础上，我们继续完善BookProvider，从而使其能够对外部应用提供数据。继续本章提到的那个例子，现在我们要提供一个BookProvider，外部应用可以通过BookProvider来访问图书信息，为了更好地演示ContentProvider的使用，用户还可以通过BookProvider访问到用户信息。为了完成上述功能，我们需要一个数据库来管理图书和用户信息，这个数据库不难实现，代码如下：

``` java
public class DbOpenHelper extends SQLiteOpenHelper {
    private static final String DB_NAME = "book_provider.db";
    public static final String BOOK_TABLE_NAME = "book";
    public static final String USER_TALBE_NAME = "user";
    private static final int DB_VERSION = 3;
    private String CREATE_BOOK_TABLE = "CREATE TABLE IF NOT EXISTS "
            + BOOK_TABLE_NAME + "(_id INTEGER PRIMARY KEY," + "name TEXT)";
    private String CREATE_USER_TABLE = "CREATE TABLE IF NOT EXISTS "
            + USER_TALBE_NAME + "(_id INTEGER PRIMARY KEY," + "name TEXT,"
            + "sex INT)";
    public DbOpenHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }
    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(CREATE_BOOK_TABLE);
        db.execSQL(CREATE_USER_TABLE);
    }
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        // TODO ignored
    }
}

```

上述代码是一个最简单的数据库的实现，我们借助SQLiteOpenHelper来管理数据库的创建、升级和降级。下面我们就要通过BookProvider向外界提供上述数据库中的信息了。我们知道，ContentProvider通过Uri来区分外界要访问的的数据集合，在本例中支持外界对BookProvider中的book表和user表进行访问，为了知道外界要访问的是哪个表，我们需要为它们定义单独的Uri和Uri Code，并将Uri和对应的Uri_Code相关联，我们可以使用UriMatcher的addURI方法将Uri和Uri_Code关联到一起。这样，当外界请求访问BookProvider时，我们就可以根据请求的Uri来得到Uri_Code，有了Uri_Code我们就可以知道外界想要访问哪个表，然后就可以进行相应的数据操作了，具体代码如下所示。

``` java
public class BookProvider extends ContentProvider {
    private static final String TAG = "BookProvider";
    public static final String AUTHORITY = "com.ryg.chapter_2.book.provider";

    /* 创建匹配路径的Uri */
    public static final Uri BOOK_CONTENT_URI = Uri.parse("content://"
            + AUTHORITY + "/book");
    public static final Uri USER_CONTENT_URI = Uri.parse("content://"
            + AUTHORITY + "/user");


    public static final int BOOK_URI_CODE = 0;
    public static final int USER_URI_CODE = 1;
    private static final UriMatcher sUriMatcher = new UriMatcher(
            UriMatcher.NO_MATCH);
    static {
        sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
        sUriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
    }

    ...
}
```
从上面代码可以看出，我们分别为book表和user表指定了Uri，分别为“content://com.ryg.chapter_2.book.provider/book”和“content:/com.ryg.chapter_2.book.provider/user”，这两个Uri所关联的Uri_Code分别为0和1。这个关联过程是通过下面的语句来完成的：

``` java
    private String getTableName(Uri uri) {
        String tableName = null;
        switch (sUriMatcher.match(uri)) {
        case BOOK_URI_CODE:
            tableName = DbOpenHelper.BOOK_TABLE_NAME;
            break;
        case USER_URI_CODE:
            tableName = DbOpenHelper.USER_TALBE_NAME;
            break;
            default:break;
        }
        return tableName;
    }
```

接着，我们就可以实现query、update、insert、delete方法了。如下是query方法的实现，首先我们要从Uri中取出外界要访问的表的名称，然后根据外界传递的查询参数就可以进行数据库的查询操作了，这个过程比较简单。

``` java
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        Log.d(TAG, "query, current thread:" + Thread.currentThread().getName());
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        return mDb.query(table, projection, selection, selectionArgs, null, null, sortOrder, null);
    }
```

另外三个方法的实现思想和query是类似的，只有一点不同，那就是update、insert和delete方法会引起数据源的改变，这个时候我们需要通过ContentResolver的 notifyChange方法来通知外界当前ContentProvider中的数据已经发生改变。要观察一个ContentProvider中的数据改变情况，可以通过ContentResolver的 registerContentObserver方法来注册观察者，通过unregisterContentObserver方法来解除观察者。对于这三个方法，这里不再详细解释了，BookProvider的完整代码如下：

```
public class BookProvider extends ContentProvider {
    private static final String TAG = "BookProvider";
    public static final String AUTHORITY = "com.ryg.chapter_2.book.provider";
    public static final Uri BOOK_CONTENT_URI = Uri.parse("content://"
            + AUTHORITY + "/book");
    public static final Uri USER_CONTENT_URI = Uri.parse("content://"
            + AUTHORITY + "/user");
    public static final int BOOK_URI_CODE = 0;
    public static final int USER_URI_CODE = 1;
    private static final UriMatcher sUriMatcher = new UriMatcher(
            UriMatcher.NO_MATCH);
    static {
        sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
        sUriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
    }
    private Context mContext;
    private SQLiteDatabase mDb;
    @Override
    public boolean onCreate() {
        Log.d(TAG, "onCreate, current thread:" + Thread.currentThread().getName());
        mContext = getContext();
        initProviderData();
        return true;
    }
    private void initProviderData() {
        mDb = new DbOpenHelper(mContext).getWritableDatabase();
        mDb.execSQL("delete from " + DbOpenHelper.BOOK_TABLE_NAME);
        mDb.execSQL("delete from " + DbOpenHelper.USER_TALBE_NAME);
        mDb.execSQL("insert into book values(3,'Android');");
        mDb.execSQL("insert into book values(4,'Ios');");
        mDb.execSQL("insert into book values(5,'Html5');");
        mDb.execSQL("insert into user values(1,'jake',1);");
        mDb.execSQL("insert into user values(2,'jasmine',0);");
    }
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        Log.d(TAG, "query, current thread:" + Thread.currentThread().getName());
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        return mDb.query(table, projection, selection, selectionArgs, null, null, sortOrder, null);
    }
    @Override
    public String getType(Uri uri) {
        Log.d(TAG, "getType");
        return null;
    }
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        Log.d(TAG, "insert");
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        mDb.insert(table, null, values);
        mContext.getContentResolver().notifyChange(uri, null);
        return uri;
    }
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        Log.d(TAG, "delete");
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        int count = mDb.delete(table, selection, selectionArgs);
        if (count > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }
    @Override
    public int update(Uri uri, ContentValues values, String selection,
            String[] selectionArgs) {
        Log.d(TAG, "update");
        String table = getTableName(uri);
        if (table == null) {
            throw new IllegalArgumentException("Unsupported URI: " + uri);
        }
        int row = mDb.update(table, values, selection, selectionArgs);
        if (row > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return row;
    }
    private String getTableName(Uri uri) {
        String tableName = null;
        switch (sUriMatcher.match(uri)) {
        case BOOK_URI_CODE:
            tableName = DbOpenHelper.BOOK_TABLE_NAME;
            break;
        case USER_URI_CODE:
            tableName = DbOpenHelper.USER_TALBE_NAME;
            break;
            default:break;
        }
        return tableName;
    }
}

```
需要注意的是，query、update、insert、delete四大方法是存在**多线程并发访问**的，因此方法内部要做好线程同步。在本例中，由于采用的是SQLite并且只有一个SQLiteDatabase的连接，所以可以正确应对多线程的情况。具体原因是SQLiteDatabase内部对数据库的操作是有同步处理的，但是如果通过多个SQLiteDatabase对象来操作数据库就无法保证线程同步，因为SQLiteDatabase对象之间无法进行线程同步。如果ContentProvider的底层数据集是一块内存的话，比如是List，在这种情况下同List的遍历、插入、删除操作就需要进行线程同步，否则就会引起并发错误，这点是尤其需要注意的。到这里BookProvider已经实现完成了，接着我们在外部访问一下它，看看是否能够正常工作。

``` java
public class ProviderActivity extends Activity {
    private static final String TAG = "ProviderActivity";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_provider);
//         Uri uri = Uri.parse("content://com.ryg.chapter_2.book.provider");
//         getContentResolver().query(uri, null, null, null, null);
//         getContentResolver().query(uri, null, null, null, null);
//         getContentResolver().query(uri, null, null, null, null);
//
        Uri bookUri = Uri.parse("content://com.ryg.chapter_2.book.provider/book");
        ContentValues values = new ContentValues();
        values.put("_id", 6);
        values.put("name", "程序设计的艺术");
        getContentResolver().insert(bookUri, values);
        Cursor bookCursor = getContentResolver().query(bookUri, new String[]{"_id", "name"}, null, null, null);
        while (bookCursor.moveToNext()) {
            Book book = new Book();
            book.bookId = bookCursor.getInt(0);
            book.bookName = bookCursor.getString(1);
            Log.d(TAG, "query book:" + book.toString());
        }
        bookCursor.close();
        Uri userUri = Uri.parse("content://com.ryg.chapter_2.book.provider/user");
        Cursor userCursor = getContentResolver().query(userUri, new String[]{"_id", "name", "sex"}, null, null, null);
        while (userCursor.moveToNext()) {
            User user = new User();
            user.userId = userCursor.getInt(0);
            user.userName = userCursor.getString(1);
            user.isMale = userCursor.getInt(2) == 1;
            Log.d(TAG, "query user:" + user.toString());
        }
        userCursor.close();
    }
}

```

默认情况下，BookProvider的数据库中有三本书和两个用户，在上面的代码中，我们首先添加一本书：“程序设计的艺术”。接着查询所有的图书，这个时候应该查询出四本书，因为我们刚刚添加了一本。然后查询所有的用户，这个时候应该查询出两个用户。是不是这样呢？我们运行一下程序，看一下log。

```
2019-03-26 15:43:18.451 18553-18553/com.ryg.chapter_2 D/ProviderActivity: query book:[bookId:3, bookName:Android]
2019-03-26 15:43:18.451 18553-18553/com.ryg.chapter_2 D/ProviderActivity: query book:[bookId:4, bookName:Ios]
2019-03-26 15:43:18.452 18553-18553/com.ryg.chapter_2 D/ProviderActivity: query book:[bookId:5, bookName:Html5]
2019-03-26 15:43:18.453 18553-18553/com.ryg.chapter_2 D/ProviderActivity: query book:[bookId:6, bookName:程序设计的艺术]
2019-03-26 15:43:18.460 18553-18553/com.ryg.chapter_2 D/ProviderActivity: query user:User:{userId:1, userName:jake, isMale:true}, with child:{null}
2019-03-26 15:43:18.461 18553-18553/com.ryg.chapter_2 D/ProviderActivity: query user:User:{userId:2, userName:jasmine, isMale:false}, with child:{null}

```

从上述log可以看到，我们的确查询到了4本书和2个用户，这说明BookProvider已经能够正确地处理外部的请求了，读者可以自行验证一下update和delete操作，这里就不再验证了。同时，由于ProviderActivity和BookProvider运行在两个不同的进程中，因此，这也构成了进程间的通信。ContentProvider除了支持对数据源的增删改查这四个操作，还支持自定义调用，这个过程是通过ContentResolver的Call 方法和ContentProvider的Call方法来完成的。关于使用ContentProvider来进行IPC就介绍到这里。

## 六、使用Socket

我们通过Socket来实现进程间的通信。Socket也称为“套接字”，是网络通信中的概念，它分为流式套接字和用户数据报套接字两种，分别对应于网络的传输控制层中的TCP和UDP协议。TCP协议是**面向连接**的协议，提供稳定的双向通信功能，TCP连接的建立需要经过“三次握手”才能完成，为了提供稳定的数据传输功能，其本身提供了超时重传机制，因此具有很高的稳定性；而UDP是**无连接**的，提供不稳定的单向通信功能，当然UDP也可以实现双向通信功能。在性能上，UDP具有更好的效率，其缺点是不保证数据一定能够正确传输，尤其是在网络拥塞的情况下。关于TCP和UDP的介绍就这么多，更详细的资料请查看相关网络资料。接下来我们演示一个跨进程的聊天程序，两个进程可以通过Socket来实现信息的传输，Socket本身可以支持传输任意字节流，这里为了简单起见，仅仅传输文本信息，很显然，这是一种IPC方式。

使用Socket来进行通信，有两点需要注意，首先需要声明权限：

``` xml
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

其次要注意不能在主线程中访问网络，因为这会导致我们的程序无法在Android4.0及其以上的设备中运行，会抛出如下异常：android.os.NetworkOnMain ThreadException。而且进行网络操作很可能是耗时的，如果放在主线程中会影响程序的响应效率，从这方面来说，也不应该在主线程中访问网络。下面就开始设计我们的聊天室程序了，比较简单，首先在远程Service建立一个TCP服务，然后在主界面中连接TCP服务，连接上了以后，就可以给服务端发消息。对于我们发送的每一条文本消息，服务端都会随机地回应我们一句话。为了更好地展示Socket的工作机制，在服务端我们做了处理，使其能够和多个客户端同时建立连接并响应。

先看一下服务端的设计，当Service启动时，会在线程中建立TCP服务，这里监听的是8688端口，然后就可以等待客户端的连接请求。当有客户端连接时，就会生成一个新的Socket，通过每次新创建的Socket就可以分别和不同的客户端通信了。服务端每收到一次客户端的消息就会随机回复一句话给客户端。当客户端断开连接时，服务端这边也会相应的关闭对应Socket并结束通话线程，这点是如何做到的呢？方法有很多，这里是通过判断服务端输入流的返回值来确定的，当客户端断开连接后，服务端这边的输入流会返回null，这个时候我们就知道客户端退出了。服务端的代码如下所示。



```java
public class TCPServerService extends Service {
    private boolean mIsServiceDestoryed = false;
    private String[] mDefinedMessages = new String[] {
            "你好啊，哈哈",
            "请问你叫什么名字呀？",
            "今天北京天气不错啊，shy",
            "你知道吗？我可是可以和多个人同时聊天的哦",
            "给你讲个笑话吧：据说爱笑的人运气不会太差，不知道真假。"
    };

    @Override
    public void onCreate() {
        new Thread(new TcpServer()).start();
        super.onCreate();
    }
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
    @Override
    public void onDestroy() {
        mIsServiceDestoryed = true;
        super.onDestroy();
    }
    private class TcpServer implements Runnable {
        @SuppressWarnings("resource")
        @Override
        public void run() {
            ServerSocket serverSocket = null;
            try {
                serverSocket = new ServerSocket(8688);
            } catch (IOException e) {
                System.err.println("establish tcp server failed, port:8688");
                e.printStackTrace();
                return;
            }
            while (!mIsServiceDestoryed) {
                try {
                    // 接受客户端请求
                    final Socket client = serverSocket.accept();
                    System.out.println("accept");
                    new Thread() {
                        @Override
                        public void run() {
                            try {
                                responseClient(client);
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        };
                    }.start();

                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    private void responseClient(Socket client) throws IOException {
        // 用于接收客户端消息
        BufferedReader in = new BufferedReader(new InputStreamReader(
                client.getInputStream()));
        // 用于向客户端发送消息
        PrintWriter out = new PrintWriter(new BufferedWriter(
                new OutputStreamWriter(client.getOutputStream())), true);
        out.println("欢迎来到聊天室！");
        while (!mIsServiceDestoryed) {
            String str = in.readLine();
            System.out.println("msg from client:" + str);
            if (str == null) {
                break;
            }
            int i = new Random().nextInt(mDefinedMessages.length);
            String msg = mDefinedMessages[i];
            out.println(msg);
            System.out.println("send :" + msg);
        }
        System.out.println("client quit.");
        // 关闭流
        MyUtils.close(out);
        MyUtils.close(in);
        client.close();
    }
}

```

接着看一下客户端，客户端Activity启动时，会在onCreate中开启一个线程去连接服务端Socket，至于为什么用线程在前面已经做了介绍。为了确定能够连接成功，这里采用了超时重连的策略，每次连接失败后都会重新建立尝试建立连接。当然为了降低重试机制的开销，我们加入了休眠机制，即每次重试的时间间隔为1000毫秒。

``` java
        Socket socket = null;
        while (socket == null) {
            try {
                socket = new Socket("localhost", 8688);
                mClientSocket = socket;
                mPrintWriter = new PrintWriter(new BufferedWriter(
                        new OutputStreamWriter(socket.getOutputStream())), true);
                mHandler.sendEmptyMessage(MESSAGE_SOCKET_CONNECTED);
                System.out.println("connect server success");
            } catch (IOException e) {
                SystemClock.sleep(1000);
                System.out.println("connect tcp server failed, retry...");
            }
        }
```
服务端连接成功以后，就可以和服务端进行通信了。下面的代码在线程中通过while循环不断地去读取服务端发送过来的消息，同时当Activity退出时，就退出循环并终止线程。

``` java
        try {
            // 接收服务器端的消息
            BufferedReader br = new BufferedReader(new InputStreamReader(
                    socket.getInputStream()));
            while (!TCPClientActivity.this.isFinishing()) {
                String msg = br.readLine();
                System.out.println("receive :" + msg);
                if (msg != null) {
                    String time = formatDateTime(System.currentTimeMillis());
                    final String showedMsg = "server " + time + ":" + msg + "\n";
                    mHandler.obtainMessage(MESSAGE_RECEIVE_NEW_MSG, showedMsg).sendToTarget();
                }
            }
            System.out.println("quit...");
            MyUtils.close(mPrintWriter);
            MyUtils.close(br);
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

```
同时，当Activity退出时，还要关闭当前的Socket，如下所示。

``` java
    @Override
    protected void onDestroy() {
        if (mClientSocket != null) {
            try {
                mClientSocket.shutdownInput();
                mClientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        super.onDestroy();
    }
```

接着是发送消息的过程，这个就很简单了，这里不再详细说明。客户端的完整代码如下：

``` java

public class TCPClientActivity extends Activity implements OnClickListener {
    private static final int MESSAGE_RECEIVE_NEW_MSG = 1;
    private static final int MESSAGE_SOCKET_CONNECTED = 2;
    private Button mSendButton;
    private TextView mMessageTextView;
    private EditText mMessageEditText;
    private PrintWriter mPrintWriter;
    private Socket mClientSocket;
    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case MESSAGE_RECEIVE_NEW_MSG: {
                mMessageTextView.setText(mMessageTextView.getText() + (String) msg.obj);
                break;
            }
            case MESSAGE_SOCKET_CONNECTED: {
                mSendButton.setEnabled(true);
                break;
            }
            default:
                break;
            }
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tcpclient);
        mMessageTextView = (TextView) findViewById(R.id.msg_container);
        mSendButton = (Button) findViewById(R.id.send);
        mSendButton.setOnClickListener(this);
        mMessageEditText = (EditText) findViewById(R.id.msg);
        Intent service = new Intent(this, TCPServerService.class);
        startService(service);
        new Thread() {
            @Override
            public void run() {
                connectTCPServer();
            }
        }.start();
    }
    @Override
    protected void onDestroy() {
        if (mClientSocket != null) {
            try {
                mClientSocket.shutdownInput();
                mClientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        super.onDestroy();
    }
    @Override
    public void onClick(View v) {
        if (v == mSendButton) {
            final String msg = mMessageEditText.getText().toString();
            if (!TextUtils.isEmpty(msg) && mPrintWriter != null) {
                new Thread(){
                    @Override
                    public void run() {
                        mPrintWriter.println(msg);
                    }
                }.start();
                mMessageEditText.setText("");
                String time = formatDateTime(System.currentTimeMillis());
                final String showedMsg = "self " + time + ":" + msg + "\n";
                mMessageTextView.setText(mMessageTextView.getText() + showedMsg);
            }
        }
    }
    @SuppressLint("SimpleDateFormat")
    private String formatDateTime(long time) {
        return new SimpleDateFormat("(HH:mm:ss)").format(new Date(time));
    }
    private void connectTCPServer() {
        Socket socket = null;
        while (socket == null) {
            try {
                socket = new Socket("localhost", 8688);
                mClientSocket = socket;
                mPrintWriter = new PrintWriter(new BufferedWriter(
                        new OutputStreamWriter(socket.getOutputStream())), true);
                mHandler.sendEmptyMessage(MESSAGE_SOCKET_CONNECTED);
                System.out.println("connect server success");
            } catch (IOException e) {
                SystemClock.sleep(1000);
                System.out.println("connect tcp server failed, retry...");
            }
        }
        try {
            // 接收服务器端的消息
            BufferedReader br = new BufferedReader(new InputStreamReader(
                    socket.getInputStream()));
            while (!TCPClientActivity.this.isFinishing()) {
                String msg = br.readLine();
                System.out.println("receive :" + msg);
                if (msg != null) {
                    String time = formatDateTime(System.currentTimeMillis());
                    final String showedMsg = "server " + time + ":" + msg + "\n";
                    mHandler.obtainMessage(MESSAGE_RECEIVE_NEW_MSG, showedMsg).sendToTarget();
                }
            }
            System.out.println("quit...");
            MyUtils.close(mPrintWriter);
            MyUtils.close(br);
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

上述就是通过Socket来进行进程间通信的实例，除了采用TCP套接字，还可以采用UDP套接字。另外，上面的例子仅仅是一个示例，实际上通过Socket不仅仅能实现进程间的通信，还可以实现设备间的通信，当然前提是这些设备之间的IP地址互相可见，这其中又涉及许多复杂的概念，这里就不一一介绍了。下面看一下上述例子的运行效果，如图。

![](https://sweetm-1256061026.cos.ap-beijing.myqcloud.com/blog/20190326225838.png)

## 七、Binder 连接池


如何使用AIDL在上面已经进行了介绍，这里在回顾一下大致流程：首先创建一个Service和一个AIDL接口，接着创建一个类继承自AIDL接口中的Stub类并实现Stub中的抽象方法，在Service的onBind方法中返回这个类的对象，然后客户端就可以绑定服务端Service，建立连接后就可以访问远程服务端的方法了。

上述过程就是典型的AIDL的使用流程。这本来也没什么问题，但是现在考虑一种情况：公司的项目越来越庞大了，现在有10个不同的业务模块都需要使用AIDL来进行进程间通信，那我们该怎么处理呢？也许你会说：“就按照AIDL的实现方式一个个来吧”，这是可以的，如果用这种方法，首先我们需要创建10个Service，这好像有点多啊！如果有100个地方需要用到AIDL呢，先创建100个Service？到这里，读者应该明白问题所在了。随着AIDL数量的增加，我们不能无限制地增加Service，Service是四大组件之一，本身就是一种系统资源。而且太多的Service会使得我们的应用看起来很重量级，因为正在运行的Service可以在应用详情页看到，当我们的应用详情显示有10个服务正在运行时，这看起来并不是什么好事。针对上述问题，我们需要减少Service的数量，将所有的AIDL放在同一个Service中去管理。
                                                                               在这种模式下，整个工作机制是这样的：每个业务模块创建自己的AIDL接口并实现此接口，这个时候不同业务模块之间是不能有耦合的，所有实现细节我们要单独开来，然后向服务端提供自己的唯一标识和其对应的Binder对象；对于服务端来说，只需要一个Service就可以了，服务端提供一个queryBinder接口，这个接口能够根据业务模块的特征来返回相应的Binder对象给它们，不同的业务模块拿到所需的Binder对象后就可以进行远程方法调用了。由此可见，Binder连接池的主要作用就是将每个业务模块的Binder 请求统一转发到远程Service中去执行，从而避免了重复创建Service的过程，它的工作原理如图2-10所示。

![](https://sweetm-1256061026.cos.ap-beijing.myqcloud.com/blog/20190327104946.png)


通过上面的理论介绍，也许还有点不好理解，下面对Binder连接池的代码实现做一下说明。首先，为了说明问题，我们提供了两个AIDL接口（ISecurityCenter和ICompute）来模拟上面提到的多个业务模块都要使用AIDL的情况，其中ISecurityCenter接口提供加解密功能，声明如下：

``` java
interface ISecurityCenter {
    String encrypt(String content);
    String decrypt(String password);
}
```

而ICompute接口提供计算加法的功能，声明如下：

``` java
interface ICompute {
    int add(int a, int b);
}
```

虽然说上面两个接口的功能都比较简单，但是用于分析Binder连接池的工作原理已经足够了，读者可以写出更复杂的例子。接着看一下上面两个AIDL接口的实现，也比较简单，代码如下：

``` java
public class SecurityCenterImpl extends ISecurityCenter.Stub {
    private static final char SECRET_CODE = '^';
    @Override
    public String encrypt(String content) throws RemoteException {
        char[] chars = content.toCharArray();
        for (int i = 0; i < chars.length; i++) {
            chars[i] ^= SECRET_CODE;
        }
        return new String(chars);
    }
    @Override
    public String decrypt(String password) throws RemoteException {
        return encrypt(password);
    }
}
```

现在业务模块的AIDL接口定义和实现都已经完成了，注意这里并没有为每个模块的AIDL单独创建Service，接下来就是服务端和Binder连接池的工作了。首先，为Binder连接池创建AIDL接口IBinderPool.aidl，代码如下所示。

``` java
interface IBinderPool {
    /**
     * @param binderCode, the unique token of specific Binder<br/>
     * @return specific Binder who's token is binderCode.
     */
    IBinder queryBinder(int binderCode);
}

```

接着，为Binder连接池创建远程Service并实现IBinderPool，下面是queryBinder的具体实现，可以看到请求转发的实现方法，当Binder连接池连接上远程服务时，会根据不同模块的标识即binderCode返回不同的Binder对象，通过这个Binder对象所执行的操作全部发生在远程服务端。

``` java
    public static class BinderPoolImpl extends IBinderPool.Stub {
        public BinderPoolImpl() {
            super();
        }
        @Override
        public IBinder queryBinder(int binderCode) throws RemoteException {
            IBinder binder = null;
            switch (binderCode) {
            case BINDER_SECURITY_CENTER: {
                binder = new SecurityCenterImpl();
                break;
            }
            case BINDER_COMPUTE: {
                binder = new ComputeImpl();
                break;
            }
            default:
                break;
            }
            return binder;
        }
    }

```

远程Service的实现就比较简单了，代码如下所示。

``` java
public class BinderPoolService extends Service {
    private static final String TAG = "BinderPoolService";
    private Binder mBinderPool = new BinderPool.BinderPoolImpl();
    @Override
    public void onCreate() {
        super.onCreate();
    }
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind");
        return mBinderPool;
    }
    @Override
    public void onDestroy() {
        super.onDestroy();
    }
```

下面还剩下Binder连接池的具体实现，在它的内部首先它要去绑定远程服务，绑定成功后，客户端就可以通过它的queryBinder方法去获取各自对应的Binder，拿到所需的Binder以后，不同业务模块就可以进行各自的操作了，Binder连接池的代码如下所示。

``` java
public class BinderPool {
    private static final String TAG = "BinderPool";
    public static final int BINDER_NONE = -1;
    public static final int BINDER_COMPUTE = 0;
    public static final int BINDER_SECURITY_CENTER = 1;
    private Context mContext;
    private IBinderPool mBinderPool;
    private static volatile BinderPool sInstance;
    private CountDownLatch mConnectBinderPoolCountDownLatch;
    private BinderPool(Context context) {
        mContext = context.getApplicationContext();
        connectBinderPoolService();
    }
    public static BinderPool getInsance(Context context) {
        if (sInstance == null) {
            synchronized (BinderPool.class) {
                if (sInstance == null) {
                    sInstance = new BinderPool(context);
                }
            }
        }
        return sInstance;
    }
    private synchronized void connectBinderPoolService() {
        //等待线程的数量
        mConnectBinderPoolCountDownLatch = new CountDownLatch(1);
        Intent service = new Intent(mContext, BinderPoolService.class);
        mContext.bindService(service, mBinderPoolConnection,Context.BIND_AUTO_CREATE);
        try {
            mConnectBinderPoolCountDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    /**
     * query binder by binderCode from binder pool
     *
     * @param binderCode
     *            the unique token of binder
     * @return binder who's token is binderCode<br>
     *         return null when not found or BinderPoolService died.
     */
    public IBinder queryBinder(int binderCode) {
        IBinder binder = null;
        try {
            if (mBinderPool != null) {
                binder = mBinderPool.queryBinder(binderCode);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return binder;
    }
    private ServiceConnection mBinderPoolConnection = new ServiceConnection() {
        @Override
        public void onServiceDisconnected(ComponentName name) {
            // ignored.
        }
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBinderPool = IBinderPool.Stub.asInterface(service);
            try {
                mBinderPool.asBinder().linkToDeath(mBinderPoolDeathRecipient, 0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            mConnectBinderPoolCountDownLatch.countDown();
        }
    };
    private IBinder.DeathRecipient mBinderPoolDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            Log.w(TAG, "binder died.");
            mBinderPool.asBinder().unlinkToDeath(mBinderPoolDeathRecipient, 0);
            mBinderPool = null;
            connectBinderPoolService();
        }
    };
    public static class BinderPoolImpl extends IBinderPool.Stub {
        public BinderPoolImpl() {
            super();
        }
        @Override
        public IBinder queryBinder(int binderCode) throws RemoteException {
            IBinder binder = null;
            switch (binderCode) {
            case BINDER_SECURITY_CENTER: {
                binder = new SecurityCenterImpl();
                break;
            }
            case BINDER_COMPUTE: {
                binder = new ComputeImpl();
                break;
            }
            default:
                break;
            }
            return binder;
        }
    }
}

```
Binder连接池的具体实现就分析完了，它的好处是显然易见的，针对上面的例子，我们只需要创建一个Service即可完成多个AIDL接口的工作，下面我们来验证一下效果。新创建一个Activity，在线程中执行如下操作：

``` java
    private void doWork() {
        BinderPool binderPool = BinderPool.getInsance(BinderPoolActivity.this);
        IBinder securityBinder = binderPool.queryBinder(BinderPool.BINDER_SECURITY_CENTER);
        mSecurityCenter = (ISecurityCenter) SecurityCenterImpl.asInterface(securityBinder);
        Log.d(TAG, "visit ISecurityCenter");
        String msg = "helloworld-安卓";
        System.out.println("content:" + msg);
        try {
            String password = mSecurityCenter.encrypt(msg);
            System.out.println("encrypt:" + password);
            System.out.println("decrypt:" + mSecurityCenter.decrypt(password));
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "visit ICompute");
        IBinder computeBinder = binderPool.queryBinder(BinderPool.BINDER_COMPUTE);
        mCompute = ComputeImpl.asInterface(computeBinder);
        try {
            System.out.println("3+5=" + mCompute.add(3, 5));
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
```

可以看到日志
``` logs
2019-03-27 14:15:16.006 4043-4236/com.ryg.chapter_2 D/BinderPoolActivity: visit ISecurityCenter
2019-03-27 14:15:16.006 4043-4236/com.ryg.chapter_2 I/System.out: content:helloworld-安卓
2019-03-27 14:15:16.007 4043-4236/com.ryg.chapter_2 I/System.out: encrypt:6;221)1,2:s寗匍
2019-03-27 14:15:16.008 4043-4236/com.ryg.chapter_2 I/System.out: decrypt:helloworld-安卓
2019-03-27 14:15:16.009 4043-4236/com.ryg.chapter_2 D/BinderPoolActivity: visit ICompute
2019-03-27 14:15:16.025 4043-4236/com.ryg.chapter_2 I/System.out: 3+5=8
2019-03-27 14:15:16.197 4043-4094/com.ryg.chapter_2 D/EGL_emulation: eglMakeCurrent: 0x96af8180: ver 3 0 (tinfo 0xa61ff580)
```

这里需要额外说明一下，为什么要在线程中去执行呢？这是因为在Binder连接池的实现中，我们通过CountDownLatch将bindService这一异步操作转换成了同步操作，这就意味着它有可能是耗时的，然后就是Binder方法的调用过程也可能是耗时的，因此不建议放在主线程去执行。注意到BinderPool是一个单例实现，因此在同一个进程中只会初始化一次，所以如果我们提前初始化BinderPool，那么可以优化程序的体验，比如我们可以放在Application中提前对BinderPool进行初始化，虽然这不能保证当我们调用BinderPool时它一定是初始化好的，但是在大多数情况下，这种初始化工作（绑定远程服务）的时间开销（如果BinderPool没有提前初始化完成的话）是可以接受的。另外，BinderPool中有断线重连的机制，当远程服务意外终止时，BinderPool会重新建立连接，这个时候如果业务模块中的Binder 调用出现了异常，也需要手动去重新获取最新的Binder对象，这个是需要注意的。

有了BinderPool可以大大方便日常的开发工作，比如如果有一个新的业务模块需要添加新的AIDL，那么在它实现了自己的AIDL接口后，只需要修改BinderPoollmpl中的queryBinder方法，给自己添加一个新的binderCode并返回对应的Binder对象即可，不需要做其他修改，也不需要创建新的Service。由此可见，BinderPool能够极大地提高AIDL的开发效率，并且可以避免大量的Service创建，因此，建议在AIDL开发工作中引入BinderPool机制。

## 八、各个IPC方式优缺点

|      名称       |                             优点                             |                             缺点                             |                           适用场景                           |
| :-------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|     Bundle      |                           简单易用                           |                   只能传输Bundle支持的数据                   |                     四大组件间的进行通信                     |
|    文件共享     |                           简单易用                           |        不适合高并发场景，并且无法做到进程间的及时通信        |        无并发访问情形，交换简单的数据实时性不高的场景        |
|      AIDL       |          功能强大，支持一对多并发通信，支持实时通信          |               使用稍微复杂，需要处理号线程同步               |                    一对多通信且有RPC需求                     |
|    Messenger    |          功能一般，支持一对多串行通信，支持实时通信          | 不能很好处理高并发情形，不支持RPC,数据通过Message进行传输，因此只能传输Bundle支持的数据类型 | 低并发的一对多及时通信，无RPC需求，或者无须要返回结果的RPC需求 |
| ContentProvider | 在数据源访问方面功能强大，支持一对多并发数据共享，可以通过Call方法进行扩展其他操作 |       可以理解为受约束的AIDL,主要提供数据源的CURD操作        |                   一对多的进程间的数据共享                   |
|     Socket      |   功能强大，可以通过网络传输字节流，支持一对多并发实时通信   |            实现细节稍微有点繁琐，不支持直接的RPC             |                         网络数据交换                         |

