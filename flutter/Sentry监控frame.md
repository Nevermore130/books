#### Integration ####

```dart
/// Code that provides middlewares
abstract class Integration<T extends SentryOptions> {
  /// A Callable method for the Integration interface
  FutureOr<void> call(Hub hub, T options);

  /// NoOp by default : only closeable integrations need to override
  FutureOr<void> close() {}
}
```

#### FramesTrackingIntegration

```dart
class FramesTrackingIntegration implements Integration<SentryFlutterOptions>
```

call实现

```dart
@override
  Future<void> call(Hub hub, SentryFlutterOptions options) async {
  	//...
    final widgetsBinding = options.bindingUtils.instance;
    _widgetsBinding = widgetsBinding;

    final expectedFrameDuration = await _initializeExpectedFrameDuration();
    //init SentryDelayedFramesTracker
    final framesTracker =
        SentryDelayedFramesTracker(options, expectedFrameDuration);
    //register frametracking
     widgetsBinding.registerFramesTracking(
        framesTracker.addFrame, options.clock);
    //init SpanFrameMetricsCollector
    final collector = SpanFrameMetricsCollector(options, framesTracker);
    options.addPerformanceCollector(collector);
    _collector = collector;
  }
```

widgetsBinding是 ```mixin SentryWidgetsBindingMixin on WidgetsBinding```

SentryWidgetsBindingMixin 的实现：

```dart
//1. 系统WidgetsBinding中 handleBeginFrame记录_startTimestamp
//2. 系统WidgetsBinding中 handleDrawFrame  回调给callback frame的start和endtime
mixin SentryWidgetsBindingMixin on WidgetsBinding {
  DateTime? _startTimestamp;
  FrameTimingCallback? _frameTimingCallback;
  ClockProvider? _clock;
  @internal
  void registerFramesTracking(
      FrameTimingCallback callback, ClockProvider clock) {
    _frameTimingCallback ??= callback;
    _clock ??= clock;
  }

  @override
  void handleBeginFrame(Duration? rawTimeStamp) {
    _startTimestamp = _clock?.call();

    super.handleBeginFrame(rawTimeStamp);
  }

  @override
  void handleDrawFrame() {
    super.handleDrawFrame();

    final endTimestamp = _clock?.call();
    if (_startTimestamp != null &&
        endTimestamp != null &&
        _startTimestamp!.isBefore(endTimestamp)) {
      _frameTimingCallback?.call(_startTimestamp!, endTimestamp);
    }
  }
}
```

回到```widgetsBinding.registerFramesTracking(framesTracker.addFrame, options.clock);```  

就是注册一个让系统的每个frame信息回调给自定义的callback

回调的callback是SentryDelayedFramesTracker中的addFrame函数

```dart
//widgetsBinding 回调frame的start和end time
void addFrame(DateTime startTimestamp, DateTime endTimestamp) {
    if (!_isTrackingActive || !_options.enableFramesTracking) {
      return;
    }
    if (startTimestamp.isAfter(endTimestamp)) {
      return;
    }
    if (_delayedFrames.length > maxDelayedFramesBuffer) {
      // buffer is full, we stop collecting frames until all active spans have
      // finished processing
      pause();
      return;
    }
    final duration = endTimestamp.difference(startTimestamp);
    if (duration > _expectedFrameDuration) {
      final frameTiming = SentryFrameTiming(
          startTimestamp: startTimestamp, endTimestamp: endTimestamp);
      _delayedFrames.add(frameTiming);
      _oldestFrameEndTimestamp ??= endTimestamp;
    }
  }

```

* 如果一帧的渲染时间duration 大于阈值_expectedFrameDuration 则 实例化SentryFrameTiming 对象并添加到__delayedFrames 列表中，表示记录的 slow frame列表

##### SpanFrameMetricsCollector 对象

继承至 PerformanceCollector

```dart
/// Used for collecting continuous data about vitals (slow, frozen frames, etc.)
/// during a transaction/span.
abstract class PerformanceContinuousCollector extends PerformanceCollector {
  Future<void> onSpanStarted(ISentrySpan span);

  Future<void> onSpanFinished(ISentrySpan span, DateTime endTimestamp);

  void clear();
}

```

PerformanceCollector 的作用是在 每一个transaction 或者 span中 **收集** 相关的性能slow, frozen frames 的数据，比如记录的delayframe

所以SpanFrameMetricsCollector 主要作用就是收集delayframe的数据到span或者transaction中

```dart
class SpanFrameMetricsCollector implements PerformanceContinuousCollector {
   @override
  Future<void> onSpanStarted(ISentrySpan span) async {
    return _tryCatch('onSpanFinished', () async {
      if (span is NoOpSentrySpan) {
        return;
      }
      activeSpans.add(span);
      _frameTracker.resume();
    });
  }
  
  @override
  Future<void> onSpanFinished(ISentrySpan span, DateTime endTimestamp) async {
     return _tryCatch('onSpanFinished', () async {
      if (span is NoOpSentrySpan) {
        return;
      }

      final startTimestamp = span.startTimestamp;
       //_frameTracker 就是SentryDelayedFramesTracker
      final metrics = _frameTracker.getFrameMetrics(
          spanStartTimestamp: startTimestamp, spanEndTimestamp: endTimestamp);
      metrics?.applyTo(span);

      activeSpans.remove(span);
      if (activeSpans.isEmpty) {
        clear();
      } else {
        _frameTracker.removeIrrelevantFrames(activeSpans.first.startTimestamp);
      }
    });
  }
}
```

SentryDelayedFramesTracker 的getFrameMetrics

```dart
/// Calculates the frame metrics based on start, end timestamps and the
  /// delayed frames metrics. If the delayed frames array is empty then
  /// only the total frames will be calculated.
SpanFrameMetrics? getFrameMetrics({required DateTime spanStartTimestamp,required DateTime spanEndTimestamp}) { 
      //从_delayedFrames中过滤出在span的时间区间内的，表示span中有多少delayedFrames
  	 final relevantFrames = getFramesIntersecting(
        startTimestamp: spanStartTimestamp, endTimestamp: spanEndTimestamp);
  final spanDuration =
        spanEndTimestamp.difference(spanStartTimestamp).inMilliseconds;
  
  // No slow or frozen frames detected
    if (relevantFrames.isEmpty) {
      return SpanFrameMetrics(
          totalFrameCount:
              (spanDuration / _expectedFrameDuration.inMilliseconds).ceil(),
          slowFrameCount: 0,
          frozenFrameCount: 0,
          framesDelay: 0);
    }
  
   final spanStartMs = spanStartTimestamp.millisecondsSinceEpoch;
    final spanEndMs = spanEndTimestamp.millisecondsSinceEpoch;
    final expectedDurationMs = _expectedFrameDuration.inMilliseconds;
    final frozenThresholdMs = _frozenFrameThreshold.inMilliseconds;
  
   for (final timing in relevantFrames) {
      final frameStartMs = timing.startTimestamp.millisecondsSinceEpoch;
      final frameEndMs = timing.endTimestamp.millisecondsSinceEpoch;
      final frameDurationMs = timing.duration.inMilliseconds;
   
   		// Fully contained
      effectiveDuration = frameDurationMs;
      effectiveDelay = max(0, frameDurationMs - expectedDurationMs);
     
     // Classify frame
      final isFrozen = effectiveDuration >= frozenThresholdMs;
      final isSlow = effectiveDuration > expectedDurationMs;
     
      if (isFrozen) {
        frozenFrameCount++;
        frozenFramesDuration += effectiveDuration;
      } else if (isSlow) {
        slowFrameCount++;
        slowFramesDuration += effectiveDuration;
      }
   }
  
   return SpanFrameMetrics(
        totalFrameCount: totalFrameCount,
        slowFrameCount: slowFrameCount,
        frozenFrameCount: frozenFrameCount,
        framesDelay: framesDelay);
}
```



#### 总结

1. 在FramesTrackingIntegration中 注册系统帧回调给自定义的callback
2. callback中记录delayframe信息
3. 每一个span 的start和end触发 相关collect 将性能数据记录在span中
4. PerformanceCollector 就是负责将delayframe 过滤并记录在span中



