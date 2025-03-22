### Flutter中控制器的使用与重要性。

通过实例演示了如何使用控制器从外部控制一个小组件内部的状态，避免了直接将**状态提升至外部的缺**点。接着，介绍了使用复杂类型(如类)来传递状态，以避免基础类型的复制问题。然后通过定义一个名为`doubleHolder的类，实现了对`double值的控制，避免了直接使用`double`带来的复杂性。最后创建了一个名为`FullController`的类，解决了数据传递、刷新通知和组件控制权的问题，展示了控制器在实际应用中的重要性和实用性。



```dart
class _FooState extends State<Foo>{
	Widget build(BuildContext context){
    return Container(
    	color: Colors.red.withOpacity(2),
      child: AnimatedBuilder(
      	animation: widget.controller,
        builder: (BuildContext context,Widget child){
          return Column(
          	children:[
              FlutterLogo(size: widget.controller.value * 100 + 50),
              Slider(
              	value: widget.controller.value,
                onChanged: (double value){
                  setState(){
                    widget.dh.value = value;
                  }
                }
              )
            ]
          ); //Column
       
        },// AnimatedBuilder
      )
    );
  }
}
```

FooController

```dart
class FooController extends ChangeNotifier{
  double value = 0.5
   
  double get value => _value;
    
  set value(double newValue){
    _value = newValue;
    notifyListeners();
  }
  
  void setMax(){
    _value = 1.0;
    notifyListeners();
  }
    
}
```

如何调用

```dart
Column(
	childrens:[
    Foo(controller: _controller),
    ElevatedButton(
    	child: Text("set to 100%"),
      onPressed: (){
        _controller.setMax();
      }
    )
  ]
)
```

### 继承式组件InheritedWidget

实现一个改变Color状态的InheritedWidget

```dart
class MyColor extends InheritedWidget{
  final Color color;
  
  const MyColor({super.key, required super.child, required this.color});
  
  static MyColor? maybeOf(BuildContext context){
    return context.dependOnInheritedWidgetOfExactType<MyColor>();
  }
 
  bool updateShouldNotify(covariant MyColor oldWidget){
    return oldWidget.color != color;
  }
}
```

当updateShouldNotify 发生改变的时候也会调用组件的didChangeDependencies

```dart
class _FooState extends State<Foo>{
  @override
  void didChangeDependencies(){
    super.didChangeDependencies()
  }
  
  Widget build(BuildContext context){
    return Container(
    	width: 100,
      height: 100,
      color: MyColor.maybeOf(context).color
    )
  }
}
```

如何使用MyColor

```dart
class MyApp extends StatelessWidget{
  return MyColor(
  	color: Colors.red,
    child: MaterialApp(
    	home: MyHomePage()
    )
  )
}

...
```

通过包裹MyColor继承式组件，在使用MyColor组件的地方会通过dependOnInheritedWidgetOfExactType 来找到距离最近的继承式组件，拿到MyColor里的属性

### 实现一个Provider

```dart
class MyInheritedWidget extends InheritedWidget {
  
  late LogoModel model;
  
  
  const LogoModelProvider({super.key,
                           required this.model 
                           required super.child})
  
  static MyInheritedWidget of(BuildContext context){
    return context.dependOnInheritedWidgetOfExactType<MyInheritedWidget>();
  }
    
    
  @override
  bool updateShouldNotify(covariant MyColor oldWidget){
    return true;
  }
  
  
}

```



```dart
class MyProvider extends StatefulWidget{
  final Widget child;
  final LogoModel Function() create;
  
  const MyProvider({super.key,required this.create,required this.child});
  
  @override
  State<MyProvider> createState() => _MyProviderState();
}

class _MyProviderState extends State<MyProvider> {
  late LogoModel model;
  
  @override
  void initState(){
    super.initState();
    model = widget.create();
  }
  
  @override
	Widget build(BuildContext context){
    return ListenableBuilder(
    	listenable: model,
      builder: (BuildContext context, Widget? child){
        return MyInheritedWidget(
        	model: model,
          child: widget.child
        );
      }
    )
  }
}
```

* MyProvider 传入外层的Widget ，并传入一个可创建model的create回调
* _MyProviderState 初始化的时候会调用create 生成一个model
* model 是一个ChangeNotify对象，所以当model的属性发生改变的时候会调用notifyListeners()
* _MyProviderState 返回一个ListenableBuilder 对象来监听ChangeNotify的变化
* 当mode变化时会触发builder 构造一个新的MyInheritedWidget对象
* MyInheritedWidget对象中的 updateShouldNotify 返回true表示dependOnInheritedWidgetOfExactType的地方会收到通知
* 所以当调用MyInheritedWidget.of(context).model的地方 会触发更新

```dart
class Logo extends StateLessWidget {
  const Logo({Key? key}) : super(key:key);
  
  Widget build(BuildContext context){
    final model = MyInheritedWidget.of(context).model;
    
    return Card(
    	child: Transform.flip(
      	flipX: model.flipX,
        flipY: model.flipY,
        child: FlutterLogo(size: model.size)
      )
    );
  }
}
```











