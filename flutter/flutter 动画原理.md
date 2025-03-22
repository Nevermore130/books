TickerProvider的作用创建一个Ticker对象

```dart

abstract class TickerProvider {
  /// Abstract const constructor. This constructor enables subclasses to provide
  /// const constructors so that they can be used in const expressions.
  const TickerProvider();

  /// Creates a ticker with the given callback.
  ///
  /// The kind of ticker provided depends on the kind of ticker provider.
  Ticker createTicker(TickerCallback onTick);
}

```

**Ticker回调是由[SchedulerBinding]驱动的**

FrameCallback：SchedulerBinding 类中有三个FrameCallback回调队列， 在一次绘制过程中，这三个回调队列会放在不同时机被执行

1. transientCallbacks：用于存放一些临时回调，一般存放动画回调。可以通过`SchedulerBinding.instance.scheduleFrameCallback` 添加回调
2. persistentCallbacks：用于存放一些持久的回调，不能在此类回调中再请求新的绘制帧，持久回调一经注册则不能移除。`SchedulerBinding.instance.addPersitentFrameCallback()`，这个回调中处理了布局与绘制工作。
3. postFrameCallbacks：在Frame结束时只会被调用一次，调用后会被系统移除，可由 `SchedulerBinding.instance.addPostFrameCallback()` 注册，注意，不要在此类回调中再触发新的Frame，这可以会导致循环刷新。

当每次系统绘制的之前，会回调到ui.windows的onBegainFrame，而这个onBegainFrame执行的是handleBeginFrame，将时间值回调给了每一个callback。这里注意的是，是在在绘制之前，因为我们一般在绘制之前去通过改变控件的属性值完成动画，而这个动作必须在绘制前完成

#### Ticker

```dart
/// Schedules a tick for the next frame.
///
/// This should only be called if [shouldScheduleTick] is true.
@protected
void scheduleTick({ bool rescheduling = false }) {
  _animationId = SchedulerBinding.instance.scheduleFrameCallback(_tick, rescheduling: rescheduling);
}

```

ticker传入了一个_tick对象到scheduleFrameCallback中

```dart
void _tick(Duration timeStamp) {
  assert(isTicking);
  assert(scheduled);
  _animationId = null;

  _startTime ??= timeStamp;
  _onTick(timeStamp - _startTime);

  // The onTick callback may have scheduled another tick already, for
  // example by calling stop then start again.
  if (shouldScheduleTick)
    scheduleTick(rescheduling: true);
}

```

```dart
Ticker(this._onTick, { this.debugLabel }) {
  assert(() {
    _debugCreationStack = StackTrace.current;
    return true;
  }());
}

```

onTicker是Ticker构造时候创建的。而Ticker的创建其实主要只在TickProvider中

```dart
@override
Ticker createTicker(TickerCallback onTick) {
  _ticker = Ticker(onTick, debugLabel: kDebugMode ? 'created by $this' : null);
  return _ticker;
}

```

那谁调用了这个createTicker呢，其实自然能联想到是AnimationController

```dart
AnimationController({
  double value,
  this.duration,
  this.reverseDuration,
  this.debugLabel,
  this.lowerBound = 0.0,
  this.upperBound = 1.0,
  this.animationBehavior = AnimationBehavior.normal,
  @required TickerProvider vsync,
}) : 
   _direction = _AnimationDirection.forward {
  _ticker = vsync.createTicker(_tick);
  _internalSetValue(value ?? lowerBound);
}

```

某个方法调用了scheduleTick之后，使的Ticker里面的回调函数`_tick(Duration elapsed)`被添加到transientCallbacks中，之后每次帧绘制之前候通过handleBeginFrame回调到这个方法

每次启动动画的时候我们一般使用animation.forward()

```dart
TickerFuture forward({ double from }) {
  _direction = _AnimationDirection.forward;
  //如果不为null 表示动画有起始值
  if (from != null)
    value = from;
  return _animateToInternal(upperBound);
}

TickerFuture _startSimulation(Simulation simulation) {
  _simulation = simulation;
  _lastElapsedDuration = Duration.zero;
  _value = simulation.x(0.0).clamp(lowerBound, upperBound);
 // 此处调用ticker的start
  final TickerFuture result = _ticker.start();
  _status = (_direction == _AnimationDirection.forward) ?
    AnimationStatus.forward :
    AnimationStatus.reverse;
  _checkStatusChanged();
  return result;
}


```

当调用animationController的forward的时候，最终会调到ticker的start方法，start方法调用scheduleTick开始了ticker与ScheduleBinding的绑定。

建立好绑定之后，每次帧绘制之前，都会回调到AnimationController的_tick方法中

```dart
void _tick(Duration elapsed) {
  _lastElapsedDuration = elapsed;
  final double elapsedInSeconds = elapsed.inMicroseconds.toDouble() / Duration.microsecondsPerSecond;
  assert(elapsedInSeconds >= 0.0);
  _value = _simulation.x(elapsedInSeconds).clamp(lowerBound, upperBound);
  if (_simulation.isDone(elapsedInSeconds)) {
    _status = (_direction == _AnimationDirection.forward) ?
      AnimationStatus.completed :
      AnimationStatus.dismissed;
    stop(canceled: false);
  }
  notifyListeners();
  _checkStatusChanged();
}

```

通过``_simulation.x(elapsedInSeconds).clamp(lowerBound, upperBound)``计算出value值，然后由 notifyListeners()回调给我们的具体使用。





















