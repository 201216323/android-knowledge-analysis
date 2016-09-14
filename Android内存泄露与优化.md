# Android内存泄露与优化

内存泄露是在Android开发中尤其要重视的一个问题，对开发人员开说，这是一个很容易犯也很常见的错误。优化内存泄露的问题，主要从两方面着手，一是开发人员避免写出有内存泄露的代码，二是通过一些诸如MAT的内存分析工具来找出潜在的内存泄露并解决它。

其实平时遇到的最多的情况，就是对Activity或Context保持一个长生命周期的引用。下面主要来分析一下造成内存泄露的各种原因。

## 一、静态变量导致的内存泄露
要不怎么说static关键字要慎用呢？来看看下面这段代码，Context对象为静态的，那么Activity就无法正常销毁，会常驻内存。
```java
public class MemoryActivity extends Activity{  
    public static Context mContext;  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        mContext = this;  
    }  
}  
```
这是一个很隐晦的内存泄漏的情况，在开发过程中，Context能使用ApplicationContext得尽量使用ApplicationContext，因为这个Context的生存周期和你的应用的生存周期一样长，而不是取决于activity的生存周期。除此之外，在Activity里面创建了静态的View，这就意味着该View持有一个对当前这个Activity的引用，那么Activity也是无法正常销毁的。

## 二、引用没释放导致的内存泄露
- **注册服务没取消导致的内存泄露**
假如我们在锁屏界面(LockScreen)中，监听系统中的电话服务以获取一些信息(如信号强度等)，则可以在LockScreen中定义一个PhoneStateListener的对象，同时将它注册到TelephonyManager服务中。对于LockScreen对象，当需要显示锁屏界面的时候就会创建一个LockScreen对象，而当锁屏界面消失的时候LockScreen对象就会被释放掉。但是如果在释放LockScreen对象的时候没有取消之前注册的PhoneStateListener对象，那么则会导致LockScreen无法被垃圾回收。而锁屏界面又不断的创建和销毁，则最终会由于大量的LockScreen对象没有办法被回收而引起OutOfMemory。类似的，BraodcastReceiver，ContentObserver，FileObserver在Activity onDeatory或者某类声明周期结束之后一定要unregister掉，否则这个Activity类会被system强引用，不会被内存回收。
- **集合中的对象没有及时清理导致的内存泄露**
当该集合为静态的时候，那么在集合里面对象越来越多的时候，最好要及时清理不需要用到的对象。

## 三、单例模式导致的内存泄露
单例模式的特点就是它的生命周期和Application一样，那么如果某个Activity实例被一个单例所持有，也就是说在单例里面引用了它，那么就会造成Activity对象无法正常回收释放。

## 四、资源对象未关闭导致的内存泄露
资源性对象(如Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。它们的缓冲不仅存在于Java虚拟机内，还存在于java虚拟机外。如果我们仅仅是把它的引用设置为null，而不关闭它们，往往会造成内存泄露。例如程序中经常会进行查询数据库的操作，但是经常会有使用完毕Cursor后没有关闭的情况。如果我们的查询结果集比较小，对内存的消耗不容易被发现，只有在常时间大量操作的情况下才会复现内存问题，这样就会给以后的测试和问题排查带来困难和风险。类似的，Bitmap在不需要之后，应该调用recycle回收，再置为null。

## 五、属性动画导致的内存泄露
例如下面的代码，由于该属性动画为循环动画，如果在Activity销毁时，没有取消动画，那么虽然我们看不见动画在执行，实际上动画仍然一直播放下去，这个时候Button会被动画所持有，而Button又持有对应的Activity对象，那么就会造成Activity无法正常释放。
```java
public class MemoryActivity extends Activity{  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        Button button = (Button) findViewById(R.id.btn_end);  
        ObjectAnimator animator = ObjectAnimator.ofFloat(button, "", 0,180);  
        animator.setDuration(2000);  
        animator.setRepeatCount(-1);  
        animator.start(); // 没有调用cancle()  
    }  
} 
```

## 六、Adapter未使用缓存的convertView导致的内存泄露
ListView提供每一个item所需要的view对象，初始时ListView会从BaseAdapter中根据当前的屏幕布局实例化一定数量的View对象，同时ListView会将这些view对象缓存起来。当向上滚动ListView时，原先位于最上面的Item的View对象会被回收，然后被用来构造新出现的最下面的Item。这个构造过程就是由getView()方法完成的，getView()的第二个形参View convertView就是被缓存起来的list item的view对象(初始化时缓存中没有view对象则convertView是null)。由此可以看出，如果我们不去使用 convertView，而是每次都在getView()中重新实例化一个View对象的话，即浪费资源也浪费时间，也会使得内存占用越来越大。正确的写法如下
```java
public View getView(int position, ViewconvertView, ViewGroup parent) {  
    View view = null;  
    if (convertView != null) { // 不应该直接new  
            view = convertView;   
            ...   
    } else {   
            view = new Xxx(...);  
            ...   
    }   
    return view;   
}  
```

## 七、Handler内部类内存泄露
当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有一个外部类对象（通常是一个Activity）的引用，而Handler通常会伴随着一个耗时的后台线程（例如从网络拉取图片）一起出现，这个后台线程在任务执行完毕（例如图片下载完毕）之后，通过消息机制通知Handler，然后Handler把图片更新到界面。然而，如果用户在网络请求过程中关闭了Activity，正常情况下，Activity不再被使用，它就有可能在GC检查时被回收掉，但由于这时线程尚未执行完，而该线程持有Handler的引用，这个Handler又持有Activity的引用，就导致该Activity无法被回收（即内存泄露），直到网络请求结束（例如图片下载完毕）。另外，如果你执行了Handler的postDelayed()方法，该方法会将你的Handler装入一个Message，并把这条Message推到MessageQueue中，那么在你设定的delay到达之前，会有一条MessageQueue -> Message -> Handler -> Activity的链，导致你的Activity被持有引用而无法被回收。可以在Activity结束后，关闭线程，如果你的Handler是被delay的Message持有了引用，那么调用removeCallbacks方法来移除消息队列。

内存泄露检测工具MAT的使用请参考：[http://jingyan.baidu.com/article/ae97a646b4eea8bbfc461d5a.html](http://jingyan.baidu.com/article/ae97a646b4eea8bbfc461d5a.html)
这里需要强调一点，有一些内存泄露通过Mat是查不出来的，比如native的代码，MAT对C/C++是无能为力的。
