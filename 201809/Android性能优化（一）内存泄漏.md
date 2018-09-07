### 写在前面

这几天在学习使用LeakCanary，把自己写的一些Demo和公司项目app修改了一遍，总结了一些遇到的内存泄漏的现象，想把它们写下来记录一下分享给大家。但是针对app的优化不仅仅是内存泄漏，还有其他性能的优化、代码的优化等等，将来总是要去学习去实践的，所以开了一个专题，在工作中学习、运用、总结，不定时更新，如果有错误望不吝赐教。

---

### 概述

#### 内存泄漏

>内存泄漏（Memory Leak）是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。
内存泄漏缺陷具有隐蔽性、积累性的特征，比其他内存非法访问错误更难检测。因为内存泄漏的产生原因是内存块未被释放，属于遗漏型缺陷而不是过错型缺陷。此外，内存泄漏通常不会直接产生可观察的错误症状，而是逐渐积累，降低系统整体性能，极端的情况下可能使系统崩溃。

JVM中，GC会直接回收只有弱引用和虚引用的对象；当内存不足时，会回收软引用的对象；GC不会回收具有强引用的对象。JVM判断对象是否具有强引用有两种方式：引用技术法和可达性分析法。引用技术法即每个对象中添加一个计数器，每有一个强引用便加一，失去引用减一，当计数为零时即可被GC（这里存在一种情况是两个对象相互持有，所以出现了另一种对象存活判断方法）；可达性分析法是JVM根据一些算法计算出一些GC Root点，具有强引用的对象一定可以到达一个root点，若无法到达，即为无强引用的对象，如下图：

![可达性分析法](https://img-blog.csdn.net/20180906203153691?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzZDUwMTgyMzIwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

图中4，5，6三点即可被GC回收。

Android手机的内存、CPU、GPU等相对于PC来说还有这很大的差距，所以内存泄漏对于Android的影响更大，我们在平时开发中也要针对app进行优化。

---

### LeakCanary

内存泄漏是很隐秘的，它不会像bug一样直接爆出错误，甚至crash你的app让你知道它就在这里。[LeakCanary工具(链接至Github)](https://github.com/square/leakcanary)可以很简单的接到项目中，在运行中定位内存泄漏的位置。LeakCanary的使用方式很简单，如果不明白可以百度一篇使用方法，很简单。

---

### 内存泄漏分析

下面列出最近使用LeakCanary找出的内存泄漏位置～

##### 1.Context的持有

对Context的持有一定要谨慎，如果对象无法释放导致持有的Context对象不能释放，即Activity无法释放，会占用很大内存。

1.
**泄漏：** getSystemService的使用：因为通过Context.getSystemService来获取服务时，**部分系统服务**会将context保存为一个成员变量，有可能会发生内存泄漏。这时要注意以下几点：不能静态保存ServiceManager，使得其与Activity的生命周期一致；再一个就是对context的选取。使用Application context固然安全，但Activity对一些和UI相关的服务已经优先进行了处理。

**建议：**如果服务和UI相关，则用Activity context；
如果是类似ALARM_SERVICE,CONNECTIVITY_SERVICE可以优先选用Application Context；
如果发生内存泄漏，可以考虑使用application context。

2.
**泄漏：**静态对象持有：静态对象内部保存Context对象的话，生命周期会长于Activity，这就导致Activity被销毁时仍被引用，无法被回收。

**建议：**静态对象避免持有Context对象的引用；若必须持有，可以考虑代替不使用静态对象，或在Activity销毁时去掉持有，或对Application Context进行持有。

##### 2.Handler泄漏

**泄漏：** Handler的Message被存储在MessageQueue中未被处理，或是Message在处理过程中程序对Message进行了持有，例如获取msg.obj并应用于项目中，导致Message无法被释放，从而导致了Handler（非静态）无法被释放，引用Handler的Activity也无法释放。

**建议：** Handler声明为静态；
Handler中对Activity的引用使用弱引用；
当Activity销毁时，清空Handler的MessageQueue中的消息。

##### 3.Bitmap对象

**泄漏：** 

JVM中分配图像数据：Android 2.3.3之前，Bitmap像素数据和Bitmap对象是分开来存的。像素数据是存放在native memory中，而java对象存放在Dalvik heap中。因为native中的数据JVM无法直接去操作，需要通过调用底层方法去释放，可能导致应用程序暂时超过其内存限制和崩溃。Android 3.0之后都存放在Dalvik heap中，无需再手动调用recycle。

不在JVM中分配图像数据：不在JVM中创建的图像（如截屏）以及在JVM中创建但是被硬件加速（会产生TextureCache，与图像数据大小相当）。Bitmap在释放时延时处理TextureCache；而截屏之类操作创建的native图像数据是不会被释放的，只能通过调用recycle去释放。

**建议：** 使用createBitmap创建的图片对象且没有被硬件加速过，不需要调用recycle；
被Canvas draw加速过的图片会存在TextureCache，应该及时调用recycle来释放；
不在JVM中内分配内存的，应该及时调用recycle来释放。
（详见：参考资料 2）

##### 4.资源泄漏

**泄漏：**使用各种流或是打开文件，使用完未进行关闭操作。这会导致流没有关闭，无法释放；对应的Activity在销毁时也无法被释放。

**建议：**使用 try...finally 机制来确保流被关闭，而不管是否发生错误。

##### 5.非静态内部类的静态实例（[引自：刘望舒的博客](http://liuwangshu.cn/application/performance/ram-3-memory-leak.html)）

**泄漏：**非静态内部类会持有外部类实例的引用，如果非静态内部类的实例是静态的，就会间接的长期维持着外部类的引用，阻止被系统回收。匿名内部类也有对应问题。

```
public class SecondActivity extends AppCompatActivity {
    private static Object inner;
    private Button button;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        button = (Button) findViewById(R.id.bt_next);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                createInnerClass();
                finish();
            }
        });
    }

    void createInnerClass() {
        class InnerClass {
        }
        inner = new InnerClass();//1
    }
}
```

**建议：**使用静态内部类。

##### 6.耗时操作

**泄漏：**非静态变量（如AsyncTask、TimerTask）进行耗时操作，当Activity被销毁时，由于变量在后台进行耗时操作，无法被释放；对应Activity也无法被释放。

**建议：**声明为静态变量；
在Service中进行耗时操作的执行。

##### 7.未分类

这部分是一些小的注意事项，是我在修改项目时记录下来的，算是一些不好的编码习惯吧～

**泄漏：**

(1)：Activity包含多个Fragment，将fragment放入Stack中用于onBackPressed函数点击时的判断依据。后来由于一个业务处理，我将fragment的hide和show改成了remove和add，这样的话被remove的fragment因为依旧被栈持有无法释放，导致内存泄漏。

(2)：Activity中的弹窗使用DialogFragment并持有了一个本地变量，这使得dialogFragment调用dismiss后无法被释放。

(3)：本地使用的一些变量不需要对外暴露却使用为public关键字。

(4)：使用lambda表达式时，使用LeakCanary检测会出内存泄漏；替换掉lambda则泄漏提示消失。查阅网上资料并没有查到特别准确的回答（大佬们如果知道请在评论区教教我～），我的理解是lambda表达式的解析过程中，生成的对象会对某些变量持有导致无法正常释放。

**建议：**

(1)：为每个Fragment定义static final String常量，使用Stack保存String，创建fragment时将对应String作为tag；当触发onBackPressed时，使用findFragmentByTag来获取对应fragment：

```
	/**
     * 通过tag获取fragment；若无，则新建
     * @param tag tag
     * @return fragment
     */
    private Fragment getFragmentByTag(String tag) {
        Fragment fragment = getFragmentManager().findFragmentByTag(tag);
        if (fragment == null)
            fragment = getFragmentFromCode(tag);
        return fragment;
    }
```

(2)：与（1）类似，不保存本地变量，而是通过findFragmentByTag的方式去获取（减少不必要的本地变量）；
或是本地变量在使用后记得置为null。

(3)：根据需求使用private/protected关键字来替换。

(4)：从LeakCanary检测来看，不使用lambda表达式即可达到无泄漏的结果。更具体优化方案无法给出，请大佬们补充～

---

### 结尾

这篇文章也是因为想把最近看到的学到的东西和工作中的一些例子分享给大家，内容不是很充足，很多背景资料、知识拓展也不知如何去写。我自己总觉得看过的东西记不住，总是要整理一下码字码出来，才能记忆深刻一些。文章中如果有错误，请大佬们狠狠拍砖～ 和大家共同学习共同进步！

---

#### 参考资料

[1.刘望舒-Android内存优化（三）避免可控的内存泄漏](http://liuwangshu.cn/application/performance/ram-3-memory-leak.html)

[2.朱才-是否需要主动调用Bitmap的recycle方法](https://www.cnblogs.com/zhucai/p/5413340.html)