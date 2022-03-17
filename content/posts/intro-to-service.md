---
title: "Intro to Service"
date: 2020-07-04T11:08:11+08:00
description: "Introduction to Android Service"
tags: ["android", "service"]
categories: ["programming", "android"]
--- 

<!--more-->

## 查看进程基本信息

使用 `adb shell ps|grep com.tencent.mobileqq` 可以查看 QQ 应用进程相关的基本信息

``` shell
Zac:tivi Zac$ adb shell ps | grep com.tencent.mobileqq
# curr_user   pid                                                 process name
u0_a163       6779   669 1638852  27152 0                   0 S com.tencent.mobileqq:MSF
u0_a163      22001   669 1915536 224004 0                   0 S com.tencent.mobileqq
u0_a163      27829   669 1670960 236668 0                   0 S com.tencent.mobileqq:qzone
```

## 进程的划分

为了确定在内存不足时 kill 哪些进程，Android 会依据进程中运行的组件（Activity/Service/BroadcastReceiver）和这些组件的状态，划分每个进程的重要层次结构（importance hierarchy）。这些进程按重要性排序:

### 前台进程（foreground process）

前台进程是用户当前正在执行的操作所必须的进程，应用进程内的组件可能以不同形式让其处于前台：

+ Activity 处于用户正在交互状态（onResume() 方法已调用）
+ BroadcastReceiver 正在运行（BroadcastReceiver.onReceive() 方法正在执行）
+ Service 的回调正在执行代码（Service.onCreate(), Service.onStart(), Service.onDestroy()）

Android 系统中只有少数这样的进程，而且只有在内存太低且无法运行这些进程时，才会 kill 这些进程。

### 可见进程（visible process）

可见进程正在执行用户意识到的工作，因此 kill 这种进程可能对用户体验产生负面影响。这种进程的组件可以有如下形式：

+ Activity 可见但并不处于前台（onPause() 方法已调用），常见的例子比如在当前 Activity 展示一个弹窗。
+ 使用 *Service.startForeground()* 方法启动的前台服务。
+ 特定功能的服务，如动态壁纸，输入法服务等。

### 服务进程（service process）

服务进程是持有由 *Service.startService()* 方法启动的 Service 进程。尽管这些进程用户并不是直接可见，但它做的通常是用户关心的事情，如后台上传/下载网络数据。

长时间(> 30min)运行的服务可能会被降级，即下面介绍的缓存列表，这有助于避免长时间服务占用过多系统资源（如内存泄漏）。

### 缓存进程（cached process）

缓存进程是当前不需要的进程，当别的地方需要使用内存时，系统会优先 kill 掉这种进程。在一个运行良好的系统通常会有多个可用的缓存进程，并根据需要定期删除最老的进程。

这些进程通常包含一个或多个当前用户不可见的 Activity 实例（已调用 onStop() 方法），只要 App 正确的实现 Activity 对应的生命周期方法，当系统终止此类进程时，它不会影响用户返回该 App 时的体验（在新进程中重新创建关联 Activity 时，它可以恢复之前保存的状态）。

这些进程会被记录在缓存列表中，该列表的进程终止策略依赖平台的具体实现，原则上优先保留重要进程。其他的策略包括限制最大进程数量，以及限制进程运行时间等。

## 终止进程（Low Memory Killer Daemon, lmkd）

Android lmkd 进程监控正在运行的系统内存状态，并通过杀死最不重要进程（least essential processes）来应对高内存压力，以使系统运行在可接收的水平。

### 查看手机的内存阀值
  
```shell
# root permission need
adb shell cat /sys/module/lowmemorykiller/parameters/minfree
```

### ADJ

Adj 定义在 [frameworks/.../services/java/com/android/server/am/ProcessList.java][pl] 中，`oom_adj` 表示进程的 adj 值，一般在 -17～16 间取值，adj 值越大，优先级越低，`adj < 0`的进程都是系统进程。`adj = 0` 表示进程处于前台。

通过命令查看 `pid = 4061` 进程的 adj 值：

```shell
# App 处于前台时
adb shell cat /proc/4061/oom_adj
0
# App 处于后台时
Zac:tivi Zac$ adb shell cat /proc/4061/oom_adj
11
```

### 进程终止策略

`ProcessList` 中定义的 `oomAdj`：

```java
// These are the various interesting memory levels that we will give to
// the OOM killer.  Note that the OOM killer only supports 6 slots, so we
// can't give it a different value for every possible kind of process.
private final int[] mOomAdj = new int[] {
    FOREGROUND_APP_ADJ, VISIBLE_APP_ADJ, PERCEPTIBLE_APP_ADJ,
    BACKUP_APP_ADJ, CACHED_APP_MIN_ADJ, CACHED_APP_MAX_ADJ
};
```

在系统内存不足时，依次从 `CACHED_APP_MAX_ADJ -> .. ->  FOREGROUND_APP_ADJ` 终止进程。

看完进程我们再看看服务

## Service 的生命周期

Service 的生命周期比 Activity 的要简单很多。但关注其如何创建销毁反而更加重要，因为服务可以在用户没有意识到的情况下在后台运行。

Service 的生命周期可以遵循两条不同的途径：

+ 启动服务（Started service）
该服务在别的组件（client）中调用 [startService()][startservice] 时创建，然后无限运行，必须通过 [stopSelf()][stopself] 来自行停止运行。此外，其他组件也可以通过调用 [stopService()][stopservice] 来停止服务。服务停止后，系统会将其销毁。

  > 从 Android 8 开始，如果 app 不是位于前台，系统会对该 app 创建和使用后台服务施加限制，如果 app 需要创建前台服务，可以通过调用 [startForegroundService()][start-fore-service]。这个方法创建了一个后台service，并向系统发出信号，把自己的 service 提升到前台。该 service 启动之后必须在 5s 内调用自己的 startForeground() 方法，否则系统会自动 crash 进程。

+ 绑定服务（Bound service）
该服务在别的组件（client）中调用 [bindService()][bindservice] 时创建。然后 client 通过 [IBinder][ibinder] 接口与 service 进行通信。Client 可以通过调用 [unbindService()][unbindservice] 来关闭连接。多个 clients 可以绑定（bind）到相同的服务 ，而当所有服务解绑（unbind）之后 ，系统会销毁该服务（不必主动调用 [stopService()][stopService] 来停止服务)。

这两种途径并非完全独立，实际上是 **可以共存** 的。例如可以使用 Intent 调用 [startService()][startService] 启动后台音乐服务。随后，可能用户需要加入控制播放器获取有关播放歌曲信息时，Activity 可以通过调用 [bindService()][bindservice] 绑定到该服务。这种情况下，除非**所有客户端都取消绑定，否则 [stopService()][stopservice] 或 [stopSelf()][stopself] 不会停止该服务**。

## 实现 Service 生命周期回调

与 Activity 类似，Service 也拥有生命周期回调方法，可以通过实现这些方法来监控Service 状态的变化：

``` kotlin
import android.app.Notification
import android.app.PendingIntent
import android.app.Service
import android.content.Intent
import android.os.Build.VERSION_CODES
import android.os.IBinder
import androidx.annotation.RequiresApi

/**
 * Service code sample.
 *
 * @author: Zac
 * @date: 2022/3/16
 */
class SampleService : Service() {

  companion object {
    private const val NOTIFICATION_ID = 0x1001;
    private const val CHANNEL_NAME = "default";
  }

  // Indicates how to behave if the service is killed
  private var startMode: Int = START_STICKY
  // Interface for clients that bind
  private var binder: IBinder? = null
  // Indiacates whether onRebind should be used
  private var allowRebind: Boolean = false

  private val pendingIntent: PendingIntent =
    Intent(this, MainActivity::class.java).let { notificationIntent ->
      PendingIntent.getActivity(this, 0, notificationIntent, 0)
    }

  /**
   * Create foreground service
   */
  @RequiresApi(VERSION_CODES.O)
  private val notification: Notification = Notification.Builder(this, CHANNEL_NAME)
    .setContentTitle(getText(R.string.notification_title))
    .setContentText(getText(R.string.notification_message))
    .setSmallIcon(R.drawable.icon)
    .setContentIntent(pendingIntent)
    .setTicker(getText(R.string.ticker_text))
    .build()

  override fun onCreate() {
    // The service is being created
    startForeground(NOTIFICATION_ID, notification)
  }

  override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // The service is starting, due to call startService()
    return startMode
  }

  override fun onUnbind(intent: Intent?): Boolean {
    // All clients have unbound with unbindService()
    return allowRebind
  }

  override fun onRebind(intent: Intent?) {
    // A client is binding to the service with bindService,
    // after onUnbind() has already been called
  }

  override fun onBind(intent: Intent?): IBinder? {
    // A client is binding to the service with bindService()
    return binder
  }
}
```

> 与 Activity 生命周期的回调方法不同，Service 的生命周期回调方法不需要调用超类的方法（not required call super.onCreate()）

![service_lifecycle.png](/img/service_lifecycle.png)
> 服务的生命周期，左边展示了使用 [startService()][startService] 所创建的服务的生命周期，右边展示了 [bindService()][bindService] 所创建的服务的生命周期。

通过这些方法，我们可以监控服务生命周期的两个部分：

+ 服务的 **整个生命周期**(*entire lifetime*) 从调用 onCreate() 开始，到 onDestroy() 结束。与 Activity 类似，Service 也在 onCreate() 中完成初始化，并在 onDestroy() 中释放所有资源。例如，音乐播放器可以在 onCreate() 中创建播放音乐的线程，然后在 onDestroy() 中停止该线程。
  > 无论 Service 是通过 [startService()][startService] 还是 [bindService()][bindService] 方法创建，都会调用 onCreate() 和 onDestroy() 方法。

+ 服务的 **活动生命周期**(*active lifetime*) 从调用 [onStartCommand()][onstartcommand] 或 [onBind()][onbind] 方法开始。两种方法均接收 Intent 对象，该对象分别来自 [startService()][startService] 和 [bindService()][bindService] 。对于启动服务，活动生命周期和整个生命周期同时结束。对于绑定服务，活动生命周期在 [onUnbind()][onunbind] 返回时结束。

  > 尽管启动服务是通过 [stopSelf()][stopself] 或 [stopService()][stopservice] 来停止，但该服务没有相应的回调（没有 onStop() 回调）。因此，除非是绑定服务，否则在服务停止时，系统会将其在 onDestroy() 中销毁。

上图说明了服务的典型回调方法。尽管该图分开介绍通过 [startService()][startService] 方法和通过 [bindService()][bindService] 方法创建的服务，但是不管启动方式如何，任何服务均有可能允许客户端与其绑定。因此，最初使用 [onStartCommand()][onstartcommand]（客户端调用 [startService()][startService] ）启动的服务仍有可能接收 [onBind()][onbind] 的调用（客户端调用 [bindService()][bindService] 时）。

## 创建绑定服务

创建绑定服务时我们需要提供 IBinder，该 IBinder 可用于 client 与 service 交互的编程接口。有三种定义这种接口的方式：

+ **扩展 Binder 类**
  
  服务对 app 私有，且与客户端处于同一进程。服务通过 `onBind()` 方法提供 [Binder][binder] 实例，客户端接收 Binder 并可以使用它直接访问 Binder 实现或服务中的公共方法。

  对于服务仅用于自己 app 后台工作时，该方式为首选。唯一不用该方式创建服务的场景是服务被别的 app 使用或跨进程。
+ **使用 Messenger**

  对于跨进程的场景，可以使用 [Messenger][messenger] 为服务创建接口。这种方式服务定义 Handler 来处理不同的 Message 对象。这个 Handler 是 Messenger 的基础，它可以为客户端共享一个 IBinder，允许客户端使用 Message 对象。此外，客户端可以自己定义 Messenger，这样服务也可以送回 Message。

  这种是最简单的基于 Message 的跨进程（interprocess communication, IPC）的实现，因为 Messenger 将所有的请求加入到一个线程队列中，因此我们不必将服务设置为线程安全的。
+ **使用 AIDL**
  
  [Android Interface Definition Language AIDL][aidl] 将对象分解为操作系统可以理解的基础类型，并跨进程编组对象以执行 IPC。Messenger 的底层结构基于 AIDL 实现。上面 Messenger 在单线程中创建包含所有客户端请求的队列，因此服务一次只能接收一个请求。如果我们需要服务能同时处理多个请求，那么就需要用到 AIDL。这种情况，我们的服务就需要线程安全，并且支持多线程。

  要使用 AIDL，我们需要创建一个 `.aidl` 文件定义编程接口。Android SDK tools 使用该文件生成一个抽象类，该类实现编程接口和处理 IPC，然后我们可以在服务中对其扩展。

> 官方提示：大多数 app 不应使用 AIDL 来创建绑定服务，因为它可能需要多线程功能并且可能导致实现起来更加复杂。

### 2022/3/15 Some updates

#### IntentService

从 Android 8 开始，由于[限制后台服务执行][background-limit]，目前已**不推荐**使用 IntentService，并且该类已在 Android 11 废弃，我们可以使用 [JobIntentService][jobservice] 来替换 IntentService。

另外我们还可以使用扩展 Service 的方式来实现 IntentService 提供的功能：

``` kotlin
import android.app.Notification
import android.app.PendingIntent
import android.app.Service
import android.content.Intent
import android.os.Build.VERSION_CODES
import android.os.Handler
import android.os.HandlerThread
import android.os.IBinder
import android.os.Looper
import android.os.Message
import android.os.Process.THREAD_PRIORITY_BACKGROUND
import androidx.annotation.RequiresApi

/**
 * Service code sample.
 *
 * @author: Zac
 * @date: 2022/3/16
 */
class SampleIntentService : Service() {

  companion object {
    private const val NOTIFICATION_ID = 0x1001;
    private const val CHANNEL_NAME = "default";
  }

  private var serviceLooper: Looper? = null
  private var serviceHandler: ServiceHandler? = null

  private inner class ServiceHandler(looper: Looper): Handler(looper) {
    override fun handleMessage(msg: Message) {
      // to do some work here

      // Stop the service using the startId, so that we don't stop the
      // service in the middle of handling another job
      stopSelf(msg.arg1)
    }
  }

  private val pendingIntent: PendingIntent =
    Intent(this, MainActivity::class.java).let { notificationIntent ->
      PendingIntent.getActivity(this, 0, notificationIntent, 0)
    }

  /**
   * Create foreground notification
   */
  @RequiresApi(VERSION_CODES.O)
  private val notification: Notification = Notification.Builder(this, CHANNEL_NAME)
    .setContentTitle(getText(R.string.notification_title))
    .setContentText(getText(R.string.notification_message))
    .setSmallIcon(R.drawable.icon)
    .setContentIntent(pendingIntent)
    .setTicker(getText(R.string.ticker_text))
    .build()

  override fun onCreate() {
    // Start up the thread running the service.
    // We create a separate thread because the service normally runs in
    // the process's main thread, which we don't want block. We also make
    // it background priority so CPU-intensive work will not disrupt the UI.
    HandlerThread("SampleIntentService", THREAD_PRIORITY_BACKGROUND).apply {
      start()

      // Get the HandlerThread's Looper
      serviceLooper = looper
      serviceHandler = ServiceHandler(looper)
    }
  }

  override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    startForeground(NOTIFICATION_ID, notification)

    // For each start request, send a message to start a job and deliver the
    // start ID so we know which request we're stopping when we finish the job
    serviceHandler?.obtainMessage()?.also { msg->
      msg.arg1 = startId
      serviceHandler?.sendMessage(msg)
    }

    // If service get killed, after returning from here, restart
    return START_STICKY
  }

  override fun onBind(intent: Intent?): IBinder? {
    // No binder provide here
    return null
  }
}
```

### 何时使用绑定服务和启动服务

官方文档在 [startService][startservice] 有描述：

> Note: Each call to startService() results in significant work done by the system to manage service lifecycle surrounding the processing of the intent, which can take multiple milliseconds of CPU time. Due to this cost, startService() should not be used for frequent intent delivery to a service, and only for scheduling significant work. Use bound services for high frequency调用s.

大意是指 `startService()` 方法开销比较大，因此在高频次调用服务的场景，最好使用绑定服务，启动服务仅用于安排重要工作。

### Android O 以上报 IllegalStateException

Android O 加强了后台执行的限制，App 处于后台时不允许通过 `startService()` 的形式启动服务，只能通过 `startForegroundService()` 方法启动服务，而且 App 必须在创建服务后5s内调用该服务的 `startForeground()` 方法。具体的实现如下：

``` kotlin
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
  startForegroundService(new Intent(MainActivity.this, SampleService.class));
}else{
  startService(new Intent(MainActivity.this, SampleService.class));
}
```

[startservice]:https://developer.android.com/reference/android/content/Context.html#startService(android.content.Intent)
[stopself]:https://developer.android.com/reference/android/app/Service.html#stopSelf()
[stopservice]:https://developer.android.com/reference/android/content/Context.html#stopService(android.content.Intent)
[bindservice]:https://developer.android.com/reference/android/content/Context.html#bindService(android.content.Intent,android.content.ServiceConnection,int)
[ibinder]:https://developer.android.com/reference/android/os/IBinder.html
[binder]:https://developer.android.com/reference/android/os/Binder
[onstartcommand]:https://developer.android.com/reference/android/app/Service.html#onStartCommand(android.content.Intent,int,int)
[onbind]:https://developer.android.com/reference/android/app/Service.html#onBind(android.content.Intent)
[onunbind]:https://developer.android.com/reference/android/app/Service.html#onUnbind(android.content.Intent)
[unbindservice]:https://developer.android.com/reference/android/content/Context.html#unbindService(android.content.ServiceConnection)
[pl]:https://android.googlesource.com/platform/frameworks/base/+/be4e6aa/services/java/com/android/server/am/ProcessList.java
[notiy-sample]:https://github.com/googlearchive/android-NotificationChannels/blob/master/kotlinApp/Application/src/main/java/com/example/android/notificationchannels/NotificationHelper.kt
[background-limit]:https://developer.android.com/about/versions/oreo/background#services
[jobservice]:https://developer.android.com/reference/androidx/core/app/JobIntentService
[start-fore-service]:https://developer.android.com/reference/android/content/Context#startForegroundService(android.content.Intent)
[messenger]:https://developer.android.com/reference/android/os/Messenger
[aidl]:https://developer.android.com/guide/components/aidl
