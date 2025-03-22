### 在类的头文件中尽量少引入其他头文件

```objective-c
// EocPerson.h
#import <Foundation/Foundation.h>
@interface EocPerson :NSObject
 @property (nonatomic,copy) Nsstring *firstName;
 @property (nonatomic, copy)Nsstring *lastName;
@end
  
// EocPerson.m
#import "EocPerson.h"
@implementation EOCPerson
// Implementation of methods
@end
```

@class通常可以使你不用过早地import。如果编译器看到了一行语法：

@class myCoolClass，它就知道了自己可能马上会看到类似如下的代码

```objc
myCoolClass *myObject;
```

于是它会为这个类保留一个指针的空间，然后忙其他的去了。

不过如果你需要创建或访问myObject的成员，那么仅仅一个类指针就不够了。你需要让编译器知道这些成员到底是什么。这时候就需要``#import "myCoolClass.h"``了。

当需要知道有一个类的存在时，而不关心其内部细节时，可以使用向前声明（Forward Declaring）告知编译器，即可以在 .h（头文件）中 `@class SomeClass`；而在 .m（实现文件）中引入实际的 .h；当在两个头文件中互相引入对方，则会导致「循环 引用（Chicken-Egg Situation，又称交叉引用）」，无法通过编译：

**将引入头文件的时机尽量延后，只有在确定有需要时才引入，否则会增加耦合度、拉长编译时间、产生相互依赖等问题。**

继承父类和遵循协议则不能使用向前声明，必须引入相应的头文件，因此协议最好声明在单独的头文件中

### 理解属性概念

实例变量一般通过“存取方法”（`access method`）来访问。其中“获取方法”（`getter`）用于读取变量值，而“设置方法”（`setter`）用于写入变量值。**开发者可以令编译器自动编写与属性相关的存取方法。也可以使用“点语法”（`dot syntax`）更为容易地依照类对象来访问存取其中的数据**

###### 原子性 Atomicity

`atomic` 表示这个属性的getter/setter方法是原子性的，即在多线程下访问该属性对应的getter/setter方法是原子性的，`nonatomic` 表示这个属性的getter/setter方法是非原子性的

```objective-c
@interface FooClass : NSObject

@property (atomic) id fooA;
@property (nonatomic) id fooN;

@end

//做一次等价转化：
@interface FooClass : NSObject {
	@private id _fooA;
	@private id _fooN;
}

- (id)fooA;
- (void)setFooA:(id)fooA;

- (id)fooN;
- (void)setFooN:(id)fooN;

@end

@implementation FooClass

- (id)fooA {
    @synchronized (self) {
        return _fooA;
    }
}
- (void)setFooA:(id)fooA {
    @synchronized (self) {
        return _fooA = fooA;
    }
}

- (id)fooN {
    return _fooN;
}
- (void)setFooN:(id)fooN {
    _fooN = fooN;
}

@end

```

**`atomic`属性并不是线程安全的，或者说它只保证了getter/setter方法的“线程安全”**

```objective-c
@interface FooClass : NSObject

@property (atomic) NSMutableArray *array;

@end


// foo thread
FooClass *foo = [FooClass new];
foo.array = [NSMutableArray array];

// a thread
[foo.array addObject:@"a"];

// b thread
[foo.array addObject:@"b"];

// multi-thread
[foo.array addObject:@"..."];

```

我们在多线程场景中，往`foo`对象的`array`添加对象，此时大概率会发生crash。虽然，`array`属性是`atomic`的，而且在上面这个例子中，我们在调用`array`属性的getter方法时，也保证了原子性，但是我们调用添加对象的方法`addObject:`时，`addObject:`方法并不是线程安全的，所以只要同时有两条线程访问了`addObject:`方法，就大概率会发生crash。**在我们的工程实践中，我们往往是想确保一系列的方法调用是线程安全的，而不是仅仅确保某个getter和setter方法是线程安全的**。

###### 读写性 Writability

编译器想实现属性是`readonly`的，它只需要生成getter方法而不生成setter方法即可：

##### 存储语义 Setter Semantics

- `strong` 强引用
- `weak` 弱引用
- `assign` 仅赋值
- `copy` 拷贝

如果用assign修饰对象，当对象释放后（因为不存在强引用，离开作用域对象内存可能被回收），指针的地址还是存在的，也就是说指针并没有被置为nil,下次再访问该对象就会造成野指针异常。

但weak会在对象释放的时候，把指针置空，这样就无法访问到被释放的内存了。

**`copy`特性的setter方法是调用入参对象的`copy`方法**。



