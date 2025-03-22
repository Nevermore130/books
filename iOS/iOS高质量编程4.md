### 通过委托与数据源协议进⾏对象间通信

利用协议机制，实现

```objective-c
MyView.h

//委托方声明协议
@protocol MyViewDelegate <NSObject>

//可选实现方法
@optional 
-(void)optionalFunc;

//必须实现方法
@required 
-(void)requiredFunc;

@end
  
 MyView.h
//委托方声明代理属性 注意要用weak修饰
@interface MyView : UIView
@property(nonatomic,weak)id<MyViewDelegate> delegate;
@end

```

```objective-c
# MyViewController.m

//代理方遵守协议
@interface MyViewController () <MyViewDelegate>
	@property(nonatomic,strong)MyView * myView;
@end
  
  
 # MyViewController.m
@implementation MyViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    [self.view addSubview:self.myView];
}

-(MyView *)myView{
    if (!_myView) {
        _myView = [[MyView alloc] initWithFrame:self.view.bounds];
        _myView.backgroundColor = [UIColor whiteColor];
        //持有代理
        _myView.delegate = self;
    }
    return _myView;
}

//实现协议方法
- (void)requiredFunc{
    NSLog(@"requiredFunc");
}

@end
```

```objective-c
MyView.m

//调用代理 调用前判断是否有方法实现
if ([self.delegate respondsToSelector:@selector(requiredFunc)]) {
    [self.delegate requiredFunc];
}

```

### Objective C中Alloc和AllocWithZone

###### 单例的实现代码

```	objective-c
#import "Singleton.h"

@implementation Singleton
static Singleton* _instance = nil;
+(instancetype) shareInstance
{
    static dispatch_once_t onceToken ;
    dispatch_once(&onceToken, ^{
    _instance = [[super allocWithZone:NULL] init] ;
    //为什么不用下面这句呢？
    //为什么要覆盖allocWithZone方法，到底alloc 和 allocWithZone有什么区别呢？
    //_instance = [[super alloc] init] ;
}) ;
return _instance ;
}

+(id) allocWithZone:(struct _NSZone *)zone
{
    return [Singleton shareInstance] ;
}

-(id) copyWithZone:(struct _NSZone *)zone
{
    return [Singleton shareInstance] ;
}
@end

```

* 初始化一个对象的时候，[[Class alloc] init]，其实是做了两件事。 alloc 给对象分配内存空间，init是对对象的初始化，包括设置成员变量初值这些工作。 而给对象分配空间
* 除了alloc方法之外，还有另一个方法： allocWithZone. 使用alloc方法初始化一个类的实例的时候，默认是调用了allocWithZone的方法。为了保持单例类实例的唯一性，需要覆盖所有会生成新的实例的方法，如果初始化这个单例类的时候不走[[Class alloc] init] ，而是直接 allocWithZone， 那么这个单例就不再是单例了，所以必须把这个方法也堵上。

### 实现 NSCopying 的步骤

* 声明 NSCopying 协议:在头文件中声明类遵守 NSCopying`协议。
* 实现 copyWithZone 方法：在实现文件中编写 copyWithZone方法，创建并返回对象的副本

```objective-c
@interface MyObject : NSObject<NSCopying>
 
@property (nonatomic,strong) NSString * name;
@property (nonatomic,assign) NSUInteger age;
 
@end
 
@implementation MyObject
 
 
- (id)copyWithZone:(NSZone *)zone {
    MyObject *copy = [[[self class] allocWithZone:zone] init];
    if (copy) {
        copy->_name = [_name copyWithZone:zone];
        copy->_age = _age;
    }
    return copy;
}
 
@end
```

```objective-c
    MyObject *original = [[MyObject alloc] init];
    original.name = @"Original";
    original.age = 30;
 
    MyObject *copy = [original copy];

```

**在`- (id)copyWithZone:(NSZone *)zone`方法中，一定要通过`[self class]`方法返回的对象调用`allocWithZone:`方法。因为指针可能实际指向的是MyObject的子类。**这种情况下，通过调用`[self class]`，就可以返回正确的类的类型对象。







