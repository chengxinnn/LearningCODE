# Handler



[https://blog.csdn.net/ly0724ok/article/details/117324053/](https://blog.csdn.net/ly0724ok/article/details/117324053/)



1. 简介
   Handler是一套 Android 消息传递机制,主要用于线程间通信。

用最简单的话描述： handler其实就是主线程在起了一个子线程，子线程运行并生成Message，Looper获取message并传递给Handler，Handler逐个获取子线程中的Message.

Binder/Socket用于进程间通信，而Handler消息机制用于同进程的线程间通信

可以说只要有异步线程与主线程通信的地方就一定会有 Handler。

在多线程的应用场景中，将工作线程中需更新UI的操作信息 传递到 UI主线程，从而实现 工作线程对UI的更新处理，最终实现异步消息的处理

使用Handler消息传递机制主要是为了多个线程并发更新UI的同时，保证线程安全

2. 相关概念解释
   Handler、Message、Message Queue、Looper

Message ：代表一个行为what或者一串动作Runnable, 每一个消息在加入消息队列时,都有明确的目标Handler

ThreadLocal： 线程本地存储区（Thread Local Storage，简称为TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。ThreadLocal的作用是提供线程内的局部变量TLS,这种变量在线程的生命周期内起作用,每一个线程有他自己所属的值(线程隔离)

MessageQueue (C层与Java层都有实现) ：以队列的形式对外提供插入和删除的工作, 其内部结构是以双向链表的形式存储消息的

Looper (C层与Java层都有实现) ：Looper是循环的意思,它负责从消息队列中循环的取出消息然后把消息交给Handler处理

Handler ：消息的真正处理者, 具备获取消息、发送消息、处理消息、移除消息等功能

Android消息机制：

以Handler的sendMessage方法为例，当发送一个消息后，会将此消息加入消息队列MessageQueue中。
Looper负责去遍历消息队列并且将队列中的消息分发给对应的Handler进行处理。
在Handler的handleMessage方法中处理该消息，这就完成了一个消息的发送和处理过程。
Handler示意图：

消息机制的模型：

Message：需要传递的消息，可以传递数据；
MessageQueue：消息队列，但是它的内部实现并不是用的队列，实际上是通过一个单链表的数据结构来维护消息列表，因为单链表在插入和删除上比较有优势。主要功能向消息池投递消息(MessageQueue.enqueueMessage)和取走消息池的消息(MessageQueue.next)；
Handler：消息辅助类，主要功能向消息池发送各种消息事件(Handler.sendMessage)和处理相应消息事件(Handler.handleMessage)；
Looper：不断循环执行(Looper.loop)，从MessageQueue中读取消息，按分发机制将消息分发给目标处理者。
消息机制的架构

在子线程执行完耗时操作，当Handler发送消息时，将会调用 MessageQueue.enqueueMessage ，向消息队列中添加消息。
当通过 Looper.loop 开启循环后，会不断地从线程池中读取消息，即调用 MessageQueue.next
然后调用目标Handler（即发送该消息的Handler）的 dispatchMessage 方法传递消息，然后返回到Handler所在线程，目标Handler收到消息，调用 handleMessage 方法，接收消息，处理消息。
3. Handler 的基本使用
3.1 创建 Handler
Handler 允许我们发送延时消息，如果在延时期间用户关闭了 Activity，那么该 Activity 会泄露。

这个泄露是因为 Message 会持有 Handler，而又因为 Java 的特性，内部类会持有外部类，使得 Activity 会被 Handler 持有，这样最终就导致 Activity 泄露。

解决方案：将 Handler 定义成静态的内部类，在内部持有 Activity 的弱引用，并及时移除所有消息。

public class HandlerActivity extends AppCompatActivity {

    private Button bt_handler_send;

    private static class MyHandler extends Handler {

    //弱引用持有HandlerActivity , GC 回收时会被回收掉
        private WeakReference`<HandlerActivity>` weakReference;

    public MyHandler(HandlerActivity activity) {
            this.weakReference = new WeakReference(activity);
        }

    @Override
        public void handleMessage(Message msg) {
            HandlerActivity activity = weakReference.get();
            super.handleMessage(msg);
            if (null != activity) {
                //执行业务逻辑
                Toast.makeText(activity,"handleMessage",Toast.LENGTH_SHORT).show();
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.HandlerActivity);

    //创建 Handler
        final MyHandler handler = new MyHandler(this);

    bt_handler_send = findViewById(R.id.bt_handler_send);
        bt_handler_send.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        //使用 handler 发送空消息
                        handler.sendEmptyMessage(0);

    }
                }).start();
            }
        });
    }

    @Override
    protected void onDestroy() {
        //移除所有回调及消息
        myHandler.removeCallbacksAndMessages(null);
        super.onDestroy();
    }
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
3.2 Message 获取
获取 Message 大概有如下几种方式：

Message message = myHandler.obtainMessage(); 		   //通过 Handler 实例获取
Message message1 = Message.obtain();   			      //通过 Message 获取
Message message2 = new Message();      				 //直接创建新的 Message 实例
1
2
3
通过查看源码可知，Handler 的 obtainMessage() 方法也是调用了 Message 的 obtain() 方法

public final Message obtainMessage()
{
    return Message.obtain(this);
}

1
2
3
4
5
通过查看 Message 的 obtain 方法

public static Message obtain(Handler h) {
        //调用下面的方法获取 Message
        Message m = obtain();
        //将当前 Handler 指定给 message 的 target ，用来区分是哪个 Handler 的消息
        m.target = h;

    return m;
    }

//从消息池中拿取 Message，如果有则返回，否则创建新的 Message
public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
为了节省开销，我们在使用的时候尽量复用 Message，使用前两种方式进行创建。

3.3 Handler 发送消息
Handler 提供了一些列的方法让我们来发送消息，如 send()系列 post()系列,post方法需要传入一个Runnalbe对象 ,我们来看看post方法源码

    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
1
2
3
4
不过不管我们调用什么方法，最终都会走到 MessageQueue.enqueueMessage(Message,long) 方法。
以 sendEmptyMessage(int) 方法为例：

//Handler
sendEmptyMessage(int)
  -> sendEmptyMessageDelayed(int,int)
    -> sendMessageAtTime(Message,long)
      -> enqueueMessage(MessageQueue,Message,long)
  			-> queue.enqueueMessage(Message, long);
1
2
3
4
5
6
从中可以发现 MessageQueue 这个消息队列，负责消息的入队，出队。

4. 两个实例（主线程-子线程）
   4.1 子线程向主线程
   首先我们在MainActivity中添加一个静态内部类，并重写其handleMessage方法。

private static class MyHandler extends Handler {
        private final WeakReference`<MainActivity>` mTarget;

    public MyHandler(MainActivity activity) {
            mTarget = new WeakReference`<MainActivity>`(activity);
        }

    @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            HandlerActivity activity = weakReference.get();
            super.handleMessage(msg);
            if (null != activity) {
                //执行业务逻辑
                if (msg.what == 0) {
                	Log.e("myhandler", "change textview");
                	MainActivity ma = mTarget.get();
	                ma.textView.setText("hahah");
            	}
                Toast.makeText(activity,"handleMessage",Toast.LENGTH_SHORT).show();
            }

    }
 }

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
然后创建一个类型为MyHandler的私有属性：

  private Handler handler1 = new MyHandler(this);
1
最后在onCreate回调中创建一个线程，用于接收发送消息：

new Thread(new Runnable() {
            @Override
            public void run() {
                handler1.sendEmptyMessage(0);
            }
        }).start();
1
2
3
4
5
6
总结一下：

Handler的子类对象一般是在主线程中进行创建，以便在两个线程中都能访问。我们创建了Handler类的子类MyHandler，并重写了handlerMessage方法，这个方法是当使用接收处理发送的消息的。然后我们创建了一个子线程，在子线程中我们使用MyHandler的对象调用sendEmptyMessage方法发送了一个空的Message。然后我们就能在主线程中接收到这个数据。
4.2 主线程向子线程
首先创建一个MyHandler类。

private static class MyHandler extends Handler {

    @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            if (msg.what == 0) {
                Log.e("child thread", "receive msg from main thread");
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
声明一个Handler类型的私有变量，进行默认初始化为null。

 private Handler handler1;
1
创建子线程，使handler指向新创建的MyHandler对象。

new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                handler1 = new MyHandler();
                Looper.loop();
                Log.e("child thread", "child thread end");
            }
        }).start();
1
2
3
4
5
6
7
8
9
在主线程中向子线程中发送消息

while (handler1 == null) {

    }

    handler1.sendEmptyMessage(0);
        handler1.getLooper().quitSafely();
1
2
3
4
5
6
5. Handler机制原理
5.1 Handler原理概述
普通的线程是没有looper的，如果需要looper对象，那么必须要先调用Looper.prepare方法，而且一个线程只能有一个looper

Handler是如何完成跨线程通信的？

Android中采用的是Linux中的 管道通信
关于管道，简单来说，管道就是一个文件
在管道的两端，分别是两个打开文件文件描述符，这两个打开文件描述符都是对应同一个文件，其中一个是用来读的，别一个是用来写的
消息队列创建时
调用JNI函数，初始化NativeMessageQueue对象。NativeMessageQueue则会初始化Looper对象
Looper的作用就是，当Java层的消息队列中没有消息时，就使Android应用程序主线程进入等待状态，而当Java层的消息队列中来了新的消息后，就唤醒Android应用程序的主线程来处理这个消息

Handler通过sendMessage()发送Message到MessageQueue队列
Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target来处理
经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理
将Message加入MessageQueue时，处往管道写入字符，可以会唤醒loop线程；如果MessageQueue中- 没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作
5.2 Handler与管道通信
管道，其本质是也是文件，但又和普通的文件会有所不同：管道缓冲区大小一般为1页，即4K字节。管道分为读端和写端，读端负责从管道拿数据，当数据为空时则阻塞；写端向管道写数据，当管道缓存区满时则阻塞。

在Handler机制中，Looper.loop方法会不断循环处理Message，其中消息的获取是通过 Message msg = queue.next(); 方法获取下一条消息。该方法中会调用nativePollOnce()方法，这便是一个native方法，再通过JNI调用进入Native层，在Native层的代码中便采用了管道机制。

我们知道线程之间内存共享，通过Handler通信，消息池的内容并不需要从一个线程拷贝到另一个线程，因为两线程可使用的内存时同一个区域，都有权直接访问，当然也存在线程私有区域ThreadLocal（这里不涉及）。即然不需要拷贝内存，那管道是何作用呢？

Handler机制中管道作用就是当一个线程A准备好Message，并放入消息池，这时需要通知另一个线程B去处理这个消息。线程A向管道的写端写入数据1（对于老的Android版本是写入字符W），管道有数据便会唤醒线程B去处理消息。管道主要工作是用于通知另一个线程的，这便是最核心的作用。

为什么采用管道而非Binder？

从内存角度：通信过程中Binder还涉及一次内存拷贝，handler机制中的Message根本不需要拷贝，本身就是在同一个内存。Handler需要的仅仅是告诉另一个线程数据有了。
从CPU角度，为了Binder通信底层驱动还需要为何一个binder线程池，每次通信涉及binder线程的创建和内存分配等比较浪费CPU资源。
5.3 Handler举例详解
Handler是Android消息机制的上层接口。Handler的使用过程很简单，通过它可以轻松地将一个任务切换到Handler所在的线程中去执行。通常情况下，Handler的使用场景就是更新UI

如下就是使用消息机制的一个简单实例：

public class Activity extends android.app.Activity {
	private Handler mHandler = new Handler(){
		@Override
		public void handleMessage(Message msg) {
			super.handleMessage(msg);
			System.out.println(msg.what);
		}
	};
	@Override
	public void onCreate(Bundle savedInstanceState, PersistableBundle persistentState) {
		super.onCreate(savedInstanceState, persistentState);
		setContentView(R.layout.activity_main);
		new Thread(new Runnable() {
			@Override
			public void run() {
				...............耗时操作
				Message message = Message.obtain();
				message.what = 1;
				mHandler.sendMessage(message);
			}
		}).start();
	}
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
再举个例子：怎么从主线程发送消息到子线程？（虽然这种应用场景很少）

Thread thread = new Thread(){
            @Override
            public void run() {
                super.run();
                //初始化Looper,一定要写在Handler初始化之前
                Looper.prepare();
                //在子线程内部初始化handler即可，发送消息的代码可在主线程任意地方发送
                handler=new Handler(){
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                      //所有的事情处理完成后要退出looper，即终止Looper循环
                        //这两个方法都可以，有关这两个方法的区别自行寻找答案
                        handler.getLooper().quit();
                        handler.getLooper().quitSafely();
                    }
                };

    //启动Looper循环，否则Handler无法收到消息
                Looper.loop();
            }
        };
        thread.start();
    //在主线程中发送消息
    handler.sendMessage();
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
先来解释一下第一行代码Looper.prepare();，先看看Handler构造方法

//空参的构造方法，这个方法调用了两个参数的构造方法
  public Handler() {
        this(null, false);
    }

//两个参数的构造方法
public Handler(Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
Handler的构造方法中会验证Looper，如果Looper为空，那么会抛出空指针异常
Handler在构造方法中还做了一件事，将自己的一个全局消息队列对象（mQueue）指向了Looper中的消息队列，即构造方法中的这行代码mQueue = mLooper.mQueue;
第二行代码初始化了Hanlder并且重写HandleMessage（）方法，没啥好讲的。

然后调用了Looper.loop()方法，后面会解释。

我们先来看看最后一行代码handler.sendMessage(message) 的主要作用：

先看看这行代码之后的代码执行流程：

sendMessage（）之后代码通过图中所示的几个方法，最终执行到了MessageQueue的enqueueMessage（）方法，我们来就看看它：

boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        synchronized (this) {
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
MessageQueue是一个单向列表结构，而MessageQueue 的 enqueueMessage（）方法主要做的事情就是将 Handler发送过来的 Message插入到列表中。
调用handler.senMessage()方法的时候，最终的结果只是将这个消息插入到了消息队列中
发送消息的工作已经完成，那么Looper是什么时候取的消息，来看看：

public static void loop() {
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            try {
                msg.target.dispatchMessage(msg);
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
这是一个死循环
这个循环的目的是从MessageQueue中取出消息
取消息的方法是MessageQueue.next()方法
取出消息后调用message.target对象的dispatchMessage()方法分发消息
循环跳出的条件是MessageQueue.next()方法返回了null
看到这里我们应该自然会想到，message.target.是哪个对象？dispatchMessage有什么作用？

public final class Message implements Parcelable {
 /*package*/ int flags;

    /*package*/ long when;

    /*package*/ Bundle data;

    /*package*/ Handler target;

    /*package*/ Runnable callback;

    // sometimes we store linked lists of these things
    /*package*/ Message next;
}
1
2
3
4
5
6
7
8
9
10
11
12
13
14
在Looper从MessageQueue中取出Message之后，调用了Handler的dispatchMessage（）方法
这里我们不禁要问，这个target指向了哪个Handler，再来看看之前的enqueueMessage
//Handler的方法
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
1
2
3
4
5
6
7
8
第一行代码就是，将message的target属性赋值为发送message的handler自身
Looper取出消息后，调用了发送消息的Handler的dispatchMessage（）方法，并且将message本身作为参数传了回去。到此时，代码的执行逻辑又回到了Handler中。
接着看handler的dispatchMessage（）方法

/**
    *handler的方法
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
看到这里面一个我们非常熟悉到方法了没有？—handleMessage（）方法，也是我们处理消息时候的逻辑。

最后再来一个系统的工作示意图：

6. Handler用法（android与java区别）
   6.1 刷新UI界面
   使用java：

new Thread( new Runnable() {
    public void run() {
    myView.invalidate();
    }
}).start();
1
2
3
4
5
可以实现功能，刷新UI界面。但是这样是不行的，因为它违背了单线程模型：Android UI操作并不是线程安全的并且这些操作必须在UI线程中执行。

Thread+Handler：
Handler来根据接收的消息，处理UI更新。Thread线程发出Handler消息，通知更新UI。

Handler myHandler = new Handler() {
    public void handleMessage(Message msg) {
    switch (msg.what) {
    case TestHandler.GUIUPDATEIDENTIFIER:
    myBounceView.invalidate();
    break;
    }
    super.handleMessage(msg);
    }
    };
1
2
3
4
5
6
7
8
9
10
class myThread implements Runnable {
    public void run() {
    while (!Thread.currentThread().isInterrupted()) {

    Message message = new Message();
    message.what = TestHandler.GUIUPDATEIDENTIFIER;

    TestHandler.this.myHandler.sendMessage(message);
    try {
    Thread.sleep(100);
    } catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    }
    }
    }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
6.2 定时器（延时操作）
使用java：
使用Java上自带的TimerTask类，TimerTask相对于Thread来说对于资源消耗的更低，引入import java.util.Timer; 和 import java.util.TimerTask;

public class JavaTimer extends Activity {

    Timer timer = new Timer();
    TimerTask task = new TimerTask(){
    public void run() {
    setTitle("hear me?");
    }
    };

    public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);

    timer.schedule(task, 10000);   // 延迟delay毫秒后，执行一次task。

    }
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
TimerTask + Handler：

public class TestTimer extends Activity {

    Timer timer = new Timer();
    Handler handler = new Handler(){
    public void handleMessage(Message msg) {
    switch (msg.what) {
    case 1:
    setTitle("hear me?");
    break;
    }
    super.handleMessage(msg);
    }

    };

    TimerTask task = new TimerTask(){
    public void run() {
    Message message = new Message();
    message.what = 1;
    handler.sendMessage(message);
    }
    };

    public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);

    timer.schedule(task, 10000);   // 延迟delay毫秒后，执行一次task。
    }
}

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
6.3 定时更新 UI
Runnable + Handler.postDelayed(runnable,time)：

   private Handler handler = new Handler();

    private Runnable myRunnable= new Runnable() {
    public void run() {
    if (run) {
    handler.postDelayed(this, 1000);
    count++;
    }
    tvCounter.setText("Count: " + count);

    }
    };
1
2
3
4
5
6
7
8
9
10
11
12
然后在其他地方调用

handler.post(myRunnable);
handler.post(myRunnable,time);
1
2
6.4 Handler实现延迟执行
Handler延迟2s执行一个runnable

Handler handler=new Handler();
Runnable runnable=new Runnable() {
 @Override
	public void run() {
		// TODO Auto-generated method stub
		if(xsLayout.getVisibility()==View.VISIBLE){
			xsLayout.setVisibility(View.GONE);
		}
	}
};
handler.postDelayed(runnable, 2000);
1
2
3
4
5
6
7
8
9
10
11
在runnable被执行之前取消这个定时任务

handler.removeCallbacks(runnable);
1
7. 总结
7.1 总结
Handler 的背后有着 Looper 以及 MessageQueue 的协助，三者通力合作，分工明确。

Looper ：负责关联线程以及消息的分发在该线程下从 MessageQueue 获取 Message，分发给Handler ；
MessageQueue ：是个队列，负责消息的存储与管理，负责管理由 Handler 发送过来的Message；
Handler : 负责发送并处理消息，面向开发者，提供 API，并隐藏背后实现的细节。
Handler 发送的消息由 MessageQueue 存储管理，并由 Loopler 负责回调消息到 handleMessage()。

线程的转换由 Looper 完成，handleMessage() 所在线程由 Looper.loop() 调用者所在线程决

7.2 注意
（1）一个线程有几个Handler？
多个，通常我们开发过程中就会new出不止一个Handler。
（2）一个线程有几个Looper？如何保证？
仅有1个Looper
Looper的构造是私有的，只有通过其prepare()方法构建出来，当调用了Looper的prepare()方法后，会调用ThreadLocal中的get()方法检查ThreadLocalMap中是否已经set过Looper？
如果有，则会抛出异常，提示每个线程只能有一个Looper，如果没有，则会往ThreadLocalMap中set一个new出来的Looper对象。
这样可以保证ThreadLocalMap和Looper一一对应，即一个ThreadLocalMap只会对应一个Looper。而这里的ThreadLocalMap是在Thread中的一个全局变量，也只会有一个，所以就可以保证一个Thread中只有一个Looper。
（3）Handler内存泄漏的原因？
内部类持有外部的引用。
Handler原理：由于Handler可以发送延迟消息，所以为了保证消息执行完毕后，由同一个Handler接收到，所以发送出去的Message中会持有Handler的引用，这个引用存在Message的target字段中，是Handler所有的sendMessage()方法最后都会调用enqueueMessage()，而在enqueueMessage()中会给Message的target字段赋值this。
因此Message持有Handler的引用，Handler又持有Activity的引用，所以在Message处理完之前，如果Activity被销毁了，就会造成内存泄漏。
怎么解决？可以使用static修饰Handler对象。
（4）为何主线程可以new Handler？如果想要在子线程中new Handler要做些什么准备？
因为在ActivityThread中的main()已经对Looper进行了prepar()操作，所以可以直接在主线程new Handler。
android-28的SystemServer类中：
main方法是整个android应用的入口，在子线程中调用Looper.prepare()是为了创建一个Looper对象，并将该对象存储在当前线程的ThreadLocal中，每个线程都会有一个ThreadLocal，它为每个线程提供了一个本地的副本变量机制，实现了和其它线程隔离，并且这种变量只在本线程的生命周期内起作用，可以减少同一个线程内多个方法之间的公共变量传递的复杂度。Looper.loop()方法是为了取出消息队列中的消息并将消息发送给指定的handler,通过msg.target.dispatchMassage()方法
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }

    private void run() {
        ......
        Looper.prepareMainLooper();
        ......
    }

1
2
3
4
5
6
7
8
9
10
11
12
13
14
如果想在子线程中new Handler，则需要先手动调用Looper的prepare()方法初始化Looper，再调用Looper的loop()方法使Looper运转。
      new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();			//初始化Looper
                new Handler(){
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                    }
                };
                Looper.loop();
            }
        })

1
2
3
4
5
6
7
8
9
10
11
12
13
14
（5）子线程中维护的Looper，消息队列无消息的时候的处理方案是什么？
如果不处理的话，会阻塞线程，处理方案是调用Looper的quitSafely()；
quitSafely()会调用MessageQueue的quit()方法，清空所有的Message，并调用nativeWake()方法唤醒之前被阻塞的nativePollOnce()，使得方法next()方法中的for循环继续执行，接下来发现Message为null后就会结束循环，Looper结束。如此便可以释放内存和线程。
（6）内部是如何确保线程安全的？
可以存在多个Handler往MessageQueue中添加数据（发消息时各个Handler可能处于不同线程）
添加消息的方法enqueueMessage()中有synchronize修饰，取消息的方法next()中也有synchronize修饰。
Handler的delay消息（延迟消息）时间准确吗？
由于上述的加锁操作，所以时间不能保证完全准确。
（7）使用Message时应该如何创建它？
使用Message的obtain()方法创建，直接new出来容易造成内存抖动。
内存抖动是由于频繁new对象，gc频繁回收导致，而且由于可能被别的地方持有导致无法及时回收所以会导致内存占用越来越高。
使用obtain()对内存复用，可以避免内存抖动的发生。其内部维护了一个Message池，其是一个链表结构，当调用obtain()的时候会复用表头的Message，然后会指向下一个。如果表头没有可复用的message则会创建一个新的对象，这个对象池的最大长度是50。
（8）使用Handler的postDelay后消息队列会有什么变化？
如果此时消息队列为空，不会执行，会计算消息需要等待的时间，等待时间到了继续执行。
（9）Looper死循环为什么不会导致应用卡死？
卡死就是ANR，产生的原因有2个：
1、在5s内没有响应输入的事件(例如按键，触摸等)，
2、BroadcastReceiver在10s内没有执行完毕。
事实上我们所有的Activity和Service都是运行在loop()函数中，以消息的方式存在，所以在没有消息产生的时候，looper会被block(阻塞)，主线程会进入休眠，一旦有输入事件或者Looper添加消息的操作后主线程就会被唤醒，从而对事件进行响应，所以不会导致ANR
简单来说looper的阻塞表明没有事件输入，而ANR是由于有事件没响应导致，所以looper的死循环并不会导致应用卡死。
6.3 接下来
接下来，让我们看看HandlerThread
————————————————
版权声明：本文为CSDN博主「Yawn__」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/ly0724ok/article/details/117324053/
