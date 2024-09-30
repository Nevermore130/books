该 log 包是基于 go.uber.org/zap 包封装而来的，根据需要添加了更丰富的功能。接下来，我们通过 log 包的Options，来看下 log 包所实现的功能：

```go
type Options struct {
    OutputPaths       []string `json:"output-paths"       mapstructure:"output-paths"`
    ErrorOutputPaths  []string `json:"error-output-paths" mapstructure:"error-output-paths"`
    Level             string   `json:"level"              mapstructure:"level"`
    Format            string   `json:"format"             mapstructure:"format"`
    DisableCaller     bool     `json:"disable-caller"     mapstructure:"disable-caller"`
    DisableStacktrace bool     `json:"disable-stacktrace" mapstructure:"disable-stacktrace"`
    EnableColor       bool     `json:"enable-color"       mapstructure:"enable-color"`
    Development       bool     `json:"development"        mapstructure:"development"`
    Name              string   `json:"name"               mapstructure:"name"`
}
```

* disable-caller：是否开启 caller，如果开启会在日志中显示调用日志所在的文件、函数和行号。

log 包实现了以下 3 种日志记录方法：

```go
log.Info("This is a info message", log.Int32("int_key", 10))
log.Infof("This is a formatted %s message", "info")
log.Infow("Message printed with Infow", "X-Request-ID", "fbf54504-64da-4088-9b86-67824a7fb508")

//output
2021-07-06 14:02:07.070 INFO This is a info message {"int_key": 10}
2021-07-06 14:02:07.071 INFO This is a formatted info message
2021-07-06 14:02:07.071 INFO Message printed with Infow {"X-Request-ID": "fbf54504-64da-4088-9b86-67824a7fb508"}
```

Infow 也是使用指定的 key/value 记录日志，跟 Info 的区别是：使用 Info 需要指定值的类型，通过指定值的日志类型，日志库底层不需要进行反射操作，所以使用 Info 记录日志性能最高

**log 包支持 WithValues 函数**，例如

```go
// WithValues使用
lv := log.WithValues("X-Request-ID", "7a7b9f24-4cae-4b2a-9464-69088b45b904")
lv.Infow("Info message printed with [WithValues] logger")
lv.Infow("Debug message printed with [WithValues] logger")

//output
2021-07-06 14:15:28.555 INFO Info message printed with [WithValues] logger {"X-Request-ID": "7a7b9f24-4cae-4b2a-9464-69088b45b904"}
2021-07-06 14:15:28.556 INFO Debug message printed with [WithValues] logger {"X-Request-ID": "7a7b9f24-4cae-4b2a-9464-69088b45b904"}
```

WithValues 可以返回一个携带指定 key-value 的 Logger，供后面使用

**log 包提供 WithContext 和 FromContext** 

用来将指定的 Logger 添加到某个 Context 中，以及从某个 Context 中获取 Logger

```go
// Context使用
ctx := lv.WithContext(context.Background())
lc := log.FromContext(ctx)
lc.Info("Message printed with [WithContext] logger")
```

WithContext和FromContext非常适合用在以context.Context传递的函数中，例如：

可以很方便地从 Context 中提取出指定的 key-value，作为上下文添加到日志输出中，例如

**通过调用 Log.L() 函数**，实现如下：

```go
// L method output with specified context value.
func L(ctx context.Context) *zapLogger {
    return std.L(ctx)
}
 
func (l *zapLogger) L(ctx context.Context) *zapLogger {
    lg := l.clone()
 
    requestID, _ := ctx.Value(KeyRequestID).(string)
    username, _ := ctx.Value(KeyUsername).(string)
    lg.zapLogger = lg.zapLogger.With(zap.String(KeyRequestID, requestID), zap.String(KeyUsername, username))
 
    return lg
}
```

* L() 方法会从传入的 Context 中提取出 requestID 和 username ，追加到 Logger 中，并返回 Logger
* 这时候调用该 Logger 的 Info、Infof、Infow 等方法记录日志，输出的日志中均包含 requestID 和 username 字段

**初始化log**

```go
// Init initializes logger with specified options.
func Init(opts *Options) {
	mu.Lock()
	defer mu.Unlock()
	std = New(opts)
}

// New create logger by opts which can custmoized by command arguments.
func New(opts *Options) *zapLogger {
	if opts == nil {
		opts = NewOptions()
	}

	var zapLevel zapcore.Level
	if err := zapLevel.UnmarshalText([]byte(opts.Level)); err != nil {
		zapLevel = zapcore.InfoLevel
	}
	encodeLevel := zapcore.CapitalLevelEncoder
	// when output to local path, with color is forbidden
	if opts.Format == consoleFormat && opts.EnableColor {
		encodeLevel = zapcore.CapitalColorLevelEncoder
	}

	encoderConfig := zapcore.EncoderConfig{
		MessageKey:     "message",
		LevelKey:       "level",
		TimeKey:        "timestamp",
		NameKey:        "logger",
		CallerKey:      "caller",
		StacktraceKey:  "stacktrace",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    encodeLevel,
		EncodeTime:     timeEncoder,
		EncodeDuration: milliSecondsDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}

	loggerConfig := &zap.Config{
		Level:             zap.NewAtomicLevelAt(zapLevel),
		Development:       opts.Development,
		DisableCaller:     opts.DisableCaller,
		DisableStacktrace: opts.DisableStacktrace,
		Sampling: &zap.SamplingConfig{
			Initial:    100,
			Thereafter: 100,
		},
		Encoding:         opts.Format,
		EncoderConfig:    encoderConfig,
		OutputPaths:      opts.OutputPaths,
		ErrorOutputPaths: opts.ErrorOutputPaths,
	}

	var err error
	l, err := loggerConfig.Build(zap.AddStacktrace(zapcore.PanicLevel), zap.AddCallerSkip(1))
	if err != nil {
		panic(err)
	}
	logger := &zapLogger{
		zapLogger: l.Named(opts.Name),
		infoLogger: infoLogger{
			log:   l,
			level: zap.InfoLevel,
		},
	}
	klog.InitLogger(l)
	zap.RedirectStdLog(l)

	return logger
}
```

zapLogger

```go
// zapLogger is a logr.Logger that uses Zap to log.
type zapLogger struct {
	// NB: this looks very similar to zap.SugaredLogger, but
	// deals with our desire to have multiple verbosity levels.
	zapLogger *zap.Logger
	infoLogger
}

// infoLogger is a logr.InfoLogger that uses Zap to log at a particular
// level.  The level has already been converted to a Zap level, which
// is to say that `logrLevel = -1*zapLevel`.
type infoLogger struct {
	level zapcore.Level
	log   *zap.Logger
}
```

几个核心方法

```go
// StdInfoLogger returns logger of standard library which writes to supplied zap
// logger at info level.
func StdInfoLogger() *log.Logger {
	if std == nil {
		return nil
	}
	if l, err := zap.NewStdLogAt(std.zapLogger, zapcore.InfoLevel); err == nil {
		return l
	}

	return nil
}

func (l *zapLogger) Write(p []byte) (n int, err error) {
	l.zapLogger.Info(string(p))

	return len(p), nil
}

```











