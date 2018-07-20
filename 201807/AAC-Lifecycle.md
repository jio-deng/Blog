
# Lifecycle简介
我们先来看一下[官方网站][1]给出的定义：

>Lifecycle-aware components perform actions in response to a change in the lifecycle status of another component, such as activities and fragments. These components help you produce better-organized, and often lighter-weight code, that is easier to maintain.
生命周期感知组件执行操作以响应另一个组件（例如活动和片段）的生命周期状态的更改。这些组件可帮助您生成更易于组织且通常更轻量级的代码，这些代码更易于维护。

>A common pattern is to implement the actions of the dependent components in the lifecycle methods of activities and fragments. However, this pattern leads to a poor organization of the code and to the proliferation of errors. By using lifecycle-aware components, you can move the code of dependent components out of the lifecycle methods and into the components themselves.
一种常见的模式是在活动和片段的生命周期方法中实现依赖组件的操作。但是，这种模式导致代码组织不良以及错误的增加。通过使用生命周期感知组件，您可以将依赖组件的代码移出生命周期方法并移入组件本身。

通过官方文档的解释，Lifecycle可以将Activity/Fragment的生命周期调用解耦出来。在其他组件中，可以针对生命周期进行初始化/销毁的操作。


# Lifecycle的使用
Lifecycle的使用很简单：
1. **Activity/Fragment实现LifecycleOwner，创建lifecycleRegistry**
   **(AppCompatActivity默认实现了该接口，具体实现类为SupportActivity)** 
2. **实现LifecycleObserver**
3. **lifecycleObserver.addObserver(myObserver)**

示例Demo：

```
public class MainActivity extends Activity implements LifecycleOwner {
    private static final String TAG = "MainActivity";

    LifecycleRegistry lifecycleRegistry = new LifecycleRegistry(this);
    MyObserver myObserver;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        myObserver = new MyObserver("1");
        getLifecycle().addObserver(myObserver);
    }
    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return lifecycleRegistry;
    }


    public class MyObserver implements LifecycleObserver {
        public String name;

        MyObserver(String name) {
            this.name = name;
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
        public void onCreate() {
            Log.d(TAG, "onCreate: " + name);
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_START)
        public void onStart() {
            Log.d(TAG, "onStart: " + name);
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        public void onResume() {
            Log.d(TAG, "onResume: " + name);
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        public void onPause() {
            Log.d(TAG, "onPause: " + name);
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
        public void onStop() {
            Log.d(TAG, "onStop: " + name);
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
        public void onDestroy() {
            Log.d(TAG, "onDestroy: " + name);
        }
    }
}
```

# Lifecycle源码分析

声明：源码分析主要参照了[水晶虾饺][2]大佬的文章，按照自己的理解重新组织了一下语言，侵删；如有不对的地方，欢迎大家拍砖～～

Lifecycle 的实现跟 ViewModel 类似，都是利用 Fragment 来实现它的功能。通过添加一个 fragment 到 activity 中，这个 fragment 便能够接收到各个生命周期回调。看了上面的例子，Lifecycle的重点主要在三个地方：
1. **生命周期的控制**
2. **LifecycleRegistry对事件的分发**
3. **lifecycleRegistry.addObserver(myObserver)**

下面我们来一起捋一捋～

## 生命周期的严格控制

Lifecycle为了控制生命周期，定义了两个枚举类，事件和状态

```
public abstract class Lifecycle {
   ...
    @SuppressWarnings("WeakerAccess")
    public enum Event {
        /**
         * Constant for onCreate event of the {@link LifecycleOwner}.
         */
        ON_CREATE,
        /**
         * Constant for onStart event of the {@link LifecycleOwner}.
         */
        ON_START,
        /**
         * Constant for onResume event of the {@link LifecycleOwner}.
         */
        ON_RESUME,
        /**
         * Constant for onPause event of the {@link LifecycleOwner}.
         */
        ON_PAUSE,
        /**
         * Constant for onStop event of the {@link LifecycleOwner}.
         */
        ON_STOP,
        /**
         * Constant for onDestroy event of the {@link LifecycleOwner}.
         */
        ON_DESTROY,
        /**
         * An {@link Event Event} constant that can be used to match all events.
         */
        ON_ANY
    }

    /**
     * Lifecycle states. You can consider the states as the nodes in a graph and
     * {@link Event}s as the edges between these nodes.
     */
    @SuppressWarnings("WeakerAccess")
    public enum State {
        /**
         * Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not dispatch
         * any more events. For instance, for an {@link android.app.Activity}, this state is reached
         * <b>right before</b> Activity's {@link android.app.Activity#onDestroy() onDestroy} call.
         */
        DESTROYED,

        /**
         * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
         * the state when it is constructed but has not received
         * {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} yet.
         */
        INITIALIZED,

        /**
         * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} call;
         *     <li><b>right before</b> {@link android.app.Activity#onStop() onStop} call.
         * </ul>
         */
        CREATED,

        /**
         * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onStart() onStart} call;
         *     <li><b>right before</b> {@link android.app.Activity#onPause() onPause} call.
         * </ul>
         */
        STARTED,

        /**
         * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached after {@link android.app.Activity#onResume() onResume} is called.
         */
        RESUMED;

        /**
         * Compares if this State is greater or equal to the given {@code state}.
         *
         * @param state State to compare with
         * @return true if this State is greater or equal to the given {@code state}
         */
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```

事件类对应的是生命周期的变化，状态类对应的是保存的observer所处的状态。状态和事件之间的转换关系如下：

```
public class LifecycleRegistry extends Lifecycle {
    ...
    
    static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }

    private static Event downEvent(State state) {
        switch (state) {
            case INITIALIZED:
                throw new IllegalArgumentException();
            case CREATED:
                return ON_DESTROY;
            case STARTED:
                return ON_STOP;
            case RESUMED:
                return ON_PAUSE;
            case DESTROYED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }

    private static Event upEvent(State state) {
        switch (state) {
            case INITIALIZED:
            case DESTROYED:
                return ON_CREATE;
            case CREATED:
                return ON_START;
            case STARTED:
                return ON_RESUME;
            case RESUMED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }
}
```

状态转化与事件的关系图如下：
![状态-事件 转化关系](https://img-blog.csdn.net/20180720140033417?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzZDUwMTgyMzIwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
（图片来源：[流水不腐小夏][3]）

首先，状态之间的比较是根据Enum类的ordinal()值，即变量的声明顺序。值按照声明顺序递增。上图所示，左侧的INITIALIZED状态值为1，DESTROYED为0（因为这两个状态属于起始和结束时的状态，可以看作平级）；右侧RESUMED状态值为4。

由左向右状态改变为状态提升，即upEvent；由右向左状态改变为状态下降，即downEvent。通过事件传递来改变状态，从而调用生命周期回调，完成了对生命周期的控制。我们先往下看～

## LifecycleRegistry对事件的分发

在SupportActivity类中，activity默认实现了LifecycleOwner，其中ReportFragment就是Lifecycle加入activity中的用于获取生命周期回调的fragment，在fragment中调用了lifecycleRegistry的handleLifecycleEvent函数。

```
public class LifecycleRegistry extends Lifecycle {
    ...
    
    /**
     * Sets the current state and notifies the observers.
     * <p>
     * Note that if the {@code currentState} is the same state as the last call to this method,
     * calling this method has no effect.
     *
     * @param event The event that was received
     */
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            //当sync()正在运行时，即正在同步状态的时候，mHandlingEvent为true
            //addObserver时，会对新保存的observer重新dispatchEvent，此时mAddingObserverCounter为正在进行的数量
            //在这两种情况下，程序会走到if条件语句中；此时只需将mNewEventOccurred置为true，可以加快sync中的函数返回
            //返回之后，会重新进行sync
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        //同步状态变化，并调用对应生命周期回调
        sync();
        mHandlingEvent = false;
    }
}
```

接下来看一下sync的实现：

```
public class LifecycleRegistry extends Lifecycle {
    ...
    
    /**
     * Custom list that keeps observers and can handle removals / additions during traversal.
     *
     * Invariant: at any moment of time for observer1 & observer2:
     * if addition_order(observer1) < addition_order(observer2), then
     * state(observer1) >= state(observer2),
     * 
     * Invariant的规定：无论何时，先加入的observer的state值要大于后加入的observer的状态值
     * 这个规定影响了sync的实现
     */
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();

    // happens only on the top of stack (never in reentrance),
    // so it doesn't have to take in account parents
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            //moveToState中将该变量置为true是为了让下面的函数快速返回；
            //这里在调用之前，置为false
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            //参照上文Invariant的规定，先加入的observer的state值一定要大于或等于后加入的observer的state值
            //那么在mObserverMap中，eldest的state最大，newest的state最小。如果当前mState小于eldest的state值，说明mObserverMap中的值需要更新
            //backwardPass执行时，为了不改变Invariant的规定，从队尾开始执行
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            //在backwardPass执行时，可能在回调中触发了事件状态，所以需要检查mNewEventOccurred
            //同理，Invariant规定，正序调用forwardPass
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
    
    private void forwardPass(LifecycleOwner lifecycleOwner) {
        //升序迭代器
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        //降序迭代器
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    //在回调中可能移除了observer
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                //pushParentState和popParentState的作用在下面详细解释以下
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }
}
```

sync()的主要作用是将保存的observer的状态都同步到mState，通过调用forwardPass和backwardPass，把事件传递给用户，将状态的改变转化为生命周期的调用。

## 注册观察者

lifecycleRegistry.addObserver(myObserver)，我们来看看addObserver方法

```
public class LifecycleRegistry extends Lifecycle {
    ...
    
    // we have to keep it for cases:
    // void onStart() {
    //     mRegistry.removeObserver(this);
    //     mRegistry.add(newObserver);
    // }
    // newObserver should be brought only to CREATED state during the execution of
    // this onStart method. our invariant with mObserverMap doesn't help, because parent observer
    // is no longer in the map.
    //
    // 上文在dispatchEvent前后进行的pushParentState和popParentState就是为了解决这个问题：
    // 如果在onStart中注销了observer又重新注册，这时重新注册的observer初始的状态为INITIALIZED；
    // 而如果想执行ON_START的对应回调，需要newObserver处于CREATED状态（之前的状态因为removeObserver，不存在于mObserverMap中）
    // mParentStates的作用就是为newObserver提供了CREATED这个状态
    private ArrayList<State> mParentStates = new ArrayList<>();

    private State calculateTargetState(LifecycleObserver observer) {
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

        State siblingState = previous != null ? previous.getValue().mState : null;
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        //返回最小状态
        return min(min(mState, siblingState), parentState);
    }

    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        //下文讨论ObserverWithState的构造函数
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        // 如果在回调中调用了addObserver，isReentrance为true，不进行sync
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            // dispatch事件给observer，在回调中可能会改变事件的状态，需要重新计算
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }

    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            // 生成适配器
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            // 通知observer，回调
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
}
```

在addObserver开始的时候，创建了一个ObserverWithState的实例；在ObserverWithState的构造函数中调用了Lifecycling.getCallback(observer)

```
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public class Lifecycling {
    @NonNull
    static GenericLifecycleObserver getCallback(Object object) {
        ...

        final Class<?> klass = object.getClass();
        // 生成适配器类
        int type = getObserverConstructorType(klass);
        
        ...
    }
}
```
具体代码太多就不贴了，${activity_name}_${method_name}_LifecycleAdapter，以此格式生成适配器类的类名，该适配器用于将dispatchEvent分配的事件和LifecycleObserver的对应生命周期回调联系在一起。

命名规则如下：
```
final Class<? extends GeneratedAdapter> aClass = (Class<? extends GeneratedAdapter>) Class.forName(fullPackage.isEmpty() ? adapterName : fullPackage + "." + adapterName);
Constructor<? extends GeneratedAdapter> constructor = aClass.getDeclaredConstructor(klass);
```

贴出上文例子中产生的适配器类：

```
public class MainActivity_MyObserver_LifecycleAdapter implements GeneratedAdapter {
  final MainActivity.MyObserver mReceiver;

  MainActivity_MyObserver_LifecycleAdapter(MainActivity.MyObserver receiver) {
    this.mReceiver = receiver;
  }

  @Override
  public void callMethods(LifecycleOwner owner, Lifecycle.Event event, boolean onAny,
      MethodCallsLogger logger) {
    boolean hasLogger = logger != null;
    if (onAny) {
      return;
    }
    if (event == Lifecycle.Event.ON_CREATE) {
      if (!hasLogger || logger.approveCall("onCreate", 1)) {
        mReceiver.onCreate();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_START) {
      if (!hasLogger || logger.approveCall("onStart", 1)) {
        mReceiver.onStart();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_RESUME) {
      if (!hasLogger || logger.approveCall("onResume", 1)) {
        mReceiver.onResume();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_PAUSE) {
      if (!hasLogger || logger.approveCall("onPause", 1)) {
        mReceiver.onPause();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_STOP) {
      if (!hasLogger || logger.approveCall("onStop", 1)) {
        mReceiver.onStop();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_DESTROY) {
      if (!hasLogger || logger.approveCall("onDestroy", 1)) {
        mReceiver.onDestroy();
      }
      return;
    }
  }
}
```

源码分析到这里就结束了～～相信大家也对Lifecycle的工作流程有了各自的理解。如果本文有书写错误的地方，欢迎大家拍砖～～


---------
参考资料：
1.[Jekton Android arch components 源码分析（2）—— Lifecycle][4]
2.[浅谈Android Architecture Components][5]


[1]: https://developer.android.google.cn/topic/libraries/architecture/lifecycle
[2]: https://jekton.github.io/2018/07/06/android-arch-lifecycle/
[3]: https://blog.csdn.net/guijiaoba/article/details/73692397
[4]: https://jekton.github.io/2018/07/06/android-arch-lifecycle/
[5]: https://blog.csdn.net/guijiaoba/article/details/73692397