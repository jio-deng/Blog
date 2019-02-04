### LiveData简介

我们先来看一下[官方网站][1]给出的定义：

>LiveData is an observable data holder class. Unlike a regular observable, LiveData is lifecycle-aware, meaning it respects the lifecycle of other app components, such as activities, fragments, or services. This awareness ensures LiveData only updates app component observers that are in an active lifecycle state.
LiveData是可观察的数据容器里。不同于普通的被观察者，LiveData是可预见生命周期的，意味着它不会对其他App组件的生命周期造成影响，例如activities、fragments或者services。这种对生命周期的可见性保证了LiveData只会在当前App组件（观察者）处于active生命周期状态时（STARTED和RESUMED）进行通知。

>LiveData considers an observer, which is represented by the Observer class, to be in an active state if its lifecycle is in the STARTED or RESUMED state. LiveData only notifies active observers about updates. Inactive observers registered to watch LiveData objects aren't notified about changes.
LiveData认为当观察者处于STARTED或RESUMED状态时，是处于active状态。LiveData只会在观察者处于active状态时通知updates。inactive状态下的观察者尽管注册了也不会得到改动通知。

>You can register an observer paired with an object that implements the LifecycleOwner interface. This relationship allows the observer to be removed when the state of the corresponding Lifecycle object changes to DESTROYED. This is especially useful for activities and fragments because they can safely observe LiveData objects and not worry about leaks—activities and fragments are instantly unsubscribed when their lifecycles are destroyed.
你可以注册一个 与实现了LifecycleOwner接口的对象配对的 观察者。这种关系可以保证当对应的Lifecycle对象进入DESTROYED状态时，可以移除观察者。这对于activities和fragments尤其实用，因为这样它们可以安全地观察LiveData对象，而不用担心内存泄漏-activities和fragments在被destroy时会立即取消订阅。

---

### LiveData的优势

##### 确保你的UI显示与数据一致

LiveData遵循了观察者模式，当生命周期状态变化时，LiveData会通知观察者。你可以将更新观察者UI的代码统一起来。观察者可以在任何时间任意改变时更新UI，而不仅时在app数据改变时才更新UI。


##### 无内存泄露

观察者绑定在Lifecycle上，并在相关联的Lifecycle对象被销毁时自己clean up。

##### 不会因为activities处于stopped状态而崩溃

如果观察者的生命周期处于inactive状态，例如activity处于后台栈中，但是它无法接收到任何LiveData事件。

##### 无需手动处理生命周期

UI组件只观察对应数据，而不会停止/恢复观察。LiveData在观察过程中会自动地管理这些，因为它可以感知到对应的生命周期状态。

##### 始终保持最新数据

如果生命周期进入inactive状态，在重新变为active状态时，会立刻获取到最后发送的信息。

##### 正确处理配置更改

如果一个activity或fragment因为配置更改而recreate，例如设备旋转，它会立刻接收到最新的有效数据。

##### 共享资源

可以使用单例模式来扩展LiveData，用来包装系统服务，从而在App中可以共享使用。LiveData连接系统服务一次，之后任意需要该资源的观察者都可以通过观察LiveData来获取资源。

---

##### 使用LiveData组件

1.创建LiveData实例来保存某一种类数据，这通常在ViewModel类中进行。

2.创建一个定义了onChanged()方法的观察者对象，用来控制当LiveData数据发生改变时如何进行处理。通常在UI controller中创建这个对象，例如在activity或fragment中。

3.通过observe()方法将观察者和LiveData对象连接起来。这个observe()方法需要一个LifecycleOwner对象，这将观察者订阅到了LiveData对象上，以保证可以接收到更改通知。通常在UI controller中连接对象，例如在activity或fragment中。

Note：使用observeForever(observer)方法注册观察者可以不需要和LifecycleOwner
对象关联。这种情况下，观察者被默认为永远处于active状态而且永远可以接收修改。可以通过removeObserver(observer)方法来移除这些观察者。

当你修改LiveData中保存的数据时，所有处于active状态的观察者都会被触发。

LiveData允许UI controller观察者订阅更新。当LiveData中的数据改变时，UI会立即自动更新。

##### 创建LiveData

LiveData可以用来包装任意数据，包括集合对象。LiveData对象通常存储在ViewModel中，通过一个getXxx方法来访问。


```
public class NameViewModel extends ViewModel {

    // Create a LiveData with a String
    private MutableLiveData<String> mCurrentName;

    public MutableLiveData<String> getCurrentName() {
        if (mCurrentName == null) {
            mCurrentName = new MutableLiveData<String>();
        }
        return mCurrentName;
    }

    // Rest of the ViewModel...
}
```

##### 观察LiveData

大多数情况下，应该在App组件的onCreate()方法中观察LiveData，理由如下：

1.确保不会发出多余的请求，例如在activity或fragment的onResume()方法中。

2.确保了当activity或fragment变为active状态时有可以展示的数据。当App组件进入STARTED状态时，它会从观察的LiveData对象接收到最新的数据(当被观察的LiveData对象被设置了新数据的情况下)。

通常情况下，LiveData只在数据发生改变时进行传递，且只传递给处于active状态下的观察者。一个例外的情况是：当观察者从inactive状态切换为active状态时，也会收到一次更新，但如果非初次从inactive切换到active状态，且数据未发生过变化，则不会接收到更新通知。

```
public class NameActivity extends AppCompatActivity {

    private NameViewModel mModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Other code to setup the activity...

        // Get the ViewModel.
        mModel = ViewModelProviders.of(this).get(NameViewModel.class);


        // Create the observer which updates the UI.
        final Observer<String> nameObserver = new Observer<String>() {
            @Override
            public void onChanged(@Nullable final String newName) {
                // Update the UI, in this case, a TextView.
                mNameTextView.setText(newName);
            }
        };

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        // observe()调用后，将nameObserver作为一个参数传了进去，会将mCurrentName中最近存储的数据通过调用onChanged立即传过去。如果LiveData未设置数值，则不会调用。
        mModel.getCurrentName().observe(this, nameObserver);
    }
}
```

##### 更新LiveData

LiveDate未提供public方法进行数据更新操作，需要通过MutableLiveData类提供的setValue(T)和postValue(T)，且必须通过这俩个方法对LiveData中保存的数据进行修改。

MutableLiveData代码如下：


```
package android.arch.lifecycle;

/**
 * {@link LiveData} which publicly exposes {@link #setValue(T)} and {@link #postValue(T)} method.
 *
 * @param <T> The type of data hold by this instance
 */
@SuppressWarnings("WeakerAccess")
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```


通常MutableLiveData只在ViewModel中使用，ViewModel只将其暴露给观察者。

示例：


```
mButton.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        String anotherName = "John Doe";
        mModel.getCurrentName().setValue(anotherName);
    }
});
```

Note：在UI线程中使用setValue()，工作线程中使用postValue()。

##### 扩展LiveData

继承LiveData，实现单例模式，重写onActive和onInactive函数。

```
public class StockLiveData extends LiveData<BigDecimal> {
    private static StockLiveData sInstance;
    private StockManager mStockManager;

    private SimplePriceListener mListener = new SimplePriceListener() {
        @Override
        public void onPriceChanged(BigDecimal price) {
            setValue(price);
        }
    };

    @MainThread
    public static StockLiveData get(String symbol) {
        if (sInstance == null) {
            sInstance = new StockLiveData(symbol);
        }
        return sInstance;
    }

    private StockLiveData(String symbol) {
        mStockManager = new StockManager(symbol);
    }

    @Override
    protected void onActive() {
        mStockManager.requestPriceUpdates(mListener);
    }

    @Override
    protected void onInactive() {
        mStockManager.removeUpdates(mListener);
    }
}
```

单例模式：多个空间可以使用同一个LiveData，实现数据共享;

onActive():当前LiveData拥有处于active状态的观察者时调用，开始对数据进行观察;

onInactive():当前LiveData没有处于active状态的观察者时调用，用于无监听时的一些处理。

##### LiveData中数据转换

通过Transformations.map()和.switchMap()进行转换

```
    LiveData<User> userLiveData = ...;
    LiveData<String> userName = Transformations.map(userLiveData,        user -> {
        user.name + " " + user.lastName
    });

    // map方法的源码：
    @MainThread
    public static <X, Y> LiveData<Y> map(@NonNull LiveData<X> source,
            @NonNull final Function<X, Y> func) {
        final MediatorLiveData<Y> result = new MediatorLiveData<>();
        result.addSource(source, new Observer<X>() {
            @Override
            public void onChanged(@Nullable X x) {
                result.setValue(func.apply(x));
            }
        });
        return result;
    }
    
    // swichMap方法的源码：
    @MainThread
    public static <X, Y> LiveData<Y> switchMap(@NonNull LiveData<X> trigger,
            @NonNull final Function<X, LiveData<Y>> func) {
        final MediatorLiveData<Y> result = new MediatorLiveData<>();
        result.addSource(trigger, new Observer<X>() {
            LiveData<Y> mSource;

            @Override
            public void onChanged(@Nullable X x) {
                LiveData<Y> newLiveData = func.apply(x);
                if (mSource == newLiveData) {
                    return;
                }
                if (mSource != null) {
                    result.removeSource(mSource);
                }
                mSource = newLiveData;
                if (mSource != null) {
                    result.addSource(mSource, new Observer<Y>() {
                        @Override
                        public void onChanged(@Nullable Y y) {
                            result.setValue(y);
                        }
                    });
                }
            }
        });
        return result;
    }
    
    //两者的区别在于，switchMap在onChanged中触发了一个新的改动时，会将转换后的LiveData加入观察者模式，移除前一个;
    //而map则是将数据的改动放入同一个LiveData
```

上面代码中两个方法的实现都使用了MediatorLiveData类，该类的作用是将多个数据源通过addSource方法加入到SafeIterableMap<LiveData<?>, Source<?>>中，并统一管理onActive和onInactive时的订阅。该类也可用于将多个源合并在一起，由观察者去订阅，例如：我们有两个LiveData实例1和2,我们想将1和2合并为一个liveDataMerger，无论哪一个发生改变，都将通知观察者。这时，1和2就是merger中的source。

---

### 源码分析

##### 1.修改数据

在开发中使用的MutableLiveData，代码很短

```
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

这个类只是重写了LiveData的postValue和setValue方法（原方法为protected）。


```
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }
```

setValue是需要在主线程进行调用，而postValue用于工作线程中调用。

首先先看postValue方法，注释中写同时调用liveData.postValue("a")和liveData.setValue("b");的话，b会在a之前被传回。这也是官方建议在主线程要使用setValue的原因。

在postValue中，通过synchronized关键字来保证同步。代码中postTask的值是当前保存的mPendingData是否为NOT_SET，而mPendingData被设置为NOT_SET实在最后一行，执行的mPostValueRunnable中：


```
    private final Runnable mPostValueRunnable = new Runnable() {
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            //noinspection unchecked
            setValue((T) newValue);
        }
    };
```

所以当多个postValue同时调用时，如果前面提交的未来的及处理，那么后面的提交会将前面覆盖，并在（！postTask）处返回，而runnable中最后调用的仍是setValue方法。

下面我们来看setValue方法：

```
    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        // 分发数据
        dispatchingValue(null);
    }
    
    private void dispatchingValue(@Nullable LifecycleBoundObserver initiator) {
        // mDispatchingValue是用来防止在更新观察者消息时，接到新的更新消息，造成同步问题：
        // 开始时判断是否为true，为true则返回;执行过程中为true，执行完为false。
        // 而在遍历时，通过mDispatchInvalidated来判断过程中是否有新的更新rugosa有，则停止当前的分发
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                // 当前状态变化时调用，数据如果未修改，在considerNotify中不会update observer
                considerNotify(initiator);
                initiator = null;
            } else {
                // setValue中传入的为null，即遍历订阅的观察者，调用对应onChanged方法
                for (Iterator<Map.Entry<Observer<T>, LifecycleBoundObserver>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }
    
    private void considerNotify(LifecycleBoundObserver observer) {
        if (!observer.active) {
            return;
        }
        // 当前处于inactive状态时，计算是否回调onInactive
        if (!isActiveState(observer.owner.getLifecycle().getCurrentState())) {
            observer.activeStateChanged(false);
            return;
        }
        // 数据未变化，返回
        if (observer.lastVersion >= mVersion) {
            return;
        }
        observer.lastVersion = mVersion;
        // 调用onChanged，通知数据改变
        observer.observer.onChanged((T) mData);
    }
```

##### 2.添加观察者

上文中提到过，LiveData通过observe函数进行观察者的订阅：


```
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        // 判断状态
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        
        // 将当前LifecycleOwner（activity or fragment）与observer封装成一个LifecycleBoundObserver对象，并判断当前观察者是否已经注册过
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        LifecycleBoundObserver existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && existing.owner != wrapper.owner) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        
        // 注册Lifecycle，状态改变时会回调onStateChanged函数，对LiveData的状态进行同步修改
        owner.getLifecycle().addObserver(wrapper);
    }
```

在订阅中绑定了Lifecycle，实现了LiveData的生命周期订阅，用于更新状态。而之前说过的observeForever方法也是调用了observe方法，第一个参数为ALWAYS_ON，以此达到默认为active状态的效果：


```
    private static final LifecycleOwner ALWAYS_ON = new LifecycleOwner() {

        private LifecycleRegistry mRegistry = init();

        private LifecycleRegistry init() {
            LifecycleRegistry registry = new LifecycleRegistry(this);
            registry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
            registry.handleLifecycleEvent(Lifecycle.Event.ON_START);
            registry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
            return registry;
        }

        @Override
        public Lifecycle getLifecycle() {
            return mRegistry;
        }
    };
```




[1]:https://developer.android.google.cn/topic/libraries/architecture/livedata