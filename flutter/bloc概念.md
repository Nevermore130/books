### Cubit

`Cubit` 是扩展自 `BlocBase` 的类并可以扩展用于管理任何类型的状态。

<img src="https://bloclibrary.dev/_astro/cubit_architecture_full.CT5Fr9vK_17GTki.webp" alt="Cubit 架构" style="zoom:30%;" />

状态是 `Cubit` 的输出，代表应用程序状态的一部分。UI 组件可以收到状态通知，并根据当前状态进行部分重绘。

可以像这样创建一个 `CounterCubit` ：

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
}
```

我们需要定义 `Cubit` 管理的状态类型。以上面的 `CounterCubit` 为例，状态类型是 `int` ，**但在更复杂的情况下，可能需要使用 `class` 而不是值类型。**

#### Cubit 状态变更

每一个 `Cubit` 都可以通过 `emit` 输出一个新的状态。

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);


  void increment() => emit(state + 1);
}
//emit 是保护方法，意味着它只能在 Cubit 内部被调用
```

##### 基本用法

```dart
//Cubit 公开了一个 Stream 可以用于接收实时的状态更新：
Future<void> main() async {
  final cubit = CounterCubit();
  final subscription = cubit.stream.listen(print); // 1
  cubit.increment();
  //await Future.delayed(Duration.zero) 在这个示例里是为了避免 subscription 被立即取消。
  await Future.delayed(Duration.zero);
  await subscription.cancel();
  await cubit.close();
}
```

当 `Cubit` 发出一个新的状态时，一个 `Change` 发生了。我们可以通过重写 **`onChange` 来观察 `Cubit` 的所有变化**。

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);

  @override
  void onChange(Change<int> change) {
    super.onChange(change);
    print(change);
  }
}
```

#### BlocObserver

使用 bloc 库的一个额外好处是我们可以在一个位置访问所有的 `Changes`

如果我们想对所有的 `Changes` 做出一些反应，仅需创建我们自己的 `BlocObserver` 即可

```dart
class SimpleBlocObserver extends BlocObserver {
  @override
  void onChange(BlocBase bloc, Change change) {
    super.onChange(bloc, change);
    print('${bloc.runtimeType} $change');
  }
}
```

### Bloc

`Blocs` 不是调用 `Bloc` 上的 `函数` 并直接发出新的 `状态`，而是接收 `事件` 并将传入的 `事件` 转换为传出的 `状态`。

创建 `Bloc` 跟创建 `Cubit` 类似，不过除了定义我们要管理的状态以外，我们还必须定义 `Bloc` 能够处理的事件。

事件是 Bloc 的输入。通常情况下这些事件用于响应用户的交互，类似按下按钮或者页面加载的生命周期事件等等。

```dart
sealed class CounterEvent {}


final class CounterIncrementPressed extends CounterEvent {}


class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0);
}
```

跟 `Cubit` 里的函数相反，`Bloc` 必须通过 `on<Event>` API注册事件处理程序。事件处理程序负责将任何传入事件转换为零个或多个传出状态。

```dart
sealed class CounterEvent {}


final class CounterIncrementPressed extends CounterEvent {}


class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0) {
    on<CounterIncrementPressed>((event, emit) {
      // handle incoming `CounterIncrementPressed` event
      emit(state + 1);
    });
  }
}
//blocs 和 cubits 都会忽略重复的状态。如果 state == nextState 时我们发出 State nextState ，不会有状态变更发生。
```

### 使用 Bloc

```dart
Future<void> main() async {
  final bloc = CounterBloc();
  print(bloc.state); // 0
  bloc.add(CounterIncrementPressed());
  await Future.delayed(Duration.zero);
  print(bloc.state); // 1
  await bloc.close();
}
```

和 `Cubit` 一样，`Bloc` 是一个特殊的 `Stream` 类型，这意味着我们也可以订阅 `Bloc` 以实时更新其状态：

```dart
Future<void> main() async {
  final bloc = CounterBloc();
  final subscription = bloc.stream.listen(print); // 1
  bloc.add(CounterIncrementPressed());
  await Future.delayed(Duration.zero);
  await subscription.cancel();
  await bloc.close();
}
```

### Flutter Bloc 核心概念

**BlocBuilder** 是一个 Flutter 的 Widget，它需要一个 `Bloc` 和一个 `builder` 函数。`BlocBuilder` 处理于构建响应新状态时构建的 Widget 

**`StreamBuilder`具有更简单的 API，以减少所需的样板代码量。`builder` 函数可能会被多次调用，并且必须是一个根据状态返回一个widget的[纯函数](https://en.wikipedia.org/wiki/Pure_function)。**

如果你希望在状态变化时执行一些操作，比如导航、显示对话框等…，请查看 `BlocListener`

在BlocBuilder中，如果省略了 `bloc` 参数，则 `BlocBuilder` 会自动通过 `BlocProvider` 和当前的 `BuildContext` 进行查找

只有在希望提供一个仅作用于单个 widget 且无法透过父级 `BlocProvider` 和当前的 `BuildContext` 访问的 `bloc` 时，才需要明确指定 `bloc`。

```dart
BlocBuilder<BlocA, BlocAState>(
  bloc: blocA, // provide the local bloc instance
  builder: (context, state) {
    // return widget here based on BlocA's state
  },
);
```

如果要达到更细緻地控制 widget 的更新，可以使用 `buildWhen` 参数来实现何时调用 `builder` 函数。`buildWhen` 接受前一个 bloc 状态和当前 bloc 状态，并返回一个布尔值。如果 `buildWhen` 返回 true，则 `builder` 函数将被调用并使用当前 `state` 进行widget重建。如果 buildWhen 返回 false，则 `builder` 将不会被使用当前 `state` 调用，且不会进行重建。

```dart
BlocBuilder<BlocA, BlocAState>(
  buildWhen: (previousState, state) {
    // return true/false to determine whether or not
    // to rebuild the widget with state
  },
  builder: (context, state) {
    // return widget here based on BlocA's state
  },
);
```

#### BlocProvider

`BlocProvider` 用作于依赖注入（DI）widget ，以便在子树 (subtree) 中可以提供单个 bloc 实例给多个widget 使用。

通常情况下，应该使用 `BlocProvider` 来创建新的 bloc，这样可以使该 bloc 对于子树中的其他 widget 使用。在这种情况下，由于 **`BlocProvider` 负责创建 bloc，它将自动处理关闭该 bloc**

```dart
BlocProvider(
  create: (BuildContext context) => BlocA(),
  child: ChildA(),
);
```

#### BlocListener

**BlocListener** 是一个 Flutter Widget，它接受一个 `BlocWidgetListener` 和一个可选的 `Bloc`，并在 bloc 状态发生变化时调用 `listener`。它应该用于需要每个状态变化仅执行一次的功能，例如导航、显示 `SnackBar`、显示 `Dialog` 等等。

与 `BlocBuilder` 中的 `builder` 不同的是，**`listener` 对于每个状态变化（初始状态除外）只会被调用一次，并且是一个 `void` 函数。**

```dart
BlocListener<BlocA, BlocAState>(
  listenWhen: (previousState, state) {
    // return true/false to determine whether or not
    // to call listener with state
  },
  listener: (context, state) {
    // do stuff here based on BlocA's state
  },
  child: const SizedBox(),
);
```

























