#Google参考了Windows的消息处理机制，在Android系统中实现了一套类似的消息处理机制。学习Android的消息处理机制，有几个概念（类）必须了解：
##1.       Message
消息，理解为线程间通讯的数据单元。例如后台线程在处理数据完毕后需要更新UI，则可发送一条包含更新信息的Message给UI线程。
##2.       Message Queue
消息队列，用来存放通过Handler发布的消息，按照先进先出执行。
##3.       Handler
Handler是Message的主要处理者，负责将Message添加到消息队列以及对消息队列中的Message进行处理。
##4.       Looper
循环器，扮演Message Queue和Handler之间桥梁的角色，循环取出Message Queue里面的Message，并交付给相应的Handler进行处理。
 
##5．    线程
UI thread 通常就是main thread，而Android启动程序时会替它建立一个Message Queue。
每一个线程里可含有一个Looper对象以及一个MessageQueue数据结构。在你的应用程序里，可以定义Handler的子类别来接收Looper所送出的消息。
 
##运行机理：
  每个线程都可以并仅可以拥有一个Looper实例，消息队列MessageQueue在Looper的构造函数中被创建并且作为成员变量被保存，也就是说MessageQueue相对于线程也是唯一的。Android应用在启动的时候会默认会为主线程创建一个Looper实例，并借助相关的Handler和Looper里面的MessageQueue完成对Activities、Services、Broadcase Receivers等的管理。而在子线程中，Looper需要通过显式调用Looper. Prepare()方法进行创建。Prepare方法通过ThreadLocal来保证Looper在线程内的唯一性，如果Looper在线程内已经被创建并且尝试再度创建"Only one Looper may be created per thread"异常将被抛出。
 
  Handler在创建的时候可以指定Looper，这样通过Handler的sendMessage()方法发送出去的消息就会添加到指定Looper里面的MessageQueue里面去。在不指定Looper的情况下，Handler绑定的是创建它的线程的Looper。如果这个线程的Looper不存在，程序将抛出"Can't create handler inside thread that has not called Looper.prepare()"。
 
  **整个消息处理的大概流程是：**

- 1. 包装Message对象（指定Handler、回调函数和携带数据等）；
- 2. 通过Handler的sendMessage()等类似方法将Message发送出去；
- 3. 在Handler的处理方法里面将Message添加到Handler绑定的Looper的MessageQueue；
- 4. Looper的loop()方法通过循环不断从MessageQueue里面提取Message进行处理，并移除处理完毕的Message；
- 5. 通过调用Message绑定的Handler对象的dispatchMessage()方法完成对消息的处理。
 
  **在dispatchMessage()方法里，如何处理Message则由用户指定，三个判断，优先级从高到低：**

- 1. Message里面的Callback，一个实现了Runnable接口的对象，其中run函数做处理;
- 2.Handler里面mCallback指向的一个实现了Callback接口的对象，由其handleMessage进行处理；
- 3. 处理消息Handler对象对应的类继承并实现了其中handleMessage函数，通过这个实现的handleMessage函数处理消息。   
 
 
# Android的消息机制 #

 

> android 有一种叫消息队列的说法，这里我们可以这样理解：假如一个隧道就是一个消息队列，那么里面的每一部汽车就是一个一个消息，这里我们先忽略掉超车等种种因素，只那么先进隧道的车将会先出，这个机制跟我们android 的消息机制是一样的。

## 一、    角色描述 ##


- Looper:(相当于隧道)一个线程可以产生一个Looper对象，由它来管理此线程里的Message Queue(车队,消息隧道)。


- Handler:你可以构造Handler对象来与Looper沟通，以便push新消息到Message Queue里；或者接收Looper(从Message Queue取出)所送来的消息。


- Message Queue(消息队列):用来存放线程放入的消息。


- 线程：UI thread通常就是main thread，而Android启动程序时会替它建立一个Message Queue。*每一个线程里可含有一个Looper对象以及一个MessageQueue数据结构。在你的应用程序里，可以定义Handler的子类别来接收Looper所送出的消息。*
 
**在你的Android程序里，新诞生一个线程，或执行(Thread)时，并不会自动建立其Message Loop。
Android里并没有Global的Message Queue数据结构。**

例如：

- 不同APK里的对象不能透过Massage Queue来交换讯息(Message)。
- 线程A的Handler对象可以传递消息给别的线程，让别的线程B或C等能送消息来给线程A(存于A的Message Queue里)。线程A的Message Queue里的讯息，只有线程A所属的对象可以处理。
使用Looper.myLooper可以取得当前线程的Looper对象。使用`mHandler = new EevntHandler(Looper.myLooper());`可用来构造当前线程的Handler对象；其中，EevntHandler是自已实现的Handler的子类别。使用`mHandler = new EevntHandler(Looper.getMainLooper());`可诞生用来处理main线程的Handler对象；其中，EevntHandler是自已实现的Handler的子类别。