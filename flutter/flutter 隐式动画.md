### AnimatedSwitcher

```dart
Center(
	child: AnimatedContainer(
  	duration: Duration(seconds: 1),
    width: 300,
    height: 300,
    color: Colors.orange,
    child: AnimatedSwitcher(
    	transitionBuilder: (child, animation){
        return FadeTransition(
        	opacity: animation,
          child: child
        )
      },
      duraion: Duration(seconds: 2),
      child: Text("Good",key:UniqueKey(),style:TextStyle(fontSize: 100))
    )
  )
)
```

* AnimatedSwitcher 用于控件变化的时候做动画
* 控件变化的条件：key不同，或者控件类型不同
* 如定义自定义动画效果，可通过transitionBuilder 定义

### TweenAnimationBuilder

```dart
Center(
	child: TweenAnimationBuilder(
  	duration: Duration(seconds: 1),
    tween: Tween(begin: 72.0, end: 172.0),
    builder: (BuildContext context, value, Widget child){
      return Container(
      	width: 300,
        height: 300,
        color: Colors.blue,
        child: Center(
        	child: Text("Hi",style: TextStyle(fontSize: value))
        );
      )
    },
  );
)
```

* tween 设置的补间的值，从begin到end 会通过 builder中的value 回调
* 通过value来设置需要动画变化的属性值，比如fontSize













```dart
class AnimatedCounter extends StatelessWidget {
  final Duration duration;

  final int value;

  const AnimatedCounter(this.duration, this.value);

  @override
  Widget build(BuildContext context) {
    return TweenAnimationBuilder(
      tween: Tween(end: value.toDouble()),
      duration: duration,
      builder: (BuildContext context, value, Widget? child) {
        final whole = value ~/ 1; //取整
        final decimal = value - whole; //取小数部分
        return Stack(
          children: [
            Positioned(
              top: -100 * decimal, //0 -> -100    //从顶部消失
              child: Opacity(
                opacity: 1.0 - decimal, // 1.0 -> 0.0
                child: Text(
                  "$whole",
                  style: const TextStyle(fontSize: 100),
                ),
              ),
            ),
            Positioned(
                top: 100 - decimal * 100, //100->0   //从底部出来
                child: Opacity(
                  opacity: decimal, //0 - 1.0
                  child: Text(
                    "${whole + 1}",
                    style: const TextStyle(fontSize: 100),
                  ),
                ))
          ],
        );
      },
    );
  }
}
```

* TweenAnimationBuilder  builder属性用于返回一个child view, 可以理解为动画刷新的view
* builder 回调的value 是基于duration 和 tween 设置的end value 来 告诉builder 当前动画刷新的“进度”
* 基于回调的value来刷新child需要更新的动画值，比如透明度opacity