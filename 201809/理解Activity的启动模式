### Activity的四种启动模式

当我们多次启动同一个Activity时，默认状态下，系统会创建多个实例并一一放入到任务栈中。Android在设计时也考虑到，重复创建多个相同的实例很傻，于是提供了四种启动模式供开发者修改系统默认的启动方式。四种方式为：standard、singleTop、singleTask和singleInstance。

##### standard

标准模式（默认）：每次启动Activity都会重新创建一个新的实例，被创建的实例会调用onCreate、onStart、onResume，并加入到启动该实例的任务栈中。

此时，如果使用ApplicationContext去启动则会报错，因为appContext没有所属的任务栈，解决方法为启动时为activity指定FLAG_ACTIVITY_NEW_TASK标记，这样启动时会为它创建一个新的任务栈，这个时候待启动的activity实际上是以singleTask模式启动的。

##### singleTop

栈顶复用模式：如果activity已经启动过，并且在栈顶（top activity），则不会重新启动；onCreate和onStart不会被调用，而是调用onNewIntent传入intent参数。如果Activity没有被启动过，则启动方式与standard模式一致。

##### singleTask

栈内复用模式：当处于该模式的Activity启动时，系统首先会寻找Activity想要启动的任务栈（如果没有则创建），找到后寻找栈内是否已有Activity实例：如果没有，则创建；如果有，则调用onNewIntent，并移除其上的所有Activity使其位于栈顶。

例如：栈内原为ABCD四个Activity，B为singleTask模式。此时在D中再次启动B，则调用B的onNewIntent传入intent参数，并移除CD，栈内剩余AB。

##### singleInstance

单例模式：与singleTask类似，不同点在于，该模式下的Activity只能单独处于一个栈内。启动后，后续的启动都不会再创建新的实例，除非这个单独的任务栈被系统销毁。

当Activity1启动Activity2(singleInstance)时，如果按下HOME键，再点击app，会显示是Activity1的界面：因为点击图标显示的是主栈中的Activity，而singleInstance启动的Activity存放在单独的栈中。

---

### Intent Flags

Android还提供了一些标记位，可用于在启动时设定Activity的启动模式、影响启动状态等等。有一些标记位是系统需要用到，不建议开发者调用的。

#### 启动模式

##### FLAG_ACTIVITY_SINGLE_TOP

与singleTop模式启动一致：设置该属性的activity如果在历史栈顶，则不会启动该活动。

##### FLAG_ACTIVITY_NEW_TASK

与singleTask模式启动一致：使用此标志时，如果任务已在您正在启动的活动中运行，则不会启动新活动; 当前任务将简单地以最后一次启动的状态显示在屏幕的前面。

使用FLAG_ACTIVITY_NEW_TASK标记位，则无法通过startActivityForResult的方式返回result。

##### FLAG_ACTIVITY_MULTIPLE_TASK

此标志始终与{@link #FLAG_ACTIVITY_NEW_DOCUMENT}或{@link #FLAG_ACTIVITY_NEW_TASK}配对。
在这两种情况下，仅这些标志就会搜索现有任务以查找与此Intent匹配的任务。只有在没有找到这样的任务时才会创建新任务。
与FLAG_ACTIVITY_MULTIPLE_TASK配对时，会修改这两种行为以跳过搜索匹配任务并无条件地启动新任务。

与FLAG_ACTIVITY_NEW_TASK结合使用：首先在Intent中设置FLAG_ACTIVITY_NEW_TASK, 打开Activity,则启动一个新Task；
接着在Intent中设置FLAG_ACTIVITY_MULTIPLE_TASK, 再次打开同一个Activity,则还会新启动一个Task。

##### FLAG_ACTIVITY_NEW_DOCUMENT

如果指定了这个flag，则在一个新的task中启动activity。如果想更精确的控制启动的task，可以使用documentlaunchmode属性。

一个activity在mainfest文件中添加 属性：android:documentLaunchMode时，该activity被启动时永远会创建一个新的task。该属性有4个值，用户在应用中打开一个document时会有不同的效果：

1.intoExisting：activity 会为该document请求一个已经存在的task，这与设置FLAG_ACTIVITY_NEW_DOCUMENT 且不设置FLAG_ACTIVITY_MULTIPLE_TASK 有相同的效果。

2.always： activity 会为该document创建一个新的task，即使该document已经被打开了，这与设置 FLAG_ACTIVITY_NEW_DOCUMENT 且设置FLAG_ACTIVITY_MULTIPLE_TASK 有相同的效果。

3.none：activity 不会为 document 创建新的task，该app被设置为 single task 的模式，它会重新调用用户唤醒的所有activity中的最近的一个。

4.never：activity 不会为document创建一个新的task，设置这个值复写了 FLAG_ACTIVITY_NEW_DOCUMENT 和 FLAG_ACTIVITY_MULTIPLE_TASK 标签。如果其中一个标签被设置，并且overview screen 显示该app为 single task 模式。 则该activity 会重新调用用户最近唤醒的activity。

注意： none 或 nerver 使用时，activity必须设置为 launchMode=”standard” ，如果该属性没有设置，documentLaunchMode=”none” 属性就会被使用。

#### 启动状态

##### FLAG_ACTIVITY_NO_HISTORY

新活动不会保留在历史堆栈中。 一旦用户离开它，活动就finish了。
这也可以使用{@link android.R.styleable #AndroidManifestActivity_noHistory noHistory}属性进行设置。

设置该属性的activity不会返回result。

##### FLAG_ACTIVITY_CLEAR_TOP

如果已设置了该标记，并且正在启动的活动已在当前任务中运行，则不会启动该活动的新实例，而是将关闭其上的所有其他活动，并将此Intent传递给（现在位于top的）旧活动作为新的意图。

例如，考虑一个由活动组成的任务：A，B，C，D。如果D调用带有解析为活动B组件的Intent的startActivity（），则C和D将finish，B将接收给定的Intent，导致堆栈现在是：A，B。

如果它已将其启动模式声明为“multiple”（默认值），并且您**没有**在同一意图中设置{@link #FLAG_ACTIVITY_SINGLE_TOP}，那么它将被finish并restart; 对于所有其他启动模式或如果设置了{@link #FLAG_ACTIVITY_SINGLE_TOP}，则此Intent将被传递到当前实例的onNewIntent（）。

此启动模式也可以与{@link #FLAG_ACTIVITY_NEW_TASK}结合使用：如果用于启动task栈的根activity，它将把该任务的任何当前运行的实例带到前台，然后把该栈清空为根状态。这在从通知管理器启动活动时尤其有用，比如从notification manager启动activity。

##### FLAG_ACTIVITY_FORWARD_RESULT

使用设置了该标记的intent从Activity1启动Activity2，则Activity1的回复目标将转移到Activity2。这样，Activity2可以调用{@link android.app.Activity #setResult}并将该结果发送回Activity1的回复目标。

即：在新activity中通过seResult，将结果返回到最开始的activity中

##### FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS

如果设置了该标记并且intent用于从现有活动启动新活动，则当前活动将不会被计为top activity（用于决定是否应将新意图传递到顶部而不是启动新活动的最高活动）。
之前的活动将用作顶部，假设当前活动将立即完成。

即：新活动不会保留在最近启动的活动列表中。

##### FLAG_ACTIVITY_RESET_TASK_IF_NEEDED

一般与FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET结合使用，如果设置该属性，这个activity将在一个新的task中启动或者或者被带到一个已经存在的task的顶部，这时这个activity将会作为这个task的首个页面加载。

将会导致与这个应用具有相同亲和力的task处于一个合适的状态(移动activity到这个task或者从中移出)，或者简单的重置这个task到它的初始状态

##### FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET

在当前的Task堆栈中设置一个还原点，当带有FLAG_ACTIVITY_RESET_TASK_IF_NEEDED的Intent请求启动这个堆栈时(典型的例子是用户从桌面再次启动这个应用)，还原点之上包括这个应用将会被清除。

应用场景：在email程序中预览图片时，会启动图片观览的activity，当用户离开email处理其他事情，然后下次再次从home进入email时，我们呈现给用户的应该是上次email的会话，而不是图片观览，这样才不会给用户造成困惑。

##### FLAG_ACTIVITY_REORDER_TO_FRONT

如果在传递给{@link Context＃startActivity Context.startActivity（）}的Intent中设置，则此标志将导致已启动的活动被带到其任务的历史堆栈的前面（如果它已在运行）。

例如：栈中有A、B、C、D四个Activity，现在由D启动B，B设置了FLAG_ACTIVITY_REORDER_TO_FRONT，则栈中结果为A、C、D、B。

##### FLAG_ACTIVITY_NO_USER_ACTION

禁止activity调用onUserLeaveHint()函数。onUserLeaveHint()作为activity周期的一部分，它在activity因为用户要跳转到别的activity而退到background时使用。

比如，在用户按下Home键（用户的操作），它将被调用。比如有电话进来（不属于用户的操作），它就不会被调用。
注意：通过调用finish()时该activity销毁时不会调用该函数。

如果活动是通过任何非用户驱动的事件（如电话呼叫接收或警报处理程序）启动的，则应将此标志传递给{@link Context＃startActivity Context.startActivity}，以确保暂停的活动不会认为用户已确认其通知。

##### FLAG_ACTIVITY_NO_ANIMATION

此标志将阻止系统应用活动转换动画以转到下一个活动状态（禁止activity之间的切换动画）。 

这并不意味着动画永远不会运行 - 如果在显示此处的活动之前未发生指定此标志的其他活动更改，则将使用该转换。当您要进行一系列活动操作时，可以很好地使用此标志，但用户看到的动画不应由第一个活动更改驱动，而应由后一个活动更改驱动。

##### FLAG_ACTIVITY_CLEAR_TASK

此标志将导致在活动开始之前清除与活动关联的任何现有任务。 

也就是说，活动成为空任务的新root，并且任何旧活动都已finish。 这只能与{@link #FLAG_ACTIVITY_NEW_TASK}一起使用。

##### FLAG_ACTIVITY_TASK_ON_HOME

此标志将使新启动的任务置于当前主活动任务（如果有）之上。 

也就是说，从任务中退回将始终将用户返回到HOME，即使这不是他们看到的最后一个活动。这只能与{@link #FLAG_ACTIVITY_NEW_TASK}一起使用。

##### FLAG_ACTIVITY_RETAIN_IN_RECENTS

默认情况下，{@link #FLAG_ACTIVITY_NEW_DOCUMENT}创建的文档将在用户关闭它时删除其最近任务中的条目（使用后退或其他方式可以finish）。
如果您希望将文档保留在最近，以便可以重新启动，则可以使用此标志。设置完成并且任务的活动完成后，最新条目将保留在界面中供用户重新启动它，就像顶级应用程序的最近条目一样。

##### FLAG_ACTIVITY_LAUNCH_ADJACENT

此标志仅用于分屏多窗口模式。 新活动将显示在启动它的旁边。这个标记只能与{@link #FLAG_ACTIVITY_NEW_TASK}一起使用。 

此外，如果要创建现有活动的新实例，则需要设置{@link#FLAG_ACTIVITY_MULTIPLE_TASK}。

#### 权限类

##### FLAG_GRANT_READ_URI_PERMISSION
##### FLAG_GRANT_WRITE_URI_PERMISSION

添加该标记的Intent的接收者将被授予对Intent数据中的URI和其ClipData中指定的任何URI执行读／写取操作的权限。当应用于Intent的ClipData时，将授予所有URI以及通过Intent项中的数据或其他ClipData的递归遍历;仅使用顶级Intent的授权标志。

##### FLAG_GRANT_PERSISTABLE_URI_PERMISSION

与{@link #FLAG_GRANT_READ_URI_PERMISSION}和/或{@link#FLAG_GRANT_WRITE_URI_PERMISSION}结合使用时，URI权限授权可以在设备重新启动后保留，直到使用{@link Context＃revokeUriPermission（Uri，int）}显式撤消。 
该标志仅提供持久化的可能; 接收应用程序必须调用{@link ContentResolver #takePersistableUriPermission（Uri，int）}才能实际持久化。

##### FLAG_GRANT_PREFIX_URI_PERMISSION

URI匹配方案：与{@link #FLAG_GRANT_READ_URI_PERMISSION}和/或{@link #FLAG_GRANT_WRITE_URI_PERMISSION}结合使用时，URI权限授予适用于与原始授权URI匹配的前缀的任何URI。
如果没有此标志，则URI必须与要授予的访问权限完全匹配。仅当方案，权限和前缀定义的所有路径段完全匹配时，另一个URI才被视为前缀匹配。

---

### 参考资料

1.Android开发艺术探索

[2.Shawn_Dut-android深入解析Activity的launchMode启动模式，Intent Flag，taskAffinity](https://blog.csdn.net/self_study/article/details/48055011)

[3.WangMark-Activity的启动模式 和 Intent启动选项](https://blog.csdn.net/petib_wangwei/article/details/53670141)
