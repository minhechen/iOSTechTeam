# iOS Teach Team iOS 探究 | 第七篇 异常处理和错误

探究系列**已发布文章**列表，有兴趣的同学可以翻阅一下：

[第一篇 | iOS 属性 @property 详细探究](https://mp.weixin.qq.com/s?__biz=MzAxNjIzNjI4Mg==&mid=2450076636&idx=1&sn=bd581243bfce19b83c508336f780b3e8&chksm=8c0a7849bb7df15fb2d280e8f0e971f1d31a5a435179fa62d28ab6ced8988c875f4366c991db&scene=21&token=898954029&lang=zh_CN#wechat_redirect)

[第二篇 | iOS 深入理解 Block 使用及原理](https://mp.weixin.qq.com/s?__biz=MzAxNjIzNjI4Mg==&mid=2450076720&idx=1&sn=e80b68663ea810d223a9e3a9240f0a9f&chksm=8c0a7fa5bb7df6b3d63f19040fc99b66600a77a301d1ee89d788048a8720cba6691257204cee&token=898954029&lang=zh_CN#rd)

[第三篇 | iOS 类别 Category 和扩展 Extension 及关联对象详解](https://mp.weixin.qq.com/s?__biz=MzAxNjIzNjI4Mg==&mid=2450076808&idx=1&sn=986dfd85f4785f79f4517cce4cf9b690&chksm=8c0a7f1dbb7df60b1dc2073f98c379f1acb50fd0cd8508141b6072633791c1d57d0e66016cb0&token=898954029&lang=zh_CN#rd)

[第四篇 | iOS 常用锁 NSLock ，@synchronized 等的底层实现详解](https://mp.weixin.qq.com/s?__biz=MzAxNjIzNjI4Mg==&mid=2450076878&idx=1&sn=6c452c7d885826ce12db3f754111b7cd&chksm=8c0a7f5bbb7df64d59e6461ad8cb86e6922bda3af35e488b0d7c924d16e91abe174d72de3f38&token=898954029&lang=zh_CN#rd)

[第五篇 | iOS 全面理解 Nullability](https://mp.weixin.qq.com/s?__biz=MzAxNjIzNjI4Mg==&mid=2450076953&idx=1&sn=b0f51d90c9a6eff16391fa92dd2b12dd&chksm=8c0a7e8cbb7df79a2bfe089f2d6227aa5cde67f13dfa09117184892185245287ab388a5bce25&token=898954029&lang=zh_CN#rd)

[第六篇 Equality 详细探究](https://mp.weixin.qq.com/s?__biz=MzAxNjIzNjI4Mg==&mid=2450076977&idx=1&sn=9f50109bd27400c19e43826ed2c392d4&chksm=8c0a7ea4bb7df7b21eb1e8b8a9f56d250d83a201b68cd1d515beebbba16157601566cf45b6a3&token=1873620973&lang=zh_CN#rd)


------- 正文开始 -------

### **引言**
> `Objective-C` 语言具有类似于 `Java` 和 `C++` 的异常处理语法。通过将此语法与 `NSException`、`NSError` 或自定义类一起使用，我们可以为程序添加健壮的错误处理。本文介绍一下异常语法的使用及如何处理异常情况。

---
* ### **常用介绍**

* 启用异常处理

`GNU Compiler Collection (GCC) 3.3` 及更高版本开始对 `Objective-C` 提供语言级异常处理支持。只需要打开对这些功能的支持 `-fobjc-exceptions` 开关。（**注意：** 此开关使应用程序只能在 OS X v10.3 及更高版本中运行，因为早期版本的软件中不存在对异常处理和同步的运行时支持。）

* 异常处理

异常是中断正常程序执行流程的特殊情况。硬件和软件可能产生异常的原因有很多（通常称为引发或抛出异常）。包括算术错误，例如被零除、下溢或溢出、调用未定义的指令（例如尝试调用未实现的方法）以及尝试越界访问集合元素等等。

`Objective-C` 异常支持涉及四个编译器指令：`@try`、`@catch`、`@throw` 和 `@finally`：

可能引发异常的代码包含在 `@try{}` 块中。`@catch{}` 块包含在 `@try{}` 块中抛出的异常的异常处理逻辑。我们可以有多个 `@catch{}` 块来捕获不同类型的异常。

当我们使用 `@throw` 指令抛出一个异常，它本质上是一个`Objective-C` 对象。我们通常使用 `NSException` 对象，但这不是必须的。

@finally{} 块包含无论是否引发异常都必须执行的代码。下面例子描述了一个简单的异常处理：
```
Cup *cup = [[Cup alloc] init];
 
@try {
    [cup fill];
}
@catch (NSException *exception) {
    NSLog(@"main: Caught %@: %@", [exception name], [exception reason]);
}
@finally {
    // [cup release];
}

```

* 捕获不同类型的异常

要捕获 `@try{}` 块中引发的异常，需要在 `@try{}` 块之后使用一个或多个 `@catch{}` 块。`@catch{}` 块应按从最具体异常到最不具体异常的顺序排列。这样，我们可以将异常处理定制为组，如下面的一个异常处理程序：

```
@try {
    ...
}
@catch (CustomException *ce) {   // 1
    ...
}
@catch (NSException *ne) {       // 2
    // Perform processing necessary at this level.
    ...
 
}
@catch (id ue) {
    ...
}
@finally {                       // 3
    // Perform processing necessary whether an exception occurred or not.
    ...
}
```

以下对应上面代码具体位置的作用：
1. 捕获最具体的异常类型。
2. 捕获一般的异常类型。
3. 无论是否引发异常，都需要执行的清理或其他处理操作。

* 抛出异常

当需要抛出异常时，我们需要使用适当的信息（例如异常名称和引发异常的原因）实例化一个对象，以便我们快速定位及查找异常的原因。
```
NSException *exception = [NSException exceptionWithName: @"HotTeaException" reason: @"The tea is too hot" userInfo: nil];
@throw exception;
```

**注意：** 在很多环境中，异常的使用相当普遍。例如，我们可能会抛出异常来表示程序无法正常执行，例如文件丢失或数据无法正确解析。在 `Objective-C` 中，异常是资源密集型。我们不应该将异常用于一般的流控制，或者仅仅表示错误。相反，我们应该使用方法或函数的返回值来表示发生了错误，并在错误对象中提供相关问题的信息。

在 `@catch{}` 块中，我们可以使用 `@throw` 指令抛出捕获的异常，而无需提供参数。在这种情况下省略参数有助于使我们的代码更具可读性。

不仅限于抛出 `NSException` 对象。我们可以将任何 `Objective-C` 对象作为异常对象抛出。`NSException` 类提供了有助于异常处理的方法，但如果我们愿意，也可以实现自己的方法。还可以继承 `NSException` 来实现特殊类型的异常，例如文件系统异常或通信异常等。

* ### **具体错误及异常处理**

每个程序都必须处理运行时发生的错误。例如，该程序可能无法打开文件，或者它可能无法解析 XML 文档。通常，诸如此类的错误需要程序通知用户，但也可以尝试让程序解决错误。

Cocoa（和 Cocoa Touch）为开发人员提供了用于处理这些任务的编程工具：`Foundation` 中的 `NSError` 类和 `Application Kit` 中的新方法和机制，支持应用程序中的错误处理。`NSError` 对象封装了特定的错误信息，包括引发错误的域（子系统）和要在错误警告中显示的本地化字符串。还允许应用程序中的各种对象细化错误对象中的信息，并可能尝试从错误中恢复。

**注意：** `NSError` 类在 OS X 和 iOS 上都可用。但是，错误响应和错误恢复 `API` 和机制仅在 `Application Kit (OS X)` 中可用。


* 错误对象、域和代码

Cocoa 程序使用 `NSError` 对象来传递用户需要了解的运行时的错误信息。在大多数情况下，程序会在对话框或 `Log` 中显示此错误信息。但它也可能会提示用户并要求用户尝试从错误中恢复或尝试自行纠正错误。

NSError 对象（或者简单地说，错误对象）的核心属性是错误域、特定于域的错误代码和包含与错误相关的对象的“用户信息”字典，最重要的是描述和恢复字符串。这里重点解释一下错误对象。


* 为什么有错误对象？

因为它们是对象，所以 `NSError` 类的实例比简单的错误代码和错误字符串有几个优点。它们一次封装了几条错误信息，包括各种本地化的错误字符串。`NSError` 对象也可以被归档和复制，它们可以在应用程序中传递和修改。尽管 `NSError` 不是一个抽象类（因此可以直接使用），但我们可以通过子类化来扩展 `NSError` 类。

由于分层错误域的概念，NSError 对象可以嵌入来自底层子系统的错误，从而提供有关错误的更详细和细微差别的信息。错误对象还通过保存对指定为错误恢复尝试者的对象的引用来提供错误恢复机制。

* 错误域

很大程度上由于历史原因，OS X 中的错误代码被隔离到域中。例如，键入为 `OSStatus` 的 `Carbon` 错误代码起源于 OS X 之前的 Macintosh 操作系统版本。另一方面，`POSIX` 错误代码源自 UNIX 的各种符合 POSIX 的“风格”，例如 BSD 。Foundation 框架在 `NSError.h` 中声明了以下四个主要错误域的字符串常量：

```
NSMachErrorDomain
NSPOSIXErrorDomain
NSOSStatusErrorDomain
NSCocoaErrorDomain
```
上述域常数序列表示域的一般分层，`Mach` 误差域位于最低层。我们可以通过向 `NSError` 对象发送域消息来获取错误的域。

除了四个主要域之外，还有特定于框架甚至是类组或单个类的错误域。例如，`Web Kit` 框架在其 `Objective-C` 实现中具有自己的错误域 `WebKitErrorDomain`。在 `Foundation` 框架中，`URL` 类和 `XML` 类 ( `NSXMLParserErrorDomain` ) 一样有自己的错误域 ( `NSURLErrorDomain` )。`NSStream` 类本身定义了两个错误域，一个用于 `SSL` 错误，另一个用于 `SOCKS` 错误。

Cocoa 错误域 ( `NSCocoaErrorDomain` ) 包括 Cocoa 框架的所有错误代码。当然，那些框架的特定类域中的错误代码除外。这些框架不仅包括 `Foundation`、`UIKit` 和 `Application Kit`，还包括 `Core Data` 和可能的其他 `Objective-C` 框架。（Cocoa 框架中与 Cocoa 错误域分离的错误域是在引入后者之前定义的）

域有几个有用的目的。它们为 Cocoa 程序提供了一种方法来识别正在检测错误的 OS X 子系统。它们还有助于防止来自具有相同数值的不同子系统的错误代码之间的冲突。此外，域允许基于子系统分层的错误代码之间的因果关系；例如，`NSOSStatusErrorDomain` 中的错误可能在 `NSMachErrorDomain` 中存在潜在错误。

我们可以创建自己的错误域和错误代码，以便在自己的框架或者自己的应用程序中使用。建议域的字符串常量采用 `com.company.framework_or_app.ErrorDomain` 的形式。

* 错误代码

错误代码标识特定域中的特定错误。它是一个有符号整数，分配为程序符号的值。我们可以通过向 `NSError` 对象发送代码消息来获取错误代码。通常错误代码在每个主要域的一个或多个头文件中声明和记录。

POSIX 错误代码声明的一部分：
```
#define EPERM       1       /* Operation not permitted */
#define ENOENT      2       /* No such file or directory */
#define ESRCH       3       /* No such process */
#define EINTR       4       /* Interrupted system call */
#define EIO         5       /* Input/output error */
#define ENXIO       6       /* Device not configured */
#define E2BIG       7       /* Argument list too long */
#define ENOEXEC     8       /* Exec format error */
#define EBADF       9       /* Bad file descriptor */
#define ECHILD      10      /* No child processes */
#define EDEADLK     11      /* Resource deadlock avoided */
                            /* 11 was EAGAIN */
#define ENOMEM      12      /* Cannot allocate memory */
#define EACCES      13      /* Permission denied */
#define EFAULT      14      /* Bad address *#H
```

我们可以选择要测试的错误条件，并在类似于下面的代码中使用它们：

```
// underError is underlying-error object of a Cocoa-domain error
if ( [[underError domain] isEqualToString:NSPOSIXErrorDomain] ) {
    switch([underError code]) {
        case EIO:
        {
            // handle POSIX I/O error
        }
        case EACCES:
        {
            // handle POSIX permissions error
        {
    // etc.
    }
}
```
可以声明自定义的错误代码在应用程序或框架中使用，但错误代码应属于我们自己的域。永远不应该将错误代码添加到现有的，且没有“拥有”权限的域中。

* 用户信息字典

每个 `NSError` 对象都有一个“用户信息”字典来保存域和代码之外的错误信息。我们可以通过向 `NSError` 对象发送 `userInfo` 消息来访问该字典。`NSDictionary` 对象相对于另一种容器对象的优势在于它是灵活的；它甚至可以携带有关错误的自定义信息。但是所有用户信息字典都包含（或可以包含）几个与错误相关的预定义字符串和对象值。

* 创建和返回 NSError 对象

我们可以声明和实现自己的间接返回 `NSError` 对象的方法。适合 `NSError` 参数的方法是打开和读取文件、加载资源、解析格式化文本等的方法。通常，这些方法不应通过 `NSError` 对象的存在来指示错误。相反，它们应该从方法中返回 `NO` 或 `nil` 以指示发生了错误。返回 `NSError` 对象来描述错误。

如果要在此类方法的实现中通过引用返回 `NSError` 对象，则必须创建 `NSError` 对象。我们可以通过分配它然后使用 `NSError` 的 `initWithDomain:code:userInfo:` 方法或使用类工厂方法 `errorWithDomain:code:userInfo:` 来创建一个错误对象。正如这两种方法的关键字所示，必须为初始化程序提供一个域（字符串常量）、一个错误代码（一个有符号整数）和一个包含描述性和支持信息的“用户信息”字典。

注意：只有在我们的方法中有错误并且方法直接返回 NO 时，才应该修改 NSError 参数。在将对象分配给它之前，确认参数是否为非 NULL，并且永远不要将 nil 分配给错误参数。

为了便于说明，下面的代码调用 POSIX 层的 open 函数来打开文件。如果此函数返回错误，则该方法创建 NSPOSIXErrorDomain 的 NSError 对象，该对象用作返回给调用者的自定义错误域的基础错误。

```
- (NSString *)fooFromPath:(NSString *)path error:(NSError **)anError {
 
    const char *fileRep = [path fileSystemRepresentation];
    int fd = open(fileRep, O_RDWR|O_NONBLOCK, 0);
 
    if (fd == -1) {
 
        if (anError != NULL) {
            NSString *description;
            NSDictionary *uDict;
            int errCode;
 
            if (errno == ENOENT) {
                description = NSLocalizedString(@"No file or directory at requested location", @"");
                errCode = MyCustomNoFileError;
            } else if (errno == EIO) {
                // Continue for each possible POSIX error...
            }
 
            // Create the underlying error.
            NSError *underlyingError = [[NSError alloc] initWithDomain:NSPOSIXErrorDomain
                code:errno userInfo:nil];
            // Create and return the custom domain error.
            NSDictionary *errorDictionary = @{ NSLocalizedDescriptionKey : description,
                NSUnderlyingErrorKey : underlyingError, NSFilePathErrorKey : path };
 
            *anError = [[NSError alloc] initWithDomain:MyCustomErrorDomain
                    code:errCode userInfo:errorDictionary];
        }
        return nil;
    }
}
```

* 关于错误和异常的说明

重要的是要记住 `Cocoa` 和 `Cocoa Touch` 中错误对象和异常对象之间的区别，以及何时在代码中使用其中一个或另一个。它们各自有不同的用途，不应混淆。

异常（由 `NSException` 对象表示）用于编程错误，例如数组越界或无效的方法参数。用户级错误（由 `NSError` 对象表示）用于运行时错误，例如找不到文件或无法读取某种编码的字符串时。引起异常的条件是由于编程错误；我们应该在发版之前处理这些错误。运行时错误总是会发生。因此，我们应该尽可能详细地向用户展示这些错误（通过 `NSError` 对象）。

尽管理想情况下应该在发版之前处理异常，但由于某些真正异常的情况（例如“内存不足”或“启动卷不可用”），已发版的应用程序仍然会遇到这些异常。最好让应用程序的最高层即全局应用程序（或者说系统本身）来处理这些情况。

---
### **拓展知识**

---
### **总结**

本文着重介绍比较久远的官方文档，文档虽然比较旧了，但依然是常读常新。比较详细的介绍了异常和错误的相关技术点和细节。可能有些同学日常开发中不是特别关注这一点，但我相信如果你花些时间在异常和错误处理上面，以后再碰到线上问题，或者发现异常错误相关 bug 时，会变得更游刃有余。

### **参考资料：**
* [Exception Handling](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocExceptionHandling.html#//apple_ref/doc/uid/TP30001163-CH13-SW1)

* [Introduction to Error Handling Programming Guide For Cocoa](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ErrorHandlingCocoa/ErrorHandling/ErrorHandling.html#//apple_ref/doc/uid/TP40001806-CH201-SW1)

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系进群学习交流，感谢阅读。