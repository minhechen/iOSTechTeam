# iOS Teach Team iOS中的空 nil, Nil, NULL, NSNull, kCFNulL 及空修饰符 nullable, __nullable, _Nullable, nonnull, __nonnull, __Nonnull 详细解读

### 引言

日常开发过程中，我们经常会碰到空值、空指针、空对象、空的占位对象等。在一些情况下，如果判断不好或者处理方式不对，可能会引起程序运行异常，有些特殊情况甚至会导致 Crash ，因此，熟练了解掌握它们之间的区别，将有助于我们写出更高质量的代码，本文就详细介绍一下它们之间的差别与注意事项。

---
### **常用介绍**

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

宏，Objective-C 对象使用，表示对象为空 `(id)0`。常作为对象的空指针和数组的结束标志。


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

宏，Objective-C 类使用，表示类指向空 `(Class)0`，即类指针为空。


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

宏，`C` 语言指针使用，表示空指针 `(void *)0`。常用基本数据类型为空情况。

* NSNull

```
@interface NSNull : NSObject <NSCopying, NSSecureCoding>
// 返回单例对象
+ (NSNull *)null;

@end
```

Objective-C 类，是一个空值的单例对象 `[NSNull null]`，继承自 `NSObject` ，可用于表示不允许空值的集合对象中。


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
苹果也支持使用没有下户线的写法 `nonnull` ，`nullable` ，于是就有了三种写法。

另外我们还会经常看到下面两个关键词：

* null_resettable

* null_unspecified & __null_unspecified & _Null_unspecified 

其中 `null_resettable` ，`getter` 不能返回空，`setter` 可以为空(**注意：** 使用 `null_resettable` 必须重写 `getter` 方法和 `setter` 方法，处理传递的值为空的情况)。

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

1. 其中关键词 `nonnull` ，`nullable` ，`null_resettable` ， `_Null_unspecified` 只能修饰对象，不能修饰基本数据类型。
2. 在`NS_ASSUME_NONNULL_BEGIN` 和 `NS_ASSUME_NONNULL_END` 之间，定义的所有对象属性和方法默认都是 `nonnull` 。


---
### 拓展知识

---
### 总结


---
**最后：期望接下来能再写一篇！**

### 参考资料：

[Nullability Attributes](https://clang.llvm.org/docs/AttributeReference.html#nullability-attributes)

[nil / Nil / NULL / NSNull](https://nshipster.cn/nil/)

[Difference between nullable, __nullable and _Nullable in Objective-C](https://stackoverflow.com/questions/32452889/difference-between-nullable-nullable-and-nullable-in-objective-c)