### 在对象内部尽量直接访问实例变量

```objective-c
@interface EocPerson :Nsobject
  @property(nonatomic,copy)NsString *firstName;
  @property(nonatomic,copy)Nsstring *lastName;
// Convenience for firstName +""+ lastName:
(NsString*)fullName;
(void)setFullName:(Nsstring*)fullName;
@end
```

fullName与setFullName 这两个便捷方法可以这样实现

```objective-c
(Nsstring*)fullName {
	return [Nsstring stringWithFormat:@"号@ 号@"self.firstName, self,lastName];
}
/** The following assumes all full names have exactly 2parts, The method could be rewritten to support moreexotic names. **/
(void)setFullName:(Nsstring*)fullName {
  NSArray *components =[fullName componentsseparatedBystring:@" "];
  self,firstName =[components objectAtIndex:0];
  self.lastName =[components objectAtIndex:1];
}
```

假设重写这两个方法，不经由存取方法，而是直接访问实例变量

```objective-c
(Nsstring*)fullName {
  return [NSString stringWithFormat:@"%@ %@" _firstName, _lastName];
}

(void) setFullName:(Nsstring*)fullName {
  NSArray *components = [fullName componentsseparatedBystring:@" "];
  _firstName =[components objectAtIndex:0];
  _lastName =[components objectAtIndex:1];
}
```

两种写法有几个区别

* 由于不经过OC的方法派发，直接访问实例变量的速度当然比较快，编译器生成的代码会直接访问保存对象实例变量的那块内存
* 直接访问实例变量时，不会调用“设置方法”，绕过了为相关属性所定义的“内存管理语义”，比如ARC下直接访问一个声明为copy的属性，那么并不会拷贝该属性
* 直接访问实例变量不会触发KVO
* 通过属性访问有助于排查错误，可以给“获取方法”和“设置方法”中新增断点。监控该属性的调用者及其访问时机

折中方案：写入实例变量通过“设置方法”来做，读取实例变量，直接访问之，既能提高读取操作速度，又能控制对属性的写入操作，通过“设置方法”写入变量能确保相关**属性的“内存管理语义”**得以贯彻

### 使用关联对象存放自定义数据

可以给某对象关联许多其他对象，这些对象通过"key"区分，存储对象值的时候，可以指明“存储策略”用以维护相应的"内存管理语义"。存储策略由名为obj_AssociationPolicy的枚举定义

* ``void objc_setAssociatedObject(id object, void*key, id value,obj_AssociationPolicy policy)`` 此方法以给定的键和策略为某对象设置关联对象值
* ``id objc_getAssociatedObject(id object,void*key)`` 此方法根据给定的键从某对象中获取相应的关联对象值

##### 使用场景

由于分类底层结构的限制(运行时合并到类里)，不能直接给 Category 添加成员变量，但是可以通过关联对象间接实现 Category 有成员变量的效果。

##### 关联对象用法举例

当用户按下按钮关闭该视图时，需要用委托协议(delegate protocol)来处理此动作，但是，要想设置好这个委托机制，就得把创建警告视图和处理按钮动作的代码分开。由于代码分作两块，所以读起来有点乱。比方说，我们在使用UIAlertView时，一般都会这么写:

```objective-c
(void)askUserAouestion{
UIAlertView *alert=[[UIAlertview alloc]
							initWithTitle:@"Question"
							message:@"What do you want to do?"
                    delegate:self
							cancelButtonTitle:@"Cancel"
							otherButtonTitles:@"Continue", nil];
		[alert show];
}
// UIAlertviewDelegate protocol method
-(void)alertView:(UIAlertView *)alertView
							clickedButtonAtIndex:(NsInteger)buttonIndex
if(buttonIndex ==0){
    [self docancel];
} else {
		[self doContinue];
}
```

这可以通过关联对象来做。创建完警告视图之后，设定一个与之关联的“块”``(block)``，等到执行 delegate 方法时再将其读出来。此方案的实现代码如下

```objective-c
#import <objc/runtime.h>
static void *EocMyAlertViewKey ="EocMyAlertviewkey";
(void)askUserAOuestion {
UIAlertView *alert=[[UIAlertviewalloc]
                    initWithTitle:@"Question"
                    message:@"What do you do?" 
                    delegate:self 
                    cancelButtonTitle:@"Cancel"
                    otherButtonTitles:@"Continue"， nil];
  
void (^block)(NSInteger) = ^(NSInteger buttonIndex){
  if(buttonIndex ==0){
    [self doCancel];
  }else{
    [self doContinue];
  };
  
  objc_setAssociatedObject(alert, EocMyAlertViewKey,block,BJC_ASSOCIATION_COPY);
  
  [alert show];


}
  
//UIAlertViewDelegate protocol method
 - (void) alertView:(UIAlertView*)alertView
   			clickedButtonAtIndex:(NSInteger)buttonIndex
        {
          void (^block)(NSInteger) = objc_getAssociatedObject(alertView,EocMyAlertViewKey);
          block(buttonIndex);
        }
                         
                         
```







