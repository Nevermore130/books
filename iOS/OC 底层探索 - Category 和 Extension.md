### Category 的使用场合

* 给一个类添加新的方法，可以为系统的类扩展功能。

* 创建对私有方法的前向引用：声明私有方法，把 Framework 的私有方法公开等。直接调用其他类的私有方法时编译器会报错的，这时候可以创建一个该类的分类，在分类中声明这些私有方法（不必提供方法实现），接着导入这个分类的头文件就可以正常调用这些私有方法。
* 实例方法、类方法、协议、属性（只生成 setter 和 getter 方法的声明，不会生成 setter 和 getter 方法的实现以及下划线成员变量）；
* 默认情况下，由于分类底层结构的限制，不能添加成员变量到分类中，但可以通过关联对象来间接实现这种效果。

运行时决议：Category 编译之后的底层结构时`struct category_t`，里面存储着分类的对象方法、类方法、属性、协议信息，这时候分类中的数据还没有合并到类中，而是在程序运行的时候通过`Runtime`机制将所有分类数据合并到类（类对象、元类对象）中去。（这是分类最大的特点，也是分类和扩展的最大区别，**扩展是在编译的时候就将所有数据都合并到类中去了）**

### Category 的实现原理

* 分类的实现原理取决于运行时决议；
* 同名分类方法谁能生效取决于编译顺序，最后参与编译的分类中的同名方法会最终生效；

在编译时，Category 中的数据还没有合并到类中，而是在程序运行的时候通过`Runtime`机制将所有分类数据合并到类（类对象、元类对象）中去。下面我们来看一下 Category 的加载处理过程。

* 通过`Runtime`加载某个类的所有 Category 数据；
* 把所有的分类数据（方法、属性、协议），合并到一个大数组中；
  （后面参与编译的 Category 数据，会在数组的前面）
* 将合并后的分类数据（方法、属性、协议），插入到宿主类原来数据的前面。
  （所以会优先调用最后参与编译的分类中的同名方法）

### Extension 是什么？

* Extension 有一种说法叫“匿名分类”，因为它很像分类，但没有分类名。严格来说要叫类扩展。
* Extension 是在编译的时候就将所有数据都合并到类中去了（编译时决议），而 Category 是在程序运行的时候通过`Runtime`机制将所有数据合并到类中去（运行时决议）。

### Category 场景

* 可以把不同的功能组织到不同的Category里
* 可以由多个开发者共同完成一个类
* 可以按需加载想要的category
* 模拟多继承。

为Person创建一个名为MyCategory的Category后，会自动生成Person+MyCategory.h和Person+MyCategory.m文件。我们在MyCategory中声明和实现一个read方法

```objective-c
//  Person+MyCategory.h
#import "Person.h"

@interface Person (MyCategory)

-(void)read;

@end

```

```objective-c
//  Person+MyCategory.m
#import "Person+MyCategory.h"

@implementation Person (MyCategory)

-(void)read{
    NSLog(@"调用了MyCategory的read方法！");
}

@end

```

之后我们可以在ViewController或其他地方使用分类中添加的方法，如下：

```objective-c
//  ViewController.m
#import "ViewController.h"
#import "Person+MyCategory.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    Person *p = [[Person alloc] init];
    [p read];

}

@end

```

分类可以重新实现原来类中的方法，但是会覆盖掉原来的方法，会导致原来的方法没法再使用（实际上并没有真的替换，而是Category的方法被放到了新方法列表的前面 (类比android classloader机制)

Xcode是不允许我们在Category中添加成员变量的。

Objective-C类是由Class类型来表示的，它实际上是一个指向objc_class结构体的指针。它的定义如下

```objective-c
typedef struct objc_class *Class;

//objc_class结构体的定义如下：
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类
    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
    long instance_size                      OBJC2_UNAVAILABLE;  // 该类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 该类的成员变量链表
    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存
    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
#endif
} OBJC2_UNAVAILABLE;

```

* 在上面的objc_class结构体中，ivars是objc_ivar_list（成员变量列表）指针
* methodLists是指向objc_method_list指针的指针
* 在Runtime中，objc_class结构体大小是固定的，不可能往这个结构体中添加数据，只能修改。所以ivars指向的是一个固定区域，只能修改成员变量值，不能增加成员变量个数。
* methodList是一个二维数组，所以可以修改*methodLists的值来增加成员方法，虽没办法扩展methodLists指向的内存区域，却可以改变这个内存区域的值（存储的是指针）。因此，可以动态添加方法，不能添加成员变量。

























