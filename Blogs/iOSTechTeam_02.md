# iOS Teach Team iOS深入理解Block

### 引言
> 在Objective-C日常开发中，Block是使用频率相对是比较多的，你不会每天都做启动优化，你也不会每天都做性能优化，但你有可能每天都在用Block，本文就着重介绍一下Block在日常开发中值得我们关注的点，大家一起学习。

---
* ### 代码规范
```
// 定义一个Block
typedef returnType (^BlockName)(parameterA, parameterB, ...);

eg: typedef void (^RequestResult)(BOOL result);

// 实例
^{
    NSLog(@"This is a block");
 }
```
---
* ### 本质

Block本质上是一个Objective-C的对象，它内部也有一个isa指针，它是一个封装了函数及函数调用环境的Objective-C对象，可以添加到NSArray及NSDictionary等集合中，它是基于C语言及运行时特性，有点类似标准的C函数，但除了可执行代码以外，另外包含了变量同堆或栈的自动绑定。

---
* ### 常用介绍

* Block的类型：

1. __NSGlobalBlock__
```
void (^exampleBlock)(void) = ^{
    // block
};
NSLog(@"exampleBlock is: %@",[exampleBlock class]); 
```
打印日志：`exampleBlock is: __NSGlobalBlock__`

如果一个Block没有访问外部局部变量，或者访问的是全局变量，或者静态局部变量，此时的Block就是一个全局Block，并且数据存储在全局区。

2. __NSStackBlock__
```
int temp = 100;
void (^exampleBlock)(void) = ^{
    // block
    NSLog(@"exampleBlock is: %d", temp);
};
NSLog(@"exampleBlock is: %@",[exampleBlock class]);
```
打印日志：`exampleBlock is: __NSMallocBlock__`？？？
不是说好的`__NSStackBlock__`的吗？为什么打印的是`__NSMallocBlock__`呢，这里是因为我们使用了ARC，Xcode默认帮我们做了很多事情，我们可以去Build Settings里面，找到Objective-C Automatic Reference Counting，并将其设置为No，然后再Run一次代码。
你会看到打印日志是：`exampleBlock is: __NSStackBlock__`

如果Block访问了外部局部变量，此时的Block就是一个栈Block，并且存储在栈区，由于栈区的释放是由系统控制，因此栈中的代码在作用域结束之后内存就会销毁，如果此时再调用Block就会发生问题，( **注：** 此代码运行在MRC下)如：
```
void (^simpleBlock)(void);
void callFunc() {
    int age = 10;
    simpleBlock = ^{
        NSLog(@"simpleBlock-----%d", age);
    };
}

int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        callFunc();
        simpleBlock();
        // Setup code that might create autoreleased objects goes here.
        appDelegateClassName = NSStringFromClass([AppDelegate class]);
    }
    return 0;
}
```

打印日志：`simpleBlock--------41044160`

3. __NSMallocBlock__

当一个`__NSStackBlock__`类型block做copy操作后就会将这个Block从栈上复制到堆上，而堆上的这个Block类型就是`__NSMallocBlock__`类型，在ARC环境下，编译器会根据情况，自动将Block从栈上copy到堆上，具体会进行copy的情况有如下4种：
* Block作为函数的返回值时；
* Block赋值给__strong指针，或者Block类型的成员变量时；
* Block作为Cocoa API中方法名含有usingBlock的方法参数时；
* Block作为GCD API的方法参数时；

---
### **__block的作用**

简单来说，__block作用是允许block内部访问和修改外部变量，在非MRC环境下还可以用来防止循环引用；

```
__block int age = 10;
void (^exampleBlock)(void) = ^{
    // block
    NSLog(@"1.age is: %d", age);
    age = 16;
    NSLog(@"2.age is: %d", age);
};
exampleBlock();
NSLog(@"3.age is: %d", age);
 ```
__block主要用来解决Block内部无法修改auto变量值的问题，为什么加上__block修饰之后，auto变量值就能修改了呢，这是因为，加上__block修饰之后，编译器会将__block变量包装成一个结构体__Block_byref_age_0，结构体内部*__forwarding是指向自身的指针，并且结构体内部还存储着外部auto变量。
```
struct __Block_byref_val_0 {
    void *__isa; // isa指针
    __Block_byref_val_0 *__forwarding; 
    int __flags;
    int __size; // Block结构体大小
    int age; // 捕获到的变量
}
```
![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_02/block_struct.jpg)

从上图可以看到，如果这block是在栈上，那么这个__forwarding指针就是指向它自己，当这个block从栈上复制到堆上后，栈上的__forwarding指针指向的是复制到堆上的__block结构体，堆上的__block结构体中的__forwarding指向的还是它自己，即age->__forwarding获取到堆上的__block结构体，age->__forwarding->age会把堆上的age赋值为16，因此不管是栈上还是堆上的__block结构体，最终使用到的都是堆上的__block结构体里面的数据。

---
### **__weak的作用** 
简单来说是为了防止循环引用。

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_02/block_self.png)

self本身会对block进行强引用，block会对其对象强进行引用，对于self也会形成强引用，这样就会造成循环引用的问题。我们可以通过使用__weak打破循环，使block对象对self弱引用。

此时我们注意，由于block对self的引用为weak引用，因此有可能在执行block时，self对象本身已经释放，那么我们如何保证self对象不在block内部释放呢，这就引出了下面__strong的作用。

---
### **__strong的作用**
简单来说，是防止 block 内部引用的外部 weak 变量被提前释放，进而在 block 内部无法获取 weak 变量以继续使用的情况；
```
__weak __typeof(self) weakSelf = self;
void (^exampleBlock)(void) = ^{
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    [strongSelf exampleFunc];
};
```
这样就保证了在block作用域结束之前，block内部都持有一个strongSelf对象可供使用。

但是，即便如此，依然有一个场景，就是执行```__strong __typeof(weakSelf) strongSelf = weakSelf;```之前，weakSefl 对象已经释放，这时如果是给 self 对象发送消息，这没有问题，Objective-C 的消息发送机制允许我们给一个 nil 对象发送消息，这不会出现问题。

但如果有额外的一些操作，比如说将 self 添加到数组，这时因为 self 为 nil，程序就会 Crash。

我们可以增加一层安全保护来解决这个问题，如：
```
__weak __typeof(self) weakSelf = self;
void (^exampleBlock)(void) = ^{
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        // Add opetation here
    }
};
```

---
### 拓展知识
* 思考题
Block内部修改NSMutableString、NSMutableArray、NSMutableDictionary对象，是否需要添加__block？

```
NSMutableArray *mutableArray = [[NSMutableArray alloc] init];
[mutableArray addObject:@"1"];
void (^exampleBlock)(void) = ^{
    // block
    [mutableArray addObject:@"2"];
};
exampleBlock();
NSLog(@"mutableArray: %@", mutableArray);
```
打印日志：
> mutableArray: (
    1,
    2
)

答案是：不需要，因为在Block内部，我们只是使用了对象 mutableArray 的内存地址，往其中添加内容，并没有修改其内存地址，因此不需要使用__block也可以正确执行。当我们只是使用局部变量的内存地址，而不是对其内存地址进行修改时，这时我们无需对其添加__block，如果添加了__block系统会自动创建相应的结构体，这种情况冗余且低效。

---
### 总结

使用block过程中需要我们关注的重点有4个：

1. Block的三种类型；
2. Block避免引起循环引用
3. Block对auto变量的copy操作
4. __block、__weak、__strong的作用

以上就是本文对Block的核心知识点的介绍，感谢阅读。

---
**最后：期望接下来能再写一篇！**

### 参考资料：
[Working with Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)