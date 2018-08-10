## Service的启动方式

Service(服务)是一种Android提供的可在后台运行，不具有用户交互界面的应用组件。当用户需要进行长期的或是耗时的操作，不需要用户感知时，可以使用开启服务的方式来让程序在后台运行。Service创建之后首先要在manifest中注册:

```
	<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        
		//注册Service
        <service android:name=".servicetest.NormalService" />
    </application>
```

在manifest中注册时，可以为Service附上不同的权限：exported属性可以控制是否能够隐式开启服务；process可以自定义服务启动时运行的进程(默认为当前进程)，等等。

Service有两种启动方式：start和bind。根据不同的启动方式，Service会调用不同的回调函数，具有不同的特性。

---

#### start方式启动

```
Intent intent = new Intent(this, NormalService.class);
startService(intent);
```

和显示启动Activity的方式类似，可以用这种方式显示的启动一个服务。start方式启动后，即使Activity被销毁，Service依然可以存活。Service第一次被启动时会调用onCreate，start方式还会调用onStartCommand函数：

```
    /**
     * @param intent The Intent supplied to {@link android.content.Context#startService}, 
     * as given.  This may be null if the service is being restarted after
     * its process has gone away, and it had previously returned anything
     * except {@link #START_STICKY_COMPATIBILITY}.
     * @param flags Additional data about this start request.
     * @param startId A unique integer representing this specific request to 
     * start.  Use with {@link #stopSelfResult(int)}.
     * 
     * @return The return value indicates what semantics the system should
     * use for the service's current started state.  It may be one of the
     * constants associated with the {@link #START_CONTINUATION_MASK} bits.
     * 
     * @see #stopSelfResult(int)
     */
    public @StartResult int onStartCommand(Intent intent, @StartArgFlags int flags, int startId) {
        onStart(intent, startId);
        return mStartCompatibility ? START_STICKY_COMPATIBILITY : START_STICKY;
    }
```

我们来看一下这几个参数：
**Intent intent：** 启动时的intent对象，可以将数据从Activity经由intent对象传输过来
**int flags：** 启动时是否有额外数据。可选值有0， START_FLAG_REDELIVERY， START_FLAG_RETRY三种。其中0代表无额外数据；START_FLAG_REDELIVERY表示前一次启动的服务在调用stopSelf之前就已经被kill，在这种情况下会重新建立服务，并携带intent的值；START_FLAG_RETRY表示前一次启动服务时没有到达onStartCommand函数或是调用的onStartCommand并没有返回任何值，尝试重新调用。

```
    /**
     * This flag is set in {@link #onStartCommand} if the Intent is a
     * re-delivery of a previously delivered intent, because the service
     * had previously returned {@link #START_REDELIVER_INTENT} but had been
     * killed before calling {@link #stopSelf(int)} for that Intent.
     */
    public static final int START_FLAG_REDELIVERY = 0x0001;
    
    /**
     * This flag is set in {@link #onStartCommand} if the Intent is a
     * retry because the original attempt never got to or returned from
     * {@link #onStartCommand(Intent, int, int)}.
     */
    public static final int START_FLAG_RETRY = 0x0002;
```

**int startId：**  本次启动的标示ID，在stopSelfResult(int)被使用，准确的定位到将要关闭的服务

实际上onStartCommand的返回值int类型才是最最值得注意的，它有三种可选值， START_STICKY，START_NOT_STICKY，START_REDELIVER_INTENT，它们具体含义如下：

**START_STICKY：** 如果Service因为内存不足而被kill时，当内存有空闲时，会重新尝试创建Service。此时创建Service时调用的onStartCommand方法中的intent参数为null（或是有挂起的pendingIntent）。此种返回值适用于不需要传值，只需要一直运行在后台等待任务的服务。
**START_NOT_STICKY：** 如果Service因为内存不足而被kill时，当内存有空余时，也不会重新尝试创建Service，只能通过代码调用自行重启服务。
**START_REDELIVER_INTENT：** 当Service因为内存不足而被kill时，会重新启动服务，并将最后一次传递的intent（携带数据）传输过去。此种返回值适用于需要立刻恢复作业进度的服务。

---

#### bind方式启动

```
Intent intent = new Intent(this, NormalService.class);
bindService(intent, conn, Service.BIND_AUTO_CREATE);
```

bind启动方式启动服务，类似于启动一个客户端-服务端中的服务端。组件绑定了服务之后，可以向服务发送请求；同时服务也可以将请求的数据返还给组件。这里涉及到组件和服务如果不在统一进程内，需要进行IPC操作（本文不含IPC讲解，想要看的大佬们可以点击参考文献【1】）。而组件和服务的交互，通过的就是IBinder对象。我们通过绑定的调用函数来一点点分析。
上文中bindService中的第二个参数conn为ServiceConnection接口，实现如下：

```
private ServiceConnection conn = new ServiceConnection() {

        /**
         * Called when a connection to the Service has been established, with
         * the {@link android.os.IBinder} of the communication channel to the
         * Service.
         *
         * 当一个服务连接建立的时候调用，Service#onBind返回的IBinder对象作为双方通信的通道
         *
         * @param name The concrete component name of the service that has
         * been connected.
         * ComponentName 记录组件（四大组件）的类信息，如包名，class名等
         * @param iBinder The IBinder of the Service's communication channel,
         * which you can now make calls on.
         * IBinder Service#onBind返回，这里返回的是自定义的扩展Binder类，
         *    客户端可使用该对象对Service进行操作或数据传输
         */
        @Override
        public void onServiceConnected(ComponentName name, IBinder iBinder) {
            NormalService.NormalBinder binder = (NormalService.NormalBinder) iBinder;
            service = binder.getService();
        }

        /**
         * Called when a connection to the Service has been lost.  This typically
         * happens when the process hosting the service has crashed or been killed.
         * This does <em>not</em> remove the ServiceConnection itself -- this
         * binding to the service will remain active, and you will receive a call
         * to {@link #onServiceConnected} when the Service is next running.
         *
         * 当服务的连接意外中断时才会被调用（正常退出时不会调用）
         * 一般发生在服务的宿主进程崩溃或被kill的情况下
         * 但这并不会将ServiceConnection移除，当服务再次运行时会再次调用onServiceConnected
         *
         * @param name The concrete component name of the service whose
         * connection has been lost.
         */
        @Override
        public void onServiceDisconnected(ComponentName name) {
            service = null;
        }
    };
```

在onServiceConnected中返回的IBinder对象，是Service#onBind中返回的，作为组件和服务交互的纽带。返回的IBinder是自定义的扩展Binder类，可以获取到Service对象。而flags则是指定绑定时是否自动创建Service。0代表不自动创建、BIND_AUTO_CREATE则代表自动创建。

对应Service中的实现如下：

```
public class NormalService extends Service {
    private static final String TAG = "NormalService";
    private NormalBinder binder = new NormalBinder();

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "NormalService -> onBind");
        //返回binder
        return binder;
    }

    @Override
    public boolean onUnbind(Intent intent) {
        Log.d(TAG, "NormalService -> onUnbind");
        //解除绑定时调用
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "NormalService -> onDestroy");
        super.onDestroy();
    }

    //扩展Binder，提供获取Service对象的方法
    class NormalBinder extends Binder {
        NormalService getService() {
            return NormalService.this;
        }
    }
}
```

解除绑定：unbindService(conn)；

---

#### 混合启动

一个Service可以通过start启动，也可以通过bind启动，也可以同时使用两种启动方式。当一个Service第一次启动时，会先调用onCreate；如果先start再bind，函数调用顺序即为onCreate -> onStartCommand -> onBind；反之则为onCreate -> onBind -> onStartCommand。

当使用两种方式启动的Service需要销毁时，需要调用unbindService(conn)和stopService(intent)后，才会调用Service#onDestroy。其中unbindService会先调用onUnbind。

---

## Service的运行环境

当启动一个Service时，服务会默认运行在当前宿主进程的主线程，不会创建自己的线程。所以在服务中如果需要进行耗时操作需要另开线程，否则会阻塞主线程。即，Service的生命周期，onBind等方法都运行在主线程。

---

#### 参考资料
1.[zejian_ - 关于Android Service真正的完全详解，你需要知道的一切][1]
2.Android开发艺术探索-Service解析

[1]:https://blog.csdn.net/javazejian/article/details/52709857