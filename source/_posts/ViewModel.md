---
title: ViewModel源码剖析
date: 2021-09-19 18:44:37
categories:
- [Android]
tags: ViewModel源码剖析, Android, ViewModel
---



# ViewModel 源码剖析



# 示例代码

ViewModel负责数据存储，借助LiveData，实现异步获取数据，完成后，通知UI观察者，更新UI

```java
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<User>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // Do an asynchronous operation to fetch users.
				List<User> requestResult = requestUsers();
				// Once value changes, observer will be notified.
				users.set(requestResult);
    }
}
```

UI层注册数据变更监听。注意ViewModelProvider构造器参数是哪个Activity，Fragment实例，ViewModel实例化后，其引用寄存在该实例内。他们都实现ViewModelStoreOwner接口。

```java
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.
				// 1. ViewModel实例挂载在this实例里，有点相当于是this实例的成员变量
        MyViewModel model = new ViewModelProvider(this).get(MyViewModel.class);
        // 2. 这里的this参数，决定观察者的生命周期。注意，如果此参数是activity实例，而当前类是fragment，则fragment实例内存泄漏
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```



注意，observe这里第一个参数，决定观察者的生命周期。所以，如下示例代码，此参数是activity实例，而当前类是fragment，则fragment实例内存泄漏，因为监听实例生命周期和Activity一致，而它持有Fragment实例引用，导致Fragment在Activity生命周期内无法被回收。

```java
public class MyFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.
        MyViewModel model = new ViewModelProvider(this).get(MyViewModel.class);
        
        model.getUsers().observe(requireActivity(), users -> {
            // update UI
        });
    }
}
```





# 源码剖析

先从ViewModel实例化开始，首先要解决ViewModel如何实例化，并存放实例，便后续获取实例。

```java
androidx.lifecycle.ViewModelProvider.java
/**
 * Creates {@code ViewModelProvider}. This will create {@code ViewModels}
 * and retain them in a store of the given {@code ViewModelStoreOwner}.
 * <p>
 * This method will use the
 * {@link HasDefaultViewModelProviderFactory#getDefaultViewModelProviderFactory() default factory}
 * if the owner implements {@link HasDefaultViewModelProviderFactory}. Otherwise, a
 * {@link NewInstanceFactory} will be used.
 */
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}

/**
 * Creates {@code ViewModelProvider}, which will create {@code ViewModels} via the given
 * {@code Factory} and retain them in a store of the given {@code ViewModelStoreOwner}.
 *
 * @param owner   a {@code ViewModelStoreOwner} whose {@link ViewModelStore} will be used to
 *                retain {@code ViewModels}
 * @param factory a {@code Factory} which will be used to instantiate
 *                new {@code ViewModels}
 */
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
    this(owner.getViewModelStore(), factory);
}

/**
 * Creates {@code ViewModelProvider}, which will create {@code ViewModels} via the given
 * {@code Factory} and retain them in the given {@code store}.
 *
 * @param store   {@code ViewModelStore} where ViewModels will be stored.
 * @param factory factory a {@code Factory} which will be used to instantiate
 *                new {@code ViewModels}
 */
public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}


```

Activity，Fragment实例，他们都实现ViewModelStoreOwner接口，看这名字，顾名思义，就是提供存放ViewModel实例的地方。同时，ViewModelStoreOwner 可能自带ViewModel构造工厂，没有的话，就使用ViewModelProvider 内的NewInstanceFactory实例。

ViewModelStoreOwner 返回ViewModelStore，内部就是一个Map，key是`ViewModelProvider(this).get(MyViewModel.class);`get方法传入ViewModel class的canonicalName，value就是对应的ViewModel实例啦。Activity 或者 Fragment 持有ViewModelStore引用。

```java
androidx.lifecycle.ViewModelProvider
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
    } else {
        viewModel = mFactory.create(modelClass);
    }
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```

总结下，ViewModelStoreOwner 提供负责存储ViewModel的类：ViewModelStore，ViewModel由Factory创建，然后放在ViewModelStore实例内，下次直接从ViewModelStore 通过get方法获取，单例。



![ViewModel](https://s3.bmp.ovh/imgs/2021/09/8a107bebc30102c8.png)

# AndroidViewModel & ViewModel

ViewModel 不可以强引用Context，如果确实需要，则使用AndroidViewModel，它持有Application的引用。解决需要的同时，避免Activity泄漏。

`ViewModelProvider` 默认实例化的ViewModel 由NewInstanceFactory 创建。AndroidViewModel 由AndroidViewModelFactory 通过作为实例化`ViewModelProvider` 构造器的Factory 参数创建。

Activity & Fragment 也同时实现`HasDefaultViewModelProviderFactory`接口。看下`SavedStateViewModelFactory` ，区分是实例化 ViewModel 还是AndroidViewModel，并增加保存状态的处理。

```java
androidx.lifecycle.SavedStateViewModelFactory
@NonNull
@Override
public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
    boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
    Constructor<T> constructor;
    if (isAndroidViewModel && mApplication != null) {
        constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
    } else {
        constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
    }
    // doesn't need SavedStateHandle
    if (constructor == null) {
        return mFactory.create(modelClass);
    }

    SavedStateHandleController controller = SavedStateHandleController.create(
            mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
    try {
        T viewmodel;
        if (isAndroidViewModel && mApplication != null) {
            viewmodel = constructor.newInstance(mApplication, controller.getHandle());
        } else {
            viewmodel = constructor.newInstance(controller.getHandle());
        }
        viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
        return viewmodel;
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Failed to access " + modelClass, e);
    } catch (InstantiationException e) {
        throw new RuntimeException("A " + modelClass + " cannot be instantiated.", e);
    } catch (InvocationTargetException e) {
        throw new RuntimeException("An exception happened in constructor of "
                + modelClass, e.getCause());
    }
}
```

ViewModel就是一个壳，与ViewModelStoreOwner共存亡。但Activity因onConfigurationChanged重建后，会获取上一个Activity ViewModelStore，恢复上一个ViewModel。

```java
androidx.activity.ComponentActivity
void ensureViewModelStore() {
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
}
```

# OnRequeryFactory 接口

用途：提供从ViewModelProvider get 返回ViewModel实例前的回调，可以在返回前对ViewModel做些预处理。SavedStateViewModelFactory实现此接口。

```java
androidx.lifecycle.SavedStateViewModelFactory
@Override
void onRequery(@NonNull ViewModel viewModel) {
    attachHandleIfNeeded(viewModel, mSavedStateRegistry, mLifecycle);
}
```

```java
androidx.lifecycle.ViewModelProvider
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
				// ViewModel返回前处理
        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
    } else {
        viewModel = mFactory.create(modelClass);
    }
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```


