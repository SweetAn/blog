---
title: MVVM之ViewModel
date: 2018-02-26 20:27:57
tags: 
- MVVM
categories: 
- Android
- 框架
---

谷歌网址
https://developer.android.com/topic/libraries/architecture/viewmodel#java

# ViewModel

The ViewModel class is designed to store and manage UI-related data in a lifecycle conscious way. The ViewModel class allows data to survive configuration changes such as screen rotations.
>ViewModel类旨在以生命周期意识的方式存储和管理与UI相关的数据。 ViewModel类允许数据在配置更改（例如屏幕旋转）后继续存在。

The Android framework manages the lifecycles of UI controllers, such as activities and fragments. The framework may decide to destroy or re-create a UI controller in response to certain user actions or device events that are completely out of your control.

If the system destroys or re-creates a UI controller, any transient UI-related data you store in them is lost. For example, your app may include a list of users in one of its activities. When the activity is re-created for a configuration change, the new activity has to re-fetch the list of users. For simple data, the activity can use the onSaveInstanceState() method and restore its data from the bundle in onCreate(), but this approach is only suitable for small amounts of data that can be serialized then deserialized, not for potentially large amounts of data like a list of users or bitmaps.

Another problem is that UI controllers frequently need to make asynchronous calls that may take some time to return. The UI controller needs to manage these calls and ensure the system cleans them up after it's destroyed to avoid potential memory leaks. This management requires a lot of maintenance, and in the case where the object is re-created for a configuration change, it's a waste of resources since the object may have to reissue calls it has already made.

UI controllers such as activities and fragments are primarily intended to display UI data, react to user actions, or handle operating system communication, such as permission requests. Requiring UI controllers to also be responsible for loading data from a database or network adds bloat to the class. Assigning excessive responsibility to UI controllers can result in a single class that tries to handle all of an app's work by itself, instead of delegating work to other classes. Assigning excessive responsibility to the UI controllers in this way also makes testing a lot harder.

It's easier and more efficient to separate out view data ownership from UI controller logic.

## The lifecycle of a ViewModel  生命周期

ViewModel objects are scoped to the Lifecycle passed to the ViewModelProvider when getting the ViewModel. The ViewModel remains in memory until the Lifecycle it's scoped to goes away permanently: in the case of an activity, when it finishes, while in the case of a fragment, when it's detached.

ViewModel的生命周期在activity中时当onDestory之后结束，在fragment时在 detached之后结束。

Figure 1 illustrates （插图）the various lifecycle states of an activity as it undergoes（经历） a rotation and then is finished. The illustration also shows the lifetime of the ViewModel next to the associated activity lifecycle. This particular diagram illustrates the states of an activity. The same basic states apply to the lifecycle of a fragment.

![viewmodel-lifecycle](assets/viewmodel-lifecycle.png)


## Share data between fragments  分享数据在fragment之间

It's very common that two or more fragments in an activity need to communicate with each other. Imagine(想象一下) a common case of master-detail fragments, where you have a fragment in which the user selects an item from a list and another fragment that displays the contents of the selected item. This case is never trivial(不重要的) as both fragments need to define some interface description, and the owner activity must bind the two together. In addition, both fragments must handle the scenario(场景) where the other fragment is not yet created or visible.

This common pain point can be addressed by using ViewModel objects. These fragments can share a ViewModel using their activity scope to handle this communication, as illustrated by the following sample code:

``` java
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}


public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, { item ->
           // Update the UI.
        });
    }
}
```

Notice that both fragments retrieve the activity that contains them. That way, when the fragments each get the ViewModelProvider, they receive the same SharedViewModel instance, which is scoped to this activity.

This approach offers the following benefits（好处）:

* The activity does not need to do anything, or know anything about this communication.
* Fragments don't need to know about each other besides the SharedViewModel contract. If one of the fragments disappears, the other one keeps working as usual.
* Each fragment has its own lifecycle, and is not affected by the lifecycle of the other one. If one fragment replaces the other one, the UI continues to work without any problems.

## Replacing Loaders with ViewModel  替换加载器

Loader classes like CursorLoader are frequently used to keep the data in an app's UI in sync with a database. You can use ViewModel, with a few other classes, to replace the loader. Using a ViewModel separates your UI controller from the data-loading operation, which means you have fewer strong references between classes.

In one common approach to using loaders, an app might use a CursorLoader to observe the contents of a database. When a value in the database changes, the loader automatically triggers a reload of the data and updates the UI:

![viewmodel-loader](assets/viewmodel-loader.png)

Figure 2. Loading data with loaders
ViewModel works with Room and LiveData to replace the loader. The ViewModel ensures that the data survives a device configuration change. Room informs your LiveData when the database changes, and the LiveData, in turn, updates your UI with the revised data.

![viewmodel-replace-loader](assets/viewmodel-replace-loader.png)

Figure 3. Loading data with ViewModel