# iOS Teach Team iOS 深入理解 Block

### 引言
> 在 iOS 日常开发中，Block 的使用频率是比较多的，我们不会每天都做启动优化，也不会每天都做性能优化，但有可能每天都会用到 Block 。本文就着重介绍一下 Block 在日常开发中值得我们关注的技术点，大家一起学习。

---
* ### 代码规范
```
// 定义一个 Block
typedef returnType (^BlockName)(parameterA, parameterB, ...);

eg: typedef void (^RequestResult)(BOOL result);

// 实例
^{
    NSLog(@"This is a block");
 }
```
---
* ### **本质**

Block 本质上是一个 Objective-C 的对象，它内部也有一个 `isa` 指针，它是一个封装了函数及函数调用环境的 Objective-C 对象，可以添加到 `NSArray` 及 `NSDictionary` 等集合中，它是基于 `C` 语言及运行时特性，有点类似标准的 `C` 函数。但除了可执行代码以外，另外包含了变量同堆或栈的自动绑定。

---
* ### **常用介绍**

### **Block 的类型：**

1. __NSGlobalBlock__
```
void (^exampleBlock)(void) = ^{
    // block
};
NSLog(@"exampleBlock is: %@",[exampleBlock class]); 
```
打印日志：`exampleBlock is: __NSGlobalBlock__`

如果一个 Block 没有访问外部局部变量，或者访问的是全局变量，或者静态局部变量，此时的 Block 就是一个全局 Block ，并且数据存储在全局区。

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
不是说好的 `__NSStackBlock__` 的吗？为什么打印的是`__NSMallocBlock__` 呢？这里是因为我们使用了 ARC ，Xcode 默认帮我们做了很多事情。

我们可以去 `Build Settings` 里面，找到 `Objective-C Automatic Reference Counting` ，并将其设置为 `No` ，然后再 Run 一次代码。你会看到打印日志是：`exampleBlock is: __NSStackBlock__`

如果 Block 访问了外部局部变量，此时的 Block 就是一个栈 Block ，并且存储在栈区。由于栈区的释放是由系统控制，因此栈中的代码在作用域结束之后内存就会销毁，如果此时再调用 Block 就会发生问题，( **注：** 此代码运行在MRC下)如：
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

当一个 `__NSStackBlock__` 类型 Block 做 `copy` 操作后就会将这个 Block 从栈上复制到堆上，而堆上的这个 Block 类型就是 `__NSMallocBlock__` 类型。在 ARC 环境下，编译器会根据情况，自动将 Block 从栈上 `copy` 到堆上。具体会进行 `copy` 的情况有如下 4 种：
* Block 作为函数的返回值时；
* Block 赋值给 __strong 指针，或者赋值给 Block 类型的成员变量时；
* Block 作为 Cocoa API 中方法名含有 usingBlock 的方法参数时；
* Block 作为 GCD API 的方法参数时；

---
### **__block 的作用**

简单来说，`__block` 作用是允许 `block` 内部访问和修改外部变量，在 ARC 环境下还可以用来防止循环引用；

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
`__block` 主要用来解决 `block` 内部无法修改 `auto` 变量值的问题，为什么加上 `__block` 修饰之后，`auto` 变量值就能修改了呢？

这是因为，加上 `__block` 修饰之后，编译器会将 `__block` 变量包装成一个结构体 `__Block_byref_age_0` ，结构体内部 `*__forwarding` 是指向自身的指针，并且结构体内部还存储着外部 `auto` 变量。
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

从上图可以看到，如果 `block` 是在栈上，那么这个 `__forwarding` 指针就是指向它自己，当这个 `block` 从栈上复制到堆上后，栈上的 `__forwarding` 指针指向的是复制到堆上的 `__block` 结构体。堆上的 `__block` 结构体中的 `__forwarding` 指向的还是它自己，即 `age->__forwarding` 获取到堆上的 `__block` 结构体，`age->__forwarding->age` 会把堆上的 `age` 赋值为 16 。因此不管是栈上还是堆上的 `__block` 结构体，最终使用到的都是堆上的 `__block` 结构体里面的数据。

---
### **__weak的作用** 

简单来说是为了防止循环引用。

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_02/block_self.png)

`self` 本身会对 `block` 进行强引用，`block` 会对其对象强进行引用，对于 `self` 也会形成强引用，这样就会造成循环引用的问题。我们可以通过使用 `__weak` 打破循环，使 `block` 对象对 `self` 弱引用。

此时我们注意，由于 `block` 对 `self` 的引用为 `weak` 引用，因此有可能在执行 `block` 时，`self` 对象本身已经释放，那么我们如何保证 `self` 对象不在 `block` 内部释放呢？这就引出了下面`__strong` 的作用。

---
### **__strong的作用**
简单来说，是防止 `block` 内部引用的外部 `weak` 变量被提前释放，进而在 `block` 内部无法获取 `weak` 变量以继续使用的情况；
```
__weak __typeof(self) weakSelf = self;
void (^exampleBlock)(void) = ^{
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    [strongSelf exampleFunc];
};
```
这样就保证了在 `block` 作用域结束之前，`block` 内部都持有一个 `strongSelf` 对象可供使用。

但是，即便如此，依然有一个场景，就是执行 ```__strong __typeof(weakSelf) strongSelf = weakSelf;``` 之前，`weakSelf` 对象已经释放，这时如果给 `self` 对象发送消息，这没有问题，Objective-C 的消息发送机制允许我们给一个 `nil` 对象发送消息，这不会出现问题。

但如果有额外的一些操作，比如说将 `self` 添加到数组，这时因为 `self` 为 `nil`，程序就会 Crash。

我们可以增加一层安全保护来解决这个问题，如：
```
__weak __typeof(self) weakSelf = self;
void (^exampleBlock)(void) = ^{
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        // Add operation here
    }
};
```

---
### **拓展知识**
* 思考题

`Block` 内修改外部 `NSMutableString` 、`NSMutableArray` 、`NSMutableDictionary` 对象，是否需要添加 `__block` 修饰？

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

答案是：不需要，因为在 `Block` 内部，我们只是使用了对象 `mutableArray` 的内存地址，往其中添加内容。并没有修改其内存地址，因此不需要使用 `__block` 也可以正确执行。当我们只是使用局部变量的内存地址，而不是对其内存地址进行修改时，我们无需对其添加`__block` ，如果添加了 `__block` 系统会自动创建相应的结构体，这种情况冗余且低效。

---
### **总结**

使用 `Block` 过程中需要我们关注的重点有 4 个：

1. Block 的三种类型；
2. Block 避免引起循环引用；
3. Block 对 auto 变量的 copy 操作；
4. __block、__weak、__strong 的作用；

以上就是本文对 Block 的核心知识点的介绍，感谢阅读。

---
### **参考资料：**
[Working with Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系交流，感谢阅读。