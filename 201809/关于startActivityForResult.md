## 话题：关于startActivityForResult

1.startActivityForResult的使用场景是什么？
onActivityResult回调里面的resultCode和requestCode的含义是什么？

2.Activity A启动B的时候，在B中何时该执行setResult？
setResult可以位于Activity的finish方法之后吗？

---

### startActivityForResult的使用场景

##### startActivityForResult

活动A启动活动B，并需要将活动B得到的结果在finish之后传回到A时，可以在A中使用startActivityForResult来启动活动B，在B结束时，活动A通过onActivityResult来接受B返回的值。

```
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
}
```

我们常用的方法是带有两个参数的方法：Intent和int。方法中调用了startActivityForResult(intent, requestCode, bundle)。
（startActivity中调用了startActivityForResult(intent， -1) ）。

```
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //启动activity
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
				//当requestCode >= 0时，才会有返回值
                mStartedActivity = true;
            }
            cancelInputsAndStartExitTransition(options);
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

* Intent intent：用于启动一个Activity，内含bundle对象，可携带参数至下一个Activity。

* int requestCode：启动Activity时设置的request值，用于被启动Activity返回值时，标记返回结果所属Activity，需设置为>=0。

* Bundle options：有关如何启动Activity的其他附加选项。

请注意，此方法只应与定义为返回结果的Intent协议一起使用。 在其他协议（例如{@link Intent＃ACTION_MAIN}或{@link Intent＃ACTION_VIEW}）中，您可能无法获得预期的结果。 例如，如果您要启动的活动使用{@link Intent＃FLAG_ACTIVITY_NEW_TASK}，则它将不会在您的任务中运行，因此您将立即收到取消结果。

这里有一种特殊情况：如果在活动的初始onCreate（Bundle savedInstanceState）/ onResume（）期间使用requestCode> = 0调用startActivityForResult（），则在从已启动的活动返回结果之前，将不会显示您的窗口。 这是为了避免在重定向到另一个活动时出现明显的闪烁。

##### onActivityResult

```
protected void onActivityResult(int requestCode, int resultCode, Intent data) {}
```

* int requestCode：启动Activity时设置的requestCode值，用于标示数据返回于哪个Activity。

* int resultCode：当被启动的Activity退出时可设置requestCode并返回。resultCode通过setResult()函数进行设置。如果活动显式返回，没有返回任何结果，或者在操作期间崩溃，则resultCode将为{@link #RESULT_CANCELED}。

* Intent data：可以将结果数据返回给调用者（各种数据可以附加到Intent中）。

##### setResult

```
public final void setResult(int resultCode) {
        synchronized (this) {
            mResultCode = resultCode;
            mResultData = null;
        }
}
```

Android提供了三个默认resultCode：

```
	/** Standard activity result: operation canceled. */
    public static final int RESULT_CANCELED    = 0;
    /** Standard activity result: operation succeeded. */
    public static final int RESULT_OK           = -1;
    /** Start of user-defined activity results. */
    public static final int RESULT_FIRST_USER   = 1;
```

前两个很好理解，但是第三个看不出字面的意思，最后在Stack Overflow找到了答案：

>"Why in the world FIRST_USER?" -- it's a fairly common convention in managing integer return codes, in cases where multiple parties (e.g., a library author and an app author) might want to define those codes. I remember seeing this in use in C/C++ 20-25 years ago. Basically, any values lower than RESULT_FIRST_USER are reserved for the framework, and any values starting with RESULT_FIRST_USER are up for apps to use, without risk of collision. You don't see it much in Android, at least at the Java level.

意思是说：这是定义integer返回值的一种很常见的值，防止多方在定义这些值时造成混乱；大体上就是，比 RESULT_FIRST_USER 小的值都是保存于framework层的，而用户自定义的值都是高于这个值的。这种做法常见于C/C++，而不常见于Java。

---

### 执行setResult

##### 调用finish的情况

首先我们来看finish的源码：

```
private void finish(int finishTask) {
        if (mParent == null) {
            int resultCode;
            Intent resultData;
            synchronized (this) {
                resultCode = mResultCode;
                resultData = mResultData;
            }
            if (false) Log.v(TAG, "Finishing self: token=" + mToken);
            try {
                if (resultData != null) {
                    resultData.prepareToLeaveProcess(this);
                }
                //返回resultCode
                if (ActivityManager.getService()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
            } catch (RemoteException e) {
                // Empty
            }
        } else {
            mParent.finishFromChild(this);
        }
    }
```

当Activity调用finish时，会将resultCode直接传回，setResult应该在finish之前调用。

##### 点击返回键onBackPressed

假设活动A通过startActivityForResult启动了活动B，再由活动B返回活动A时，A、B的生命周期调用为B->onPause，**A->onActivityResult**，A->onStart，A->onResume，B->onStop，B->onDestroy。即当按下返回键时，setResult要在onPause或之前调用，才能将返回值传回到A中。

---

### 参考资料

[1.沙翁-setResult()的调用时机](https://www.cnblogs.com/shaweng/p/3875825.html)

[2.Will someone please explain RESULT_FIRST_USER](https://stackoverflow.com/questions/32013948/will-someone-please-explain-result-first-user)
