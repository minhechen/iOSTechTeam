# iOS Teach Team iOS中的 Nullability 即 nil, Nil, NULL, NSNull, kCFNulL 及空修饰符 nullable, __nullable, _Nullable, nonnull, __nonnull, __Nonnull 详细解读

### **引言**

日常开发过程中，我们经常会碰到空值、空指针、空对象、空的占位对象等。在一些情况下，如果判断不好或者处理方式不对，可能会引起程序运行异常，有些特殊情况甚至会导致 Crash ，因此，熟练了解掌握它们之间的区别，将有助于我们写出更高质量的代码，本文就详细介绍一下它们之间的差别与注意事项。

---
### **`Nullability` 的由来**

在 `Swift` 中，我们会使用 ? 和 ! 去显式声明一个对象、函数的的参数及函数的返回值是 `optional` 还是 `non-optional` ，而在 `Objective-C` 中则没有这一区分。当我们在 `Swift` 与 `Objective-C` 混编开发时，由于 `Swift` 编译器并不知道这个 `Objective-C` 对象、函数的参数或者函数的返回值是  `optional` 还是 `non-optional` ，这种情况下编译器会隐式地都当成是 `non-optional` 来处理，所以经常性的会因为把一个空值当做 `non-optional` 来处理而导致程序 Crash。

因此为了解决这个问题，苹果在 `Xcode 6.3` 引入了一个为 `C` 及 `Objective-C` 的新特性： `Nullability Annotations` 。
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

1. 可控性关键词 `nonnull` ，`nullable` 等只能修饰对象，不能修饰基本数据类型。
2. 在`NS_ASSUME_NONNULL_BEGIN` 和 `NS_ASSUME_NONNULL_END` 之间，定义的所有对象属性和方法默认都是 `nonnull` 。

---
### 拓展知识

1. `isEqual:`，`isEqualToString:` 及 `==` 的区别

* `==`：判断两个对象的内存地址是否相等，相等则返回 YES，不相等则返回 NO；
* `isEqual:` NSObject 及其子类中指定 isEqual: 方法来确定两个对象是否相等。在它的基本实现中，相等检查只是简单地判断相等标识，如下：
    ```
    - (BOOL)isEqual: (id)other {
        return self == other;
    }
    ```
    然而，一些 `NSObject` 的子类重写了 `isEqual:`，因此它们各自重新定义了相等的标准：

    * 如果一个对象最重要的事情是它的状态，那么它被称为值类型，它的 `observable` 属性被用来确定是否相等。

    * 如果一个对象最重要的事情是它的标识，那么它被称为引用类型，它的内存地址被用来确定是否相等。
    
    在 Foundation 框架中，下面这些 `NSObject` 的子类都有自己的相等性检查实现，只要看看它们的 `isEqualToClassName:` 方法就知道了。它们在 `isEqualToClassName:` 中确定是否相等时，相应类型的对象都遵循值语义，当需要对它们的两个实例进行比较时，推荐使用这些高级方法而不是直接使用  `isEqual:` 进行比较。具体类及方法如下：

    * `NSValue -isEqualToValue:`
    * `NSArray -isEqualToArray:`
    * `NSAttributedString -isEqualToAttributedString:`
    * `NSData -isEqualToData:`
    * `NSDate -isEqualToDate:`
    * `NSDictionary -isEqualToDictionary:`
    * `NSHashTable -isEqualToHashTable:`
    * `NSIndexSet -isEqualToIndexSet:`
    * `NSNumber -isEqualToNumber:`
    * `NSOrderedSet -isEqualToOrderedSet:`
    * `NSSet -isEqualToSet:`
    * `NSString -isEqualToString:`
    * `NSTimeZone -isEqualToTimeZone:`
    
    **注意：** `isEqualToClassName:` 方法不接受 `nil` 作为参数，如传 `nil` 编译器会给出警告，而 `isEqual:` 接受(如果传入 `nil` 则返回 `NO` )。

* `isEqualToString:` NSString 是一个很特殊的类型，先看下面代码:
```
NSString *a = @"Hello";
NSString *b = @"Hello";

// YES
if (a == b) {
    NSLog(@"a == b is Yes"); 
}

// YES
if ([a isEqual:b]) {
    NSLog(@"a isEqual b is Yes");
}

// YES
if ([a isEqualToString:b]) {
    NSLog(@"a isEqualToString b is Yes"); 
}
```
会发现上面的三种判断都是 YES ，为什么 `==` 判断也是 YES ？

这是因为苹果采用了 **字符串驻留（String Interning）** 的优化技术。在这种情况下，创建的字符串在内部被视为字符串字面量。运行时不会为这些字符串分配不同的内存空间。
> **注意：** 所有这些针对的都是静态定义的不可变字符串。

另外， `Objective-C` 选择器的名字也是作为驻留字符串储存在一个共享的字符串池当中。对于通过来回传递消息来操作的语言来说，这是一个重要的优化。能够通过指针是否相等来快速检查字符串对运行时性能有很大的影响。

* Tagged Pointers

Tagged Pointer 功能主要有如下三点：
1. Tagged Pointer 用于存储小对象，例如 NSNumber ，NSString 和 NSDate 等；
2. Tagged Pointer 值不再是地址，而是实际值。因此，它不再是真正的对象，它只是一个伪指针，一个 64 位的二进制。因此，它的内存不存储在堆中，不需要 malloc 和 free ；
3. 内存读取效率提高 3 倍，创建速度提高 106 倍；

OS X 和 iOS 都在 64 位代码中使用 Tagged Pointer 对象。在 32 位代码中没有使用 Tagged Pointer 对象，尽管在原则上这并不是不可能。开源的 `objc4-818.2/runtime/objc-internal.h` 有详细的定义及介绍:
```
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,

    // 60-bit reserved
    OBJC_TAG_RESERVED_7        = 7, 

    // 52-bit payloads
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_Photos_2          = 9,
    OBJC_TAG_Photos_3          = 10,
    OBJC_TAG_Photos_4          = 11,
    OBJC_TAG_XPC_1             = 12,
    OBJC_TAG_XPC_2             = 13,
    OBJC_TAG_XPC_3             = 14,
    OBJC_TAG_XPC_4             = 15,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    OBJC_TAG_CGColor           = 18,
    OBJC_TAG_NSIndexSet        = 19,
    OBJC_TAG_NSMethodSignature = 20,
    OBJC_TAG_UTTypeRecord      = 21,

    // When using the split tagged pointer representation
    // (OBJC_SPLIT_TAGGED_POINTERS), this is the first tag where
    // the tag and payload are unobfuscated. All tags from here to
    // OBJC_TAG_Last52BitPayload are unobfuscated. The shared cache
    // builder is able to construct these as long as the low bit is
    // not set (i.e. even-numbered tags).
    OBJC_TAG_FirstUnobfuscatedSplitTag = 136, // 128 + 8, first ext tag with high bit set

    OBJC_TAG_Constant_CFString = 136,

    OBJC_TAG_First60BitPayload = 0, 
    OBJC_TAG_Last60BitPayload  = 6, 
    OBJC_TAG_First52BitPayload = 8, 
    OBJC_TAG_Last52BitPayload  = 263,

    OBJC_TAG_RESERVED_264      = 264
}
```
本质上来说 Tagged Pointer 就是 Tag + Data 组合的一个内存占用 8 个字节 64 位的伪指针：
* Tag 为特殊标记，用于区分是否是 Tagged Pointer 指针以及区分 NSNumber、NSDate、NSString 等对象类型；
* Data 为对象对应存储的值。

在运行效率上，很多涉及 Tagged Pointer 类型相关功能，苹果都有针对性的进行了优化，因此执行起来效率特别高，具体可在源码中搜索 `isTaggedPointer` 进一步查看。

另外，在源码 objc-runtime-new.mm 中有一段注释对 Tagged pointer objects 进行了解释，具体如下：
```
/***********************************************************************
* Tagged pointer objects.
*
* Tagged pointer objects store the class and the object value in the 
* object pointer; the "pointer" does not actually point to anything.
* 
* Tagged pointer objects currently use this representation:
* (LSB)
*  1 bit   set if tagged, clear if ordinary object pointer
*  3 bits  tag index
* 60 bits  payload
* (MSB)
* The tag index defines the object's class. 
* The payload format is defined by the object's class.
*
* If the tag index is 0b111, the tagged pointer object uses an 
* "extended" representation, allowing more classes but with smaller payloads:
* (LSB)
*  1 bit   set if tagged, clear if ordinary object pointer
*  3 bits  0b111
*  8 bits  extended tag index
* 52 bits  payload
* (MSB)
*
* Some architectures reverse the MSB and LSB in these representations.
*
* This representation is subject to change. Representation-agnostic SPI is:
* objc-internal.h for class implementers.
* objc-gdb.h for debuggers.
**********************************************************************/
```
详细介绍此处就不在翻译，重点说一下：
* 1 bit 用来标识是否是 Tagged Pointer；
* 3 bits 用来标识类型；
* 60 bits 负载数据容量 即存储对象数据；

**注意：** 此处不对 Tagged Pointer 实现原理做详细介绍，有兴趣的同学可以 Google 一下 Tagged Pointer，有很多大神介绍的非常详尽。

由于 Tagged Pointer 是一个伪指针，而不是一个真正的对象，因此它并没有 isa 指针。所以当我们调用 Tagged Pointer 对应的 isa 指针时，程序会报错，比如调用 `isKindOfClass`

---
### 总结


---
**最后：期望接下来能再写一篇！**

### 参考资料：

[Nullability Attributes](https://clang.llvm.org/docs/AttributeReference.html#nullability-attributes)

[nil / Nil / NULL / NSNull](https://nshipster.cn/nil/)

[Difference between nullable, __nullable and _Nullable in Objective-C](https://stackoverflow.com/questions/32452889/difference-between-nullable-nullable-and-nullable-in-objective-c)

[Implementing Equality and Hashing](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html)