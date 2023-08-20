## android Handler
#问题：
#1.请你介绍与一下Handler消息机制
 答：Handler是Android中用于线程间通信的一种机制，一个handler对应一个线程，它只能在所属线程里面发送消息，一个线程可拥有多个handler，他们将消息发送给LoopQueue，由Looper统一调度；一个线程也只有一个Looper；主要作用是用于异步消息发送；Handler将消息通过sendMessage的方式将消息发送至LoopQueue，并有Looper将消息取出并发送给对应的Handler；每一个Looper都对应一个消息队列，在Looper创建时创建，Looper的创建时机是在线程执行Looper.prepare方法时，此方法保证在单个线程中只能有一个Looper,以下是Looper中获取Looper的方法
 
 ```
  public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

```
创建Looper方法如下
```
  private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
任何一个线程要使用Handler之前都必须使用prepare方法，不然就会抛出异常
```
public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
以上方式已被@Deprecated，其实从上面也可以看到，创建Handler主要是为其中的几个变量赋值，即:mLooper,mCallback,mAsyncronous,
```
 @UnsupportedAppUsage
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
mLooper一般是由Looper.preppare时创建，ThreadLocal是Looper自带的，一个全局的静态变量；
ThreadLocal维护一个线程、Value的map，在Looper里面，Value就是一个个Looper的实例
#问题：
#Handler引起的内存泄露原因及解决方案
#原因
在使用Hanlder时，通常会重写handler的handleMessage，或者在handler中创建callback，抑或是传送message的时候自带calllBack，在里面更新UI，不可避免的就要持有activity或者fragment；若该方法在界面销毁时还没有执行完毕，那么界面就没法正常GC；下面理一下
callBack持有activity/fragment，message持有callback，MessageQueue持有message，Looper持有MessageQueue（在Handler里面也有mQueue，但创建MessageQueue仍然是Looper的事，Handler的mQueue是从外面传进来的）
```
 @UnsupportedAppUsage
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
Looper持有handler，而ThreadLoccal持有Looper，ThreadLocal则是一个静态类，为GCroot，这一条GCC路径就不会被回收。换言之，只要在一个线程中使用了Looper.prepare，那么此线程自带的Looper便会存在，除非线程销毁了；但一般情况下使用Looper都在主线程中，可销毁不得。
那么，Looper创建后什么时候才会销毁呢。从上面就可以看出，Looper是跟线程的生命周期绑定的，除非线程销毁，不然Looper一般是不会主动销毁的。那么Looper是如何跟Thread深度绑定的呢，答案就是Thrad，线程本身就存在一个ThreadMap，可以储存一些线程的私有变量，在Looper.myLooper时，就通过线程的getMap方法拿到<ThreadLocal,Looper>的唯一looper，虽然ThreadLocal是同一个，但线程不是同一个，所以<ThreadLocal,Looper>中的looper仍然不一样，这样就能在不同线程维护不同的循环队列了。

```
ThreadLocal.get():
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
#解决方法
1.java
```
import android.app.Activity;
import android.os.Handler;
import android.os.Message;

import java.lang.ref.WeakReference;

public class MyActivity extends Activity {
    private final MyHandler handler = new MyHandler(this);

    // ...

    private static class MyHandler extends Handler {
        private final WeakReference<MyActivity> activityRef;

        MyHandler(MyActivity activity) {
            activityRef = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MyActivity activity = activityRef.get();
            if (activity != null) {
                // 使用 activity
            }
        }
    }
}

```
kotlin
```
class MyActivity : AppCompatActivity() {
    private val handler = MyHandler(this)

    // ...
}

class MyHandler(activity: MyActivity) : Handler() {
    private val activityRef: WeakReference<MyActivity> = WeakReference(activity)

    override fun handleMessage(msg: Message) {
        val activity = activityRef.get()
        if (activity != null) {
            // 使用 activity
        }
    }
}

```
可见，在java和kotlin中都使用了弱引用，弱引用在垃圾回收时当GC收集器发现对象存在弱引用时，仍会回收该对象，当是强引用时则不会回收。
上方java和Kotlin代码有所不同，这是因为在Java中静态内部类不会持有外部引用，在kotlin中由于语法糖的存在，将其简化了，嵌套类默认作为静态内部类处理，但如果使用了inner，那么就可以访问外部类的实例，自然也持有了外部类。同样的，匿名内部类也能访问外部类的成员，它也持有外部类，我们平时使用的lambda一大部分就是创建了很多内部类，要注意lambda中的方法执行时间很长,的情况，会不会导致内存泄漏。


#问题
#Handler，Thread，HandlerThread之前的区别
脑瘫问题，没什么关系，只是为了实现各自的功能，引用了对方罢了，Handler使用实现了跨线程的异步方式，HandlerThread就是封装了Handler的Thread。

 
