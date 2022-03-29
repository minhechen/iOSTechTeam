# iOS Teach Team iOS 全面理解 Nullability 即 nil, Nil, NULL, NSNull, kCFNulL 及空值修饰符 nullable, __nullable, _Nullable, nonnull, __nonnull, __Nonnull 详细解读

### **引言**

日常开发过程中，我们经常会碰到空值、空指针、空对象、空的占位对象等。在一些情况下，如果判断不好或者处理方式不对，可能会引起程序运行异常，有些特殊情况甚至会导致 Crash ，因此，熟练了解掌握它们之间的区别，将有助于我们写出更高质量的代码，本文就详细介绍一下它们之间的差别与注意事项。

---
### **`Nullability` 的由来**

在 `Swift` 中，我们会使用 `?` 和 `!` 去显式声明一个对象或函数的的参数及函数的返回值是 `optional` 还是 `non-optional` ，而在 Objective-C 中则没有这一区分。当我们在 Swift 与 Objective-C 混编开发时，由于 `Swift` 编译器并不知道这个 Objective-C 对象或函数的参数或者函数的返回值是  `optional` 还是 `non-optional` ，这种情况下编译器会隐式地都当成是 `non-optional` 来处理，所以经常性的会因为把一个空值当做 `non-optional` 来处理而导致程序 Crash。

因此为了解决这个问题，苹果在 `Xcode 6.3` 为 `C` 及 `Objective-C` 引入了一个新特性： `Nullability Annotations` 。
> Nullability annotations for C and Objective-C are available starting in Xcode 6.3


#### **空值相关**

* nil

```
#ifndef nil
# if __has_feature(cxx_nullptr)
#   define nil nullptr
# else
#   define nil __DARWIN_NULL
# endif
#endif
```

**宏：** Objective-C 对象使用，表示对象为空 `(id)0`。常作为对象的空指针和数组的结束标志。


* Nil 

```
#ifndef Nil
# if __has_feature(cxx_nullptr)
#   define Nil nullptr
# else
#   define Nil __DARWIN_NULL
# endif
#endif
```

**宏：** Objective-C 类使用，表示类指向空 `(Class)0`，即类指针为空。


* NULL

```
#if !defined(NULL)
#if defined(__GNUG__)
    #define NULL __null
#elif defined(__cplusplus)
    #define NULL 0
#else
    #define NULL ((void *)0)
#endif
#endif
```

**宏：** `C` 语言指针使用，表示空指针 `(void *)0`。常在用基本数据类型为空情况。

* NSNull

```
@interface NSNull : NSObject <NSCopying, NSSecureCoding>
// 返回单例对象
+ (NSNull *)null;

@end
```

**Objective-C 类：** 是一个空值的单例对象 `[NSNull null]`，继承自 `NSObject` ，可用于表示不允许空值的集合对象中。


* kCFNull

```
const CFNullRef kCFNull;	// the singleton null instance
```

单例 `CFNull` 对象，等于 `NSNull` 单例

---

#### **空值修饰符相关：**

有些同学可能奇怪为什么会有这么多写法，其实他们都是两两成对出现的：

* nullable & nonnull 

* _Nullable & _Nonnull 

*  __nullable & __nonnull 

其中 `nonnull` ， `_Nonnull` ， `__nonnull` 三个修饰的参数表示对象不可以是 NULL 或 nil ，如果向它们修饰的对象传 NULL 或 nil 的话，编译器会产生警告。

其中 `nullable` ， `_Nullable` ， `__nullable` 三个修饰的参数表示对象可以是 NULL 或 nil 。

功能上来说，它们没有什么区别，只是使用上位置有所区别：

```
// 不带下划线关键词 nonnull ， nullable 不同应用场景下的摆放位置
@property(nonnull, nonatomic, copy) NSString *name;
@property(nullable, nonatomic, copy) NSString *company;
- (nonnull NSString *)firstString:(nullable NSString *)str;

// 带下划线关键词 _Nonnull ，_Nullable 不同应用场景下的摆放位置
@property(nonatomic, copy) NSString * _Nonnull name;
@property(nonatomic, copy) NSString * _Nullable company;
- (NSString * _Nonnull)secondString:(NSString * _Nullable)str;

// 带下划线关键词 __nonnull ，__nullable 不同应用场景下的摆放位置
@property(nonatomic, copy) NSString * __nonnull name;
@property(nonatomic, copy) NSString * __nullable company;
- (NSString * __nonnull)thirdString:(NSString * __nullable)str;
```

关于 `Nullability Annotations` 官方介绍：
> <br>
> This feature was first released in Xcode 6.3 with the keywords __nullable and __nonnull. Due to potential conflicts with third-party libraries, we’ve changed them in Xcode 7 to the _Nullable and _Nonnull you see here. However, for compatibility with Xcode 6.3 we’ve predefined macros __nullable and __nonnull to expand to the new names.
>
> 此功能首次在 Xcode 6.3 中使用关键字 __nullable 和 __nonnull 。由于与第三方库的潜在冲突，我们在 Xcode 7 中将它们更改为您在此处看到的 _Nullable 和 _Nonnull。但是，为了与 Xcode 6.3 兼容，我们预定义了宏 __nullable 和 __nonnull 以扩展新名称
>
苹果也支持使用没有下划线的写法 `nonnull` ，`nullable` ，于是就有了三种写法。

另外我们还会经常看到下面两个关键词：

* null_resettable

* null_unspecified & __null_unspecified & _Null_unspecified 

其中 `null_resettable` ，`getter` 不能返回空，`setter` 可以为空(**注意：** 使用 `null_resettable` 必须重写 `getter` 方法和 `setter` 方法，处理值为空的情况)。

其中 `null_unspecified` ，`__null_unspecified` ， `_Null_unspecified` ， 表示不确定是否为空。

二者的具体使用：
```
// null_resettable 具体应用
@property (nonatomic, strong, null_resettable) NSString *name;

// 不带下划线关键词 null_unspecified 不同应用场景下的摆放位置
@property(null_unspecified, nonatomic, copy) NSString *name;
- (null_unspecified NSString *)firstString:(null_unspecified NSString *)str;

// 带下划线关键词 __null_unspecified 不同应用场景下的摆放位置
@property(nonatomic, copy) NSString * __null_unspecified name;
- (NSString * __null_unspecified)secondString:(NSString * __null_unspecified)str;

// 带下划线关键词 _Null_unspecified 不同应用场景下的摆放位置
@property(nonatomic, copy) NSString * _Null_unspecified name;
- (NSString * _Null_unspecified)thirdString:(NSString * _Null_unspecified)str;

```

**注意：**

1. 可空性关键词 `nonnull` ，`nullable` 等只能修饰对象，不能修饰基本数据类型。
2. 在`NS_ASSUME_NONNULL_BEGIN` 和 `NS_ASSUME_NONNULL_END` 之间，定义的所有对象属性和方法默认都是 `nonnull` 。

---
### **拓展知识**

* Sending Messages to nil

用过 Objective-C 开发的同学应该都非常熟悉这句话，正如官方描述的：
> In Objective-C, it is valid to send a message to nil—it simply has no effect at runtime. 

这说明 `nil` 本身可以足够安全地调用方法而不会崩溃。

在 `Swift` 中，我们有更多的安全性。我们可以给 `nil` 发“消息”（其实并没有），但前提是它们是链式可选项（chained optional）。只有当我们使用可选时，`nil` 才能成为一个有存在感的“something”。

* JSON 数据中带有 null 值情况

解析 `JSON` 数据时，如果接口返回数据中把 `NSNull` 传给我们，解析出来就是 `null` 空对象，如下：
```
{
  "title": "iOS Engineer",
  "age": 18,
  "name": null
}
```
这时当我们给 `null`（ `NSNull` 对象）发送消息的话，很大可能会直接Crash（ `null` 是有内存的）。

如何解决这个问题呢，通常有一下几种方式：

1. 接口端调整，修改默认数据的方式，在创建表的时候，添加上 'not null default' ；
2. 对可能出现空的字段进行非空判断；
```
if (![object isKindOfClass:[NSNull class]]){
    // ...
}
```
3. 对 `JSON` 进行字符串匹配, 替换 `null` 为空字符 ""（**注意：** 这个方法可能会有问题）；
4. 解析数据时对类型进行检查，并把 `NSNull` 类型的值替换成 `nil`；
5. 如果使用的是 `AFNetworking` 请求数据，可以使用其提供的 `((AFJSONResponseSerializer *)manager.responseSerializer).removesKeysWithNullValues = YES;` 去掉空值；
6. 使用第三方库 `NullSafe`，它是 `NSNull` 上的一个简单的 `category` ，为无法识别的消息返回 `nil` ，其代码实现非常简单，具体可见源码： [NullSafe](https://github.com/nicklockwood/NullSafe) ( https://github.com/nicklockwood/NullSafe )


---
### **总结**

随着越来越多的项目使用 `Objective-C` 与 `Swift` 混编开发，`Nullability` 越来越需要大家引起重视，一旦使用不当，可能在代码的某个角落就会出现一个 bug 甚至导致 App Crash。以上就是本文对 `iOS Nullability` 相关知识点的介绍，希望这篇文章对你有所帮助，感谢阅读。

---

### **参考资料：**

* [Nullability Attributes](https://clang.llvm.org/docs/AttributeReference.html#nullability-attributes) ( https://clang.llvm.org/docs/AttributeReference.html#nullability-attributes )

* [nil / Nil / NULL / NSNull](https://nshipster.cn/nil/) ( https://nshipster.cn/nil/ )

* [Difference between nullable, __nullable and _Nullable in Objective-C](https://stackoverflow.com/questions/32452889/difference-between-nullable-nullable-and-nullable-in-objective-c) ( https://stackoverflow.com/questions/32452889/difference-between-nullable-nullable-and-nullable-in-objective-c )

* [[NSNull length]: unrecognized selector sent to JSON objects](https://stackoverflow.com/questions/16607960/nsnull-length-unrecognized-selector-sent-to-json-objects/16610117) ( https://stackoverflow.com/questions/16607960/nsnull-length-unrecognized-selector-sent-to-json-objects/16610117 )

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系进群学习交流，感谢阅读。