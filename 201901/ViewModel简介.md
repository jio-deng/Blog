### ViewModel简介

与上一篇文章相同，先官方文档翻译，再源码分析～

我们先看一下[官方文档][1]的介绍：

>The ViewModel class is designed to store and manage UI-related data in a lifecycle conscious way. The ViewModel class allows data to survive configuration changes such as screen rotations.
ViewModel是被设计成在Lifecycle中存储和管理UI相关数据的类。ViewModel使得数据在屏幕旋转之类的配置改变的情形下不被销毁。

Android frameworl层管理着UI控制器(e.g. activity, fragment)的生命周期。framework可能会因为某些用户操作或者是系统事件而直接销毁或重建UI controller，这类事件是在控制之外的。如果出现这种情况，带有transient标志的数据都会丢失。

另一个问题是在UI controller中经常有耗时的异步请求。UI controller需要管理这些请求，保证请求结束后销毁它们，来避免潜在的内存泄漏。这种管理需要一直维护，而且在UI controller因为配置改变而重建时，可能会重新发送之前发送过的请求，浪费资源。

UI controller的主要任务是展示数据、处理用户交互、处理系统会话（例如权限请求）。要求controller也负责从数据库或网络中获取数据，使得controller变得臃肿。将数据的管理从UI controller中分离出来很简单，且更加高效。

---

##### 实现ViewModel

ViewModel在activity或fragment旋转时会自动保存，保存的数据可以立即提供给重新创建的UI controller。

```
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
    }
}

public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```

因为ViewModel被设计成生命周期超过view实例和LifecycleOwner，因此有如下注意：

**Caution: A ViewModel must never reference a view, Lifecycle, or any class that may hold a reference to the activity context.**

如果ViewModel中需要使用context，可以继承AndroidViewModel，构造函数中可以传入Application对象。

```
public class AndroidViewModel extends ViewModel {
    @SuppressLint("StaticFieldLeak")
    private Application mApplication;

    public AndroidViewModel(@NonNull Application application) {
        mApplication = application;
    }

    /**
     * Return the application.
     */
    @SuppressWarnings("TypeParameterUnusedInFormals")
    @NonNull
    public <T extends Application> T getApplication() {
        //noinspection unchecked
        return (T) mApplication;
    }
}
```

##### ViewModel的生命周期

获取ViewModel对象时，它的生命周期被限定为ViewModelProvider传入的Lifecycle。ViewModel对象会被存储在内存中，直到它被限定的Lifecycle对象被永久销毁（e.g. activity-finish,fragment-detached）。

（有道云笔记无法插图-此处插图！！！）

ViewModel对象在第一次获取时创建，并一直存在，直到UI controller结束或销毁。

##### fragments间数据传递

fragments之间的数据传递很常见，一般通过实现、调用接口的方式，通过activity传递数据;而现在可以使用同一个activity中ViewModel。假设有两个fragment，一个是列表展示，点击后在另一个fragment中展示详情。


```
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
        // same UI controller, same ViewModel
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

优点：

1.activity不需要了解fragment中的任何操作。

2.fragment只需要关注SharedViewModel，而不需要关注activity和其他的fragment。即使另一个fragment消失，也可以继续运行

3.每个fragment拥有自己的生命周期，替换fragment也可以正常运行。

##### 替代Loader类

像CursorLoader这样的加载器类经常用于使应用程序UI中的数据与数据库保持同步。使用ViewModel将数据加载操作和UI controller分开，类之间少了许多强引用。

使用Loader类：可能会使用CursorLoader去观察数据库;当数据库中数据发生改变时，loader触发一次读库并更新UI。

（有道云笔记无法插图-此处插图！！！）

使用ViewModel类：ViewModel使用Room和LiveData来代替loader。ViewModel确保数据能够不因配置改变而销毁;Room在数据库发生数据改变时通知LiveData，LiveData自动更新UI。

（有道云笔记无法插图-此处插图！！！）

---

### ViewModel源码分析

从上文的代码中我们可以看到ViewModel的创建过程：

```
ViewModelProviders.of(activity).get(XxxViewModel.class);
```

以Activity为例，先看of(FragmentActivity activity)函数：


```
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
    return of(activity, null);
}
    
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    // 获取Application对象
    Application application = checkApplication(activity);
    if (factory == null) {
        // 创建AndroidViewModelFactory
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(activity.getViewModelStore(), factory);
}
```

of函数中最后返回的是ViewModelProvider对象。我们先看factory：factory的创建过程是一个单例模式，AndroidViewModelFactory中实现了create方法


```
@NonNull
@Override
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
        //noinspection TryWithIdenticalCatches
        try {
            // 通过反射的方式创建对应ViewModel对象
            return modelClass.getConstructor(Application.class).newInstance(mApplication);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
    return super.create(modelClass);
}
```

而ViewModelProvider的创建只是将两个参数保存了起来。这两部就是of的主要工作。

接下来看get方法中做了什么：


```
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        // 检测对应ViewModel类是否存在
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    // ViewModelStore中维护了一个HashMap来保存ViewModel
    ViewModel viewModel = mViewModelStore.get(key);

    // 当ViewModel存在时，返回
    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }

    // 当ViewModel不存在时，调用of中保存的factory的create函数，利用反射创建一个ViewModel实例。并放入ViewModelStore中
    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

[1]:https://developer.android.google.cn/topic/libraries/architecture/viewmodel#java