### RotationTransition

```dart
class _MyHomePageState extends State<MyHomePage> with SingleTickerProviderStateMixin{
  	void initState() {
  		_controller = AnimationController(
  		duration: Duration(seconds: 1),
    	vsync: this
  	);
  	super.initState();
	}
  
}


```

* `SingleTickerProviderStateMixin`适用于需要单个`Ticker`的动画场景。`Ticker`是一个时钟，每一帧都会调用一个回调函数，使动画得以进行。
* 当 `AnimationController` 初始化时，它会调用 传入的`SingleTickerProviderStateMixin` 的 `createTicker` 方法来创建一个 `Ticker`。

```dart
Center(
	child: ScaleTransition(
  	scale: _controller.drive(Tween(begin: 0.5, end: 2.0)),
    child: Container(
    	width: 300,
      height: 300,
      color: Colors.blue
    )
  )
),
floatingActionButton: FloatingActionButton(
	onPressed: (){
    _controller.forward();
  }
)
```

* _controller .drive可以驱动自定义补间动画的值
* 也可以通过``Tween(begin: 0.5, end: 2.0).animate(_controller)`` 达到一样的效果
* ````Tween(begin: 0.5, end: 2.0). chain(CurveTween(curve: Curve.elasticInOut)) ``  绑定多个补间动画

### 多个Tween串联 动画

``` dart
class SlidingBox extends StatelessWidget {
  final AnimationController controller;
  final Color color;
  final Interval interval;
  
  SlidingBox(this.controller,this.color,this.interval);
  
  @override
  Widget build(BuildContext context){
    return SlideTransition(
    	positon: Tween(begin:Offset.zero, end: Offset(0.1,0))
      				.chain(CurveTween(curve: interval))
      				.animate(controller),
      child: Container(
      	width: 300,
        height: 100,
        color: color,
      )
    )
  }
}


Center(
	child: Column(
  	children: <Widget>{
      SlidingBox(_controller, Colors.blue[100], Interval(0.0, 0.2)),
      SlidingBox(_controller, Colors.blue[300], Interval(0.2, 0.4)),
      SlidingBox(_controller, Colors.blue[500], Interval(0.6, 0.8)),
      SlidingBox(_controller, Colors.blue[700], Interval(0.8, 1.0)),
    }
  )
)

```

* 通过Interval来指定 每个控件开始的时间点，因为用的是同一个_controller，所以可以达到交错执行动画的效果

### 自定义AnimationBuilder动画

```dart
Center(
	child: AnimatedBuilder(
  	animation: _controller,
    builder: (BuildContext context, Widget child){
      	return Opacity(
        	opacity: Tween(begin: 0.5, end: 0.8).evaluate(_controller),
          child: Container(
          	width: 300,
            height: Tween(begin: 100.0, end: 200.0).evaluate(_controller),
            color: Colors.blue,
            child: child
          )
        )
    },
    child: Center(
    	child: Text("Hi", style: TextStyle(fontSize: 72))
    )
  )
)
```

* animation 传入的controller 默认是 0 - 1的补间值
* builder回调的child 可以用来做优化，比如Text控件不需要动画，可以在外层child属性赋值，这样在builder中就不用重新构建Text了
* ``Tween(begin: 100.0, end: 200.0).evaluate(_controller)` `evaluate表示 用指定的补间让_controller的计算
* 也可以用animate来计算： ``Animation opacityAnimation = Tween(begin: 0.5, end: 0.8).animate(_controller)``取opacityAnimation.value作为属性动画值即可











