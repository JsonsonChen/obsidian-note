#强引用 #弱引用 #软引用 #虚引用

 强引用

Java中的引用，类似于C++的指针。通过引用，可以对堆中的对象进行操作。在某个函数中，当创建了一个对象，该对象被分配在堆中，通过这个对象的引用才能对这个对象进行操作。
 ![[Pasted image 20240415221333.png]]
假设以上代码是在方法内运行的，那么局部变量str将被分配在栈空间上，而对象StringBuffer实例，被分配在堆空间中。局部变量str指向StringBuffer实例所在的堆空间，通过str可以操作该实例，那么str就是StringBuffer的引用。

![[Pasted image 20240415221214.png]]

此时，运行一个赋值语句：
![[Pasted image 20240415221355.png]]

那么，str所指向的对象也将被str1所指向，同时在局部栈空间上会分配空间存放str1变量。此时，该StringBuffer实例就有两个引用。对引用的”==”操作用于表示两个操作数所指向的堆空间地址是否相同，不表示两个操作数所指向的对象是否相等。

![[Pasted image 20240415221241.png]]

如果不使用时，要通过如下方式来弱化引用，如下：

str = null;     // 帮助垃圾收集器回收此对象
显式地设置str为null，或超出对象的生命周期范围，则gc认为该对象不存在引用，这时就可以回收这个对象。具体什么时候收集这要取决于gc的算法。

强引用特点：

强引用可以直接访问目标对象。
强引用所指向的对象在任何时候都不会被系统回收。JVM宁愿抛出OOM异常，也不会回收强引用所指向的对象。（除非将他显式地设置str为null）
强引用可能导致内存泄露。
软引用 (SoftReference)
软引用的介绍

java.lang.ref包中提供了几个类：SoftReference类、WeakReference类和PhantomReference类，它们分别代表软引用、弱引用和虚引用。ReferenceQueue类表示引用队列，它可以和这三种引用类联合使用，以便跟踪Java虚拟机回收所引用的对象的活动。

如果一个对象只具有软引用（就是说软引用指向了某个对象，但只要该对象不是强引用或没有被强引用指向），那么如果内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。

    ReferenceQueue queue = new ReferenceQueue();
    SoftReference ref= new SoftReference(aMyObject, queue);
    
那么当这个SoftReference所软引用的aMyOhject被垃圾收集器回收的同时，ref所强引用的SoftReference对象被列入ReferenceQueue。也就是说，ReferenceQueue中保存的对象是Reference对象，而且是已经失去了它所软引用的对象的Reference对象。另外从ReferenceQueue这个名字也可以看出，它是一个队列，当我们调用它的poll()方法的时候，如果这个队列中不是空队列，那么将返回队列前面的那个Reference对象。
在任何时候，我们都可以调用ReferenceQueue的poll()方法来检查是否有它所关心的非强可及对象被回收。如果队列为空，将返回一个null,否则该方法返回队列中前面的一个Reference对象。利用这个方法，我们可以检查哪个SoftReference所软引用的对象已经被回收。于是我们可以把这些失去所软引用的对象的SoftReference对象清除掉。

软引用的使用

使用场景举例：用来处理图片这种占用内存大的类的，所以，我们应该这样使用。

    View view = findViewById(R.id.button);
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.ic_launcher);
    Drawable drawable = new BitmapDrawable(bitmap);
    //把drawable 做成软引用
    SoftReference<Drawable> drawableSoftReference = 
                                                new SoftReference<Drawable>(drawable);
    Drawable bgdrawable = drawableSoftReference.get();
    if(bgdrawable != null) {
        view.setBackground(bgdrawable);
    }
这样的好处是
通过软引用的get()方法，取得drawable对象实例的强引用，发现对象被未回收。在GC在内存充足的情况下，不会回收软引用对象。此时view的背景显示。
实际情况中，我们会获取很多图片，然后可能给很多个view展示，这种情况下很容易内存吃紧导致oom，内存吃紧，系统开始会GC。

这次GC后，drawables.get()不再返回Drawable对象，而是返回null，这时屏幕上背景图不显示，说明在系统内存紧张的情况下，软引用被回收。

使用软引用以后，在OutOfMemory异常发生之前，这些缓存的图片资源的内存空间可以被释放掉的，从而避免内存达到上限，避免Crash发生。

需要注意的是，在垃圾回收器对这个Java对象回收前，SoftReference类所提供的get方法会返回Java对象的强引用，一旦垃圾线程回收该Java对象之后，get方法将返回null。所以在获取软引用对象的代码中，一定要判断是否为null，以免出现NullPointerException异常导致应用崩溃。

弱引用 (WeakReference)
弱引用的介绍

如果一个对象只具有弱引用（就是说弱引用指向了某个对象，但只要该对象不是强引用或没有被强引用指向），那么在垃圾回收器线程扫描的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。弱引用也可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。
弱引用与软引用的根本区别在于：只具有弱引用的对象拥有更短暂的生命周期，可能随时被回收。而只具有软引用的对象只有当内存不够的时候才被回收，在内存足够的时候，通常不被回收。

弱引用的使用

使用场景举例：结合handler的使用，防止内存泄露。

import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.support.v7.app.AppCompatActivity;
import java.lang.ref.WeakReference;
 
public class MainActivity extends AppCompatActivity {
 
    private Handler mHandler;
    private Thread mThread;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        mHandler = new MineHandler(this);
        mThread = new MineThread(this).start() ;
    }
 
    //静态内部类不持有外部类的引用（不持有Activity的引用）
    private static class MineHandler extends Handler {
        WeakReference<MainActivity> weakReference ;
 
        public MineHandler(MainActivity activity){
            weakReference  = new WeakReference<MainActivity>( activity) ;
        }
 
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (weakReference.get() != null ){
                //更新UI update android ui
 
            }
        }
    }
 
    //静态内部类不持有外部类的引用（不持有Activity的引用）
    private static class MineThread extends Thread {
 
           WeakReference<MainActivity> weakReference;
 
           public MineThread(MainActivity activity) {
                 this.weakReference = new WeakReference<MainActivity>(activity);
           }
 
           @Override
           public void run() {
                 super.run();
                 if (weakReference.get() != null){
                     //发送消息
                     mHandler.sendEmptyMessage(0);
                 }
                        
           }
    }
 
    @Override
    protected void onDestroy() {
 
        //3.MainActivity退出的时候，移除消息队列中所有消息和所有的Runnable，并且把它置为null。
        mHandler.removeCallbacksAndMessages(null);
        mHandler = null;
        super.onDestroy();
    }
}
在Android应用的开发中，为了防止内存溢出，在处理一些占用内存大而且声明周期较长的对象时候，可以尽量应用软引用和弱引用技术。
软引用，弱引用都非常适合来保存那些可有可无的缓存数据。如果这样做，当系统内存不足时，这些缓存数据会被回收，不会导致内存溢出。而当内存资源充足时，这些缓存数据又可以存在相当长的时间。

到底什么时候使用软引用，什么时候使用弱引用呢？
个人认为，如果只是想避免OutOfMemory异常的发生，则可以使用软引用。如果对于应用的性能更在意，想尽快回收一些占用内存比较大的对象，则可以使用弱引用。
还有就是可以根据对象是否经常使用来判断。如果该对象可能会经常使用的，就尽量用软引用。如果该对象不被使用的可能性更大些，就可以用弱引用。
另外，和弱引用功能类似的是WeakHashMap。WeakHashMap对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的回收，回收以后，其条目从映射中有效地移除。WeakHashMap使用ReferenceQueue实现的这种机制。

虚引用
虚引用是所有引用类型中最弱的一个。一个持有虚引用的对象，和没有引用几乎是一样的，随时都可能被垃圾回收器回收。当试图通过虚引用的get()方法取得强引用时，总是会失败。并且，虚引用必须和引用队列一起使用，它的作用在于跟踪垃圾回收过程。
当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在垃圾回收后，销毁这个对象，奖这个虚引用加入引用队列。
实际中几乎没用，暂不介绍。
最后一张图总结下：
![[Pasted image 20240415221140.png]]
                        
原文链接：https://blog.csdn.net/NakajimaFN/article/details/117922949