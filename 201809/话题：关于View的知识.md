### 话题：关于View的知识

1.View的getWidth()和getMeasuredWidth()有什么区别？

2.如何在onCreate()中拿到View的宽度和高度？

---

##### MeasuredSpec

MeasureSpec封装了从父级传递给子级的布局要求，一个MeasureSpec代表着宽或高的要求。MeasureSpec是一个int值，这么设计的原因是尽量在View绘制过程中减少对象的分配，而将所需的<size，mode> 元组封装到MeasuredSpec中，并提供pack和unpack的方法。

mode有三种：

```
        //未指定，父元素不对子元素施加任何束缚，子元素可以得到任意想要的大小
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        //父元素决定子元素的确切大小，子元素将被限定在给定的边界里而忽略它本身大小
        public static final int EXACTLY     = 1 << MODE_SHIFT;
        //子元素至多达到指定大小的值
        public static final int AT_MOST     = 2 << MODE_SHIFT;
```

MeasureSpec将size和mode封装在一起。size占据低30位，mode占据高2位。MASK用于将高2位的mode提取出来，～MASK用于将低30位的size提取出来：

```
		//mask对size的最大值做了限制
		private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

		@MeasureSpecMode@MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }
        
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
```

MeasuredSpec中生成mMeasuredSpec的方法是makeMeasureSpec。makeMeasureSpec方法在API小于等于17的设备上，对参数没有做限制，任何一个值的溢出都可能影响生成的MeasureSpec。{@link android.widget.RelativeLayout}受此错误的影响。 针对API级别大于17的应用将获得固定的、更严格的行为。

```
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode)
```

##### getWidth()和getMeasuredWidth()的区别

```
	/**
     * measure过程中计算出的宽度
     */
    @ViewDebug.ExportedProperty(category = "measurement")
    int mMeasuredWidth;
    
    public final int getMeasuredWidth() {
        return mMeasuredWidth & MEASURED_SIZE_MASK;
    }

	/**
     * view的左/右边到负控件左/右边的距离，计算出的值为view的实际显示的宽度
     */
    @ViewDebug.ExportedProperty(category = "layout")
    protected int mLeft;
    @ViewDebug.ExportedProperty(category = "layout")
    protected int mRight;
    
	@ViewDebug.ExportedProperty(category = "layout")
    public final int getWidth() {
        return mRight - mLeft;
    }
```

getMeasuredWidth是通过从MeasuredSpec中提取出size而获得的宽度（measure阶段获取的宽度），一般来说即为控件的宽度（在显示前可被修改）；而getWidth获得的是View最终显示出的大小（layout阶段获取的宽度）。

为什么会改变大小呢？

```
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

      ... 
}
```

在layout阶段，父控件通过调用setOpticalFrame或setFrame来设置View的大小，并为mLeft，mRight赋值。如果在View的onLayout回调中对mLeft和mRight进行修改，即可改变View显示的大小。所以上文说过，getMeasuredWidth计算得出的是view原始的大小，也就是这个view在XML文件中配置或者是代码中设置的大小，而不一定是最终显示的大小。

---

##### 在onCreate中获取View的高度和宽度

View的大小计算和Activity的生命周期没有直接的关系，因此在onCreate、onStart、onResume中均无法正确的获得View的，需要通过其他回调的方式来获取：

```
	/**
     * 打印控件的宽和高
     */
    private void logTextViewWidth() {
        if (textView != null) {
            int width = textView.getWidth();
            int height = textView.getHeight();
            Log.d(TAG, "get: width=" + width + ",height=" + height);
            int measuredWidth = textView.getMeasuredWidth();
            int measuredHeight = textView.getMeasuredHeight();
            Log.d(TAG, "getMeasured: width=" + measuredWidth + ",height=" + measuredHeight);

        }
    }
```

1.View.post(runnable)

View.post会将runnable对象投递到消息队列的末尾；当被执行时，View的计算流程已经结束了，这时可以获取到View的具体高度。

```
		//1.view.post//1.view.post
        textView.post(new Runnable() {
            @Override
            public void run() {
                logTextViewWidth();
            }
        });

09-19 10:35:30.578 9822-9822/com.dz.zxing D/SecondActivity: onCreate: 
09-19 10:35:30.593 9822-9822/com.dz.zxing D/SecondActivity: onStart: 
09-19 10:35:30.597 9822-9822/com.dz.zxing D/SecondActivity: onResume: 
09-19 10:35:30.628 9822-9822/com.dz.zxing D/SecondActivity: get: width=120,height=51
09-19 10:35:30.628 9822-9822/com.dz.zxing D/SecondActivity: getMeasured: width=120,height=51
```

2.ViewTreeObserver.addOnGlobalLayoutListener(OnGlobalLayoutListener listener)

监听View的onLayout绘制过程，一旦onLayout被触发，立即回调onLayoutChange方法。

```
		//2.viewTreeObserver//2.viewTreeObserver
        final ViewTreeObserver observer = textView.getViewTreeObserver();
        observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                logTextViewWidth();
                textView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
            }
        });

09-19 10:40:36.041 10191-10191/com.dz.zxing D/SecondActivity: onCreate: 
09-19 10:40:36.057 10191-10191/com.dz.zxing D/SecondActivity: onStart: 
09-19 10:40:36.061 10191-10191/com.dz.zxing D/SecondActivity: onResume: 
09-19 10:40:36.087 10191-10191/com.dz.zxing D/SecondActivity: get: width=120,height=51
09-19 10:40:36.087 10191-10191/com.dz.zxing D/SecondActivity: getMeasured: width=120,height=51
```

注意：OnGlobalLayoutListener使用后要尽快移除掉，不然每一次onLayout的调用都会触发listener的回调，影响性能。

3.View.measure(int widthMeasureSpec, int heightMeasureSpec)

通过手动调用View的measure函数来得到宽高。这个方法比较复杂，需要根据View的LayoutParams来分情况处理：

1）match_parent

该情况直接放弃。因为该情况下View的宽高为父控件的剩余空间，而此时无法获取到父控件的剩余空间，所以理论上无法测量出View的大小。

2）具体的数值（dp／px）

比如宽高都是100px，如下measure：

```
		int widthMeasureSpec = View.MeasureSpec.makeMeasureSpec(100, View.MeasureSpec.EXACTLY);
        int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec(100, View.MeasureSpec.EXACTLY);
        textView.measure(widthMeasureSpec, heightMeasureSpec);
        logTextViewWidth();

09-19 11:30:16.295 12583-12583/com.dz.zxing D/SecondActivity: onCreate: 
09-19 11:30:16.298 12583-12583/com.dz.zxing D/SecondActivity: get: width=0,height=0
09-19 11:30:16.298 12583-12583/com.dz.zxing D/SecondActivity: getMeasured: width=100,height=100
09-19 11:30:16.313 12583-12583/com.dz.zxing D/SecondActivity: onStart: 
09-19 11:30:16.316 12583-12583/com.dz.zxing D/SecondActivity: onResume: 
```

3）wrap_content

如下measure：

```
		int widthMeasureSpec = View.MeasureSpec.makeMeasureSpec((1 << 30) - 1, View.MeasureSpec.AT_MOST);
        int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec((1 << 30) - 1, View.MeasureSpec.AT_MOST);
        textView.measure(widthMeasureSpec, heightMeasureSpec);
        logTextViewWidth();

09-19 11:34:28.281 12723-12723/com.dz.zxing D/SecondActivity: onCreate: 
09-19 11:34:28.282 12723-12723/com.dz.zxing D/SecondActivity: get: width=0,height=0
09-19 11:34:28.282 12723-12723/com.dz.zxing D/SecondActivity: getMeasured: width=120,height=51
09-19 11:34:28.284 12723-12723/com.dz.zxing D/SecondActivity: onStart: 
09-19 11:34:28.289 12723-12723/com.dz.zxing D/SecondActivity: onResume: 
```

注意到（1 << 30）-1，通过分析MeasureSpec的实现可以知道，View的尺寸使用30位的二进制标示，也就是说最大是30个1，在最大化模式下，我们用View的理论上能支持的最大值去构造MeasureSpec是合理的。

附：Activity中onWindowFocusChanged方法

onWindowFocusChanged方法表示，View已经初始化完毕，宽高已经准备好了，这时执行getMeasuredWidth()和getMeasuredHeight()方法，便可以获取宽高。onWindowFocusChanged会被调用多次，当Activity窗口得到或失去焦点时均会被调用。

```
	@Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (hasFocus) {
            logTextViewWidth();
        }
    }
```

---

### 参考资料

1.Android开发艺术探索

[2.Henry__Mark-Android开发中getMeasuredWidth为0时的解决方法](https://blog.csdn.net/henry__mark/article/details/64208378)

[3.一个逗逗-MeasureSpec类理解](https://blog.csdn.net/wning1/article/details/64137354)

[4.南小爵-MeasureSpec介绍及使用](https://www.cnblogs.com/nanxiaojue/p/3536381.html)
