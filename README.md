# 面试宝典

## 介绍
我发现，平时代码编译或服务器重启的零碎时间都没有好好被利用，所以尝试利用这写临散时间，解答网上常见的一些面试题。

### 1. @property 后面可以有哪些修饰符？
atomic, nonatomic, readonly, readwrite, assign, strong, weak, copy, assign, retain, copy, setter=, getter=

@property默认修饰符是atomic, readwrite, assign或strong

### 2. 什么情况使用 weak 关键字，相比 assign 有什么不同？
什么情况使用 weak 关键字？

1. 在 ARC 中,在有可能出现循环引用的时候,往往要通过让其中一端使用 weak 来解决,比如:  delegate 代理属性或 NSTimer 属性

2. 自身已经对它进行一次强引用,没有必要再强引用一次,此时也会使用 weak ,自定义 IBOutlet 控件属性一般也使用 weak 。

不同点：

1. weak 此特质表明该属性定义了一种“非拥有关系”(nonowningrelationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同 assign 类似，然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。 而 assign 的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简单赋值操作。

2. assign 修饰非 OC 对象, weak 修饰 OC 对象。

### 3. 怎么用 copy 关键字？
属性NSString、NSArray、NSDictionary经常使用copy关键字，block 也经常使用 copy 关键字。

1. NSString、NSArray、NSDictionary使用copy关键字是为了防止给属性赋值的对象类型为NSMutableString、NSMutableArray、NSMutableDictionary，因为如果不使用copy，那么这些当可变类型的值被修改时，对应的NSString、NSArray、NSDictionary的值也会被修改。

2. block使用copy，是为了将block从栈空间移动到堆空间，从而使block可以保存block外传递进来的变量。

关于block做以下补充：

![block的类型](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/block的类型.png)
![block的copy](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/block的copy.png)
![block的变量捕获](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/block的变量捕获.png)

### 4. 这个写法会出什么问题： @property (copy) NSMutableArray *array;
1. 无论赋值对象是什么类型，copy都会将其转化为一个不可变类型，即array的实际类型为NSArray，那么当array调用NSMutableArray的api时，就会导致崩溃。
```swift
-[__NSArrayI removeObjectAtIndex:]: unrecognized selector sent to instance 0x7fcd1bc30460
```
2. 如果按题目的写法，其默认关键字还有 atomic ，atomic不是线程安全的，无法保证array多读单写。因为 atomic 本身是在array的 set 和 get 方法中加了锁，但是当通过 [array addObject:] 函数添加数据时，是不会调用 set 方法的，所以也就无法保证线程安全。另一方面同步锁因为在 set 和 get 方法中添加来锁，所以对性能会有一定的影响。所以在实际开发中，如果真的需要涉及到多读单写操作，我们也不会使用 atomic 来实现，而是在实际读写代码处，添加锁来实现。

### 5. 如何让自己的类用 copy 修饰符？如何重写带 copy 关键字的 setter？
如何让自己的类用 copy 修饰符
1. 需声明该类遵从 NSCopying 协议
2. 实现 NSCopying 协议。该协议只有一个方法:
 ```swift - (id)copyWithZone:(NSZone *)zone;```

如何重写带 copy 关键字的 setter
```swift
- setValue(NSString * value){
  if _value != value{
    _value = [value copy];
  }
}
```

### 6. @property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的
@property声明的属性，会在编译期生成对应的 getter、setter 方法，并会生成名称为 _属性名称 的实例变量。

要注意的两点：
1. 可以在类的实现代码里通过 @synthesize 语法来指定实例变量的名字
2. 如果属性声明时带有@dynamic关键字，则告诉计算机不需要在编译期生成 getter、setter 方法。

### 7. @protocol 和 category 中如何使用 @property

在协议和分类当中，都可以声明属性，但是我们先看看类对象（每个类只有一个类对象）的底层结构：
![objc_class](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/objc_class.png)


分类的信息会在运行时阶段被动态的加入到类对象的 class_rw_t (readwrite)中，但是从 class_rw_t 的结构可以看出，并没有能够保存 实例变量 的容器，而对象的实例变量是在编译期生成并保存在 class_ro_t （readonly）当中的。所以得出结论如下：

协议和分类中声明的属性，只会为属性生成对应的 getter、setter 方法，但是并不会生成实例变量。

我们可以使用运行时动态的给对象添加成员变量：

```swift
objc_setAssociatedObject
objc_getAssociatedObject
```
但这并不被推荐

### 8. runtime 如何实现 weak 属性
首先，苹果为了内存优化，将 weak 弱引用保存在散列表（Hash表）中：

![SideTables](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/SideTables.png)

一个对象，如果存在 weak 声明的属性，那么该属性就会被加入到一个数组中，而数组则会保存在这个散列表中。当这个对象被释放时，程序就会通过这个散列表，以对象为key，找到这个数组，并依次对其中的对象进行nil赋值操作。

### 9. weak属性需要在dealloc中置nil么？
不需要，理由请看8

### 10. @synthesize和@dynamic分别有什么作用？
@synthesize可以改变属性在编译器生成实例变量的名称。
@dynamic告知编译器不为属性生成对应的 getter、setter 方法。

### 11. ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些？
@property默认修饰符是atomic, readwrite, assign或strong

### 12. 用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题？
请查看问题3

### 13. @synthesize合成实例变量的规则是什么？假如property名为foo，存在一个名为_foo的实例变量，那么还会自动合成新变量么？
1. 如果指定了成员变量的名称,会生成一个指定的名称的成员变量。
2. 如果这个成员已经存在了就不再生成了。

所以存在一个名为_foo的实例变量，就不会再生成新变量了。

### 14. objc中向一个nil对象发送消息将会发生什么？
首先，需要搞明白2个问题：
1. 什么是isa指针
2. 消息传递机制

isa指针是用于对象指向类对象，类对象指向元类对象的一个指针。而类对象和元类对象中又分别存放对象方法和类方法。
在消息传递机制中，就是通过isa指针来寻找到方法的实际调用地址的。

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，然后在发送消息的时候，objc_msgSend方法不会返回值，所谓的返回内容都是具体调用时执行的。 那么，回到本题，如果向一个nil对象发送消息，首先在寻找对象的isa指针时就是0地址返回了，所以不会出现任何错误。

### 15. objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间有什么关系？
任何方法调用，都会在编译后转换为objc_msgSend()函数。该函数需要2个参数，一个是对象，一个是函数名称，即objc_msgSend(obj，@selector(foo))、

### 16. 什么时候会报unrecognized selector的异常？
当对象通过消息专递，动态方法解析，消息转发这一系列操作之后，发现这个方法任然无法被执行时，就会报该错。

什么是消息传递？
![消息传递](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/消息传递.png)
![消息发送](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/消息发送.png)

什么是动态方法解析？
![动态方法解析](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/动态方法解析.png)

什么是消息转发？
![消息转发](https://github.com/rogertan30/iOSInterviewQuestions/blob/master/images/消息转发.png)






