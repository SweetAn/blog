---
title: MVVM之LiveData介绍
date:  2018-02-26 21:27:57
tags:
---

# LiveData 


[原文网址](https://developer.android.com/topic/libraries/architecture/livedata#the_advantages_of_using_livedata)

LiveData is an observable data holder class. Unlike a regular observable, LiveData is lifecycle-aware(生命周期), meaning it respects the lifecycle of other app components, such as activities, fragments, or services. This awareness ensures LiveData only updates app component observers that are in an active lifecycle state.

LiveData considers an observer, which is represented(代表) by the Observer class, to be in an active state if its lifecycle is in the STARTED or RESUMED state. LiveData only notifies active observers about updates. Inactive observers registered to watch LiveData objects aren't notified about changes.

You can register an observer paired with an object that implements the LifecycleOwner interface. This relationship allows the observer to be removed when the state of the corresponding Lifecycle object changes to DESTROYED. This is especially useful for activities and fragments because they can safely observe LiveData objects and not worry about leaks—activities and fragments are instantly unsubscribed when their lifecycles are destroyed.

> 翻译： 您可以注册与实现LifecycleOwner接口的对象配对的观察者。 此关系允许在相应Lifecycle对象的状态更改为DESTROYED时删除观察者。 这对于活动和片段特别有用，因为它们可以安全地观察LiveData对象而不用担心泄漏 - 活动和片段在其生命周期被破坏时立即取消订阅。

## The advantages of using LiveData
Using LiveData provides the following advantages:

### Ensures （确保）your UI matches your data state

LiveData follows the observer pattern. LiveData notifies Observer objects when the lifecycle state changes. You can consolidate  your code to update the UI in these Observer objects. Instead of updating the UI every time the app data changes, your observer can update the UI every time there's a change.

>LiveData遵循观察者模式。 生命周期状态更改时，LiveData会通知Observer对象。 您可以合并代码以更新这些Observer对象中的UI。 每次应用程序数据更改时，您的观察者都可以在每次更改时更新UI，而不是更新UI每时每刻。

### No memory leaks
Observers are bound to Lifecycle objects and clean up after themselves when their associated lifecycle is destroyed.

### No crashes due to stopped activities
If the observer's lifecycle is inactive, such as in the case of an activity in the back stack, then it doesn’t receive any LiveData events.

### No more manual lifecycle handling
UI components just observe relevant（有关） data and don’t stop or resume observation. LiveData automatically manages all of this since it’s aware of the relevant lifecycle status changes while observing.

> UI组件只是观察相关（有关）数据，不会停止或恢复观察。 LiveData自动管理所有这些，因为它在观察时意识到相关的生命周期状态变化。

### Always up to date data
If a lifecycle becomes inactive(不活跃的), it receives the latest data upon becoming active again. For example, an activity that was in the background receives the latest data right after it returns to the foreground.
>如果生命周期变为非活动状态，它将在再次变为活动状态时接收最新数据。 例如，后台活动在返回前台后立即接收最新数据。

