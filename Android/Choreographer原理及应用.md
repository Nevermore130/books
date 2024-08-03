### 刷新率

刷新率代表屏幕在一秒内刷新屏幕的次数，这个值用赫兹来表示，取决于硬件的固定参数。这个值一般是60Hz，即每16.66ms刷新一次屏幕。

### 帧速率

帧速率代表了GPU在一秒内绘制操作的帧数，比如30FPS/60FPS。在这种情况下，高点的帧速率总是好的。

### VSYNC

刷新率和帧速率需要协同工作，才能让应用程序的内容显示到屏幕上，GPU会获取图像数据进行绘制，然后硬件负责把内容呈现到屏幕上，这将在应用程序的生命周期中周而复始地发生。

**刷新率和帧速率并不是总能够保持相同的节奏：**

- **如果帧速率实际上比刷新率快**

那么就会出现一些视觉上的问题，当帧速率在100fps而刷新率只有75Hz的时候，GPU所渲染的图像并非全部都被显示出来。

- **屏幕刷新率比帧速率快的情况**

如果屏幕刷新率比帧速率快，屏幕会在两帧中显示同一个画面。此时用户会很明显地察觉到动画卡住了或者掉帧，然后又恢复了流畅，这通常被称为闪屏，跳帧，延迟。

**VSYNC是为了解决屏幕刷新率和GPU帧率不一致导致的“屏幕撕裂”问题**。

### FPS

FPS：Frame Per Second，即每秒显示的帧数，也叫帧率。Android设备的FPS一般是60FPS，即每秒刷新60次，也就是60帧，每一帧的时间最多只有`1000/60=16.67ms`。一旦某一帧的绘制时间超过了限制，就会发生掉帧，用户在连续两帧会看到同样的画面。也就是上面说的屏幕刷新率比帧速率快的情况。

### ViewRootImpl.setView()

ViewRootImpl.setView()是在什么时候被调用的：`ActivityThread.handleResumeActivity()->WindowManagerImpl.addView()->WindowManagerGlobal.addView()->ViewRootImpl.setView()`

```java
/**
* We have one child
*/
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            ...

            //注释1 开始三大流程（测量、布局、绘制）
            requestLayout();
            ...
            //注释2 添加View到WindowManagerService,这里是利用Binder跨进程通信，调用Session.addToDisplay()
            //将Window添加到屏幕
            res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
            ...
        }
    }
}
```

从ViewRootImpl.requestLayout()开始，既是View的首次绘制流程

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

### Choreographer 编舞者

```java
//ViewRootImpl.java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

void scheduleTraversals() {
    //注释1 标记是否已经开始了，开始了则不再进入
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //注释2 同步屏障，保证绘制消息（是异步的消息）的优先级
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //注释3 监听VSYNC信号，下一次VSYNC信号到来时，执行给进去的mTraversalRunnable
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        //标记已经完成
        mTraversalScheduled = false;
        //移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        //开始三大流程 measure layout draw
        performTraversals();
        ...
    }
}
```



1. 在一次VSYNC信号期间多次调用scheduleTraversals是没有意义的，所以用了个标志位标记一下
2. 发送了一个屏障消息，让同步的消息不能执行，只能执行异步消息，而绘制的消息是异步的，保证了绘制的消息的优先级。绘制任务肯定高于其他的同步任务的。关于Handler同步屏障的具体详情可以阅读一下文章[Handler同步屏障](https://link.zhihu.com/?target=https%3A//github.com/xfhy/Android-Notes)
3. 利用Choreographer，调用了它的postCallback方法，暂时不知道拿来干嘛的，后面详细介绍

### Choreographer 初始化

```java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    //Binder代理IWindowSession，与WMS通信
    mWindowSession = WindowManagerGlobal.getWindowSession();
    mDisplay = display;
    //初始化当前线程  一般就是主线程，一般是在WindowManagerGlobal.addView()里面调用的
    mThread = Thread.currentThread();
    mWidth = -1;
    mHeight = -1;
    //Binder代理 IWindow
    mWindow = new W(this);
    //当前是不可见的
    mViewVisibility = View.GONE;
    mFirst = true; // true for the first time the view is added
    mAdded = false;
    ...
    //初始化Choreographer，从getInstance()方法名，看起来像是单例
    mChoreographer = Choreographer.getInstance();
    ...
}
```

Choreographer的构造方法

```java
//Choreographer.java
private Choreographer(Looper looper, int vsyncSource) {
    //把Looper传进来放起
    mLooper = looper;
    //FrameHandler初始化  传入Looper
    mHandler = new FrameHandler(looper);
    // USE_VSYNC 在 Android 4.1 之后默认为 true，
    // FrameDisplayEventReceiver是用来接收VSYNC信号的
    mDisplayEventReceiver = USE_VSYNC
            ? new FrameDisplayEventReceiver(looper, vsyncSource)
            : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;

    //一帧的时间，60FPS就是16.66ms
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

    // 回调队列
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
    // b/68769804: For low FPS experiments.
    setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
}
```

### Choreographer 流程原理

现在我们来说一下Choreographer的postCallback()，也就是ViewRootImpl使用的地方

```java
//ViewRootImpl是使用的这个
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}
public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    ...
    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}
private final CallbackQueue[] mCallbackQueues;

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        //将mTraversalRunnable存入mCallbackQueues数组callbackType处的队列中
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        //传入的delayMillis是0，这里dueTime是等于now的
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

```java
//Choreographer.java
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            //走这里

            // 如果当前线程是初始化Choreographer时的线程，直接申请VSYNC，否则立刻发送一个异步消息到初始化Choreographer时的线程中申请VSYNC
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            //这里是未开启VSYNC的情况，Android 4.1之后默认开启
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            if (DEBUG_FRAMES) {
                Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
            }
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

通过调用scheduleVsyncLocked()来监听VSYNC信号，这个信号是由硬件发出来的，信号来了的时候才开始绘制工作。

```java
//Choreographer.java
private final FrameDisplayEventReceiver mDisplayEventReceiver;

private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}

private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
    ...
}

//DisplayEventReceiver.java
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        //注册监听VSYNC信号，会回调dispatchVsync()方法
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

mDisplayEventReceiver是一个FrameDisplayEventReceiver，FrameDisplayEventReceiver继承自DisplayEventReceiver。在DisplayEventReceiver里面有一个方法scheduleVsync()，这个方法是用来注册监听VSYNC信号的，它是一个native方法



当有VSYNC信号来临时，native层会回调DisplayEventReceiver的dispatchVsync方法

```java
//DisplayEventReceiver.java
// Called from native code.
@SuppressWarnings("unused")
private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
    onVsync(timestampNanos, builtInDisplayId, frame);
}
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
}
```

当收到VSYNC信号时，回调dispatchVsync方法，走到了onVsync方法，这个方法被子类FrameDisplayEventReceiver覆写了的

```java
//FrameDisplayEventReceiver.java  
//它是Choreographer的内部类
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
            Log.d(TAG, "Received vsync from secondary display, but we don't support "
                    + "this case yet.  Choreographer needs a way to explicitly request "
                    + "vsync for a specific display to ensure it doesn't lose track "
                    + "of its scheduled vsync.");
            scheduleVsync();
            return;
        }

        //timestampNanos是VSYNC回调的时间戳  以纳秒为单位
        long now = System.nanoTime();
        if (timestampNanos > now) {
            timestampNanos = now;
        }

        if (mHavePendingVsync) {
            Log.w(TAG, "Already have a pending vsync event.  There should only be "
                    + "one at a time.");
        } else {
            mHavePendingVsync = true;
        }

        mTimestampNanos = timestampNanos;
        mFrame = frame;
        //自己是一个Runnable，把自己传了进去
        Message msg = Message.obtain(mHandler, this);
        //异步消息，保证优先级
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        ...
        doFrame(mTimestampNanos, mFrame);
    }
}
```

在onVsync()方法中，其实主要内容就是发个消息（应该是为了切换线程），然后执行run方法。而在run方法中，调用了Choreographer的doFrame方法。这个方法有点长，我们来理一下

```java
//Choreographer.java

//frameTimeNanos是VSYNC信号回调时的时间
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }
        ...
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        //jitterNanos为当前时间与VSYNC信号来时的时间的差值，如果Looper有很多异步消息等待处理（或者是前一个异步消息处理特别耗时，当前消息发送了很久才得以执行），那么处理当来到这里时可能会出现很大的时间间隔
        final long jitterNanos = startNanos - frameTimeNanos;

        //mFrameIntervalNanos是帧间时长，一般手机上为16.67ms
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            //想必这个日志大家都见过吧，主线程做了太多的耗时操作或者绘制起来特别慢就会有这个
            //这里的逻辑是当掉帧个数超过30，则输出相应日志
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            frameTimeNanos = startNanos - lastFrameOffset;
        }
        ...
    }

    try {
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        //执行回调
        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
    }
}
```

doFrame大体做了2件事，一个是可能会给开发者打个日志提醒下卡顿，另一个是执行回调。

当VSYNC信号来临时，记录了此时的时间点，也就是这里的frameTimeNanos。**而执行doFrame()时，是通过Looper的消息循环来的**，这意味着前面有消息没执行完，那么当前这个消息的执行就会被阻塞在那里。时间太长了，而这个是处理界面绘制的，如果时间长了没有即时进行绘制，就会出现掉帧。源码中也打了log，在掉帧30的时候。

下面来看一下执行回调的过程

```java
//Choreographer.java
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        final long now = System.nanoTime();
        //根据callbackType取出相应的CallbackRecord
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        mCallbacksRunning = true;
        ...
    }
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            //
            c.run(frameTimeNanos);
        }
    } finally {
        ...
    }
}
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            //会走到这里来，因为ViewRootImpl的scheduleTraversals时，postCallback传过来的token是null。
            ((Runnable)action).run();
        }
    }
}
```

从mCallbackQueues数组中找到callbackType对应的CallbackRecord，然后执行队列里面的所有元素（CallbackRecord）的run方法。然后也就是**执行到了ViewRootImpl的scheduleTraversals时，postCallback传过来的mTraversalRunnable（是一个Runnable）**。回顾一下:

```java
//ViewRootImpl.java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

整个流程也就完成了，从doTraversal()开始就是View的三大流程（measure、layout、draw）了。Choreographer的使命也基本完成了。



Choreographer的工作流程。简单总结一下：

1. 从ActivityThread.handleResumeActivity开始，`ActivityThread.handleResumeActivity()->WindowManagerImpl.addView()->WindowManagerGlobal.addView()->初始化ViewRootImpl->初始化Choreographer->ViewRootImpl.setView()`
2. 在ViewRootImpl的setView中会调用`requestLayout()->scheduleTraversals()`,然后是建立同步屏障
3. 通过Choreographer线程单例的postCallback()提交一个任务mTraversalRunnable，这个任务是用来做View的三大流程的（measure、layout、draw）
4. Choreographer.postCallback()内部通过DisplayEventReceiver.nativeScheduleVsync()向系统底层注册VSYNC信号监听，当VSYNC信号来临时，会回调DisplayEventReceiver的dispatchVsync()，最终会通知FrameDisplayEventReceiver.onVsync()方法。
5. 在onVsync()中取出之前传入的任务mTraversalRunnable，执行run方法，开始绘制流程。







