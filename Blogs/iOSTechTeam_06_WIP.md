# iOS Teach Team iOS ==， isEqual:，isEqualToString的别详细解读

### 引言

---
* ### 代码规范

---
* ### 本质

---
* ### 常用介绍

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
    
    **注意：** `isEqualToClassName:` 方法不接受 `nil` 作为参数，如传 `nil` 编译器会给出警告，而 `isEqual:` 接受（如果传入 `nil` 则返回 `NO` )。

* `isEqualToString:` NSString 是一个很特殊的类型，先看下面代码：
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

另外， `Objective-C` 选择器的名字也是作为驻留字符串储存在一个共享的字符串池当中。对于通过来回传递消息来操作的语言来说，这是一个重要的优化。能够通过指针是否相等来快速检查字符串，这对运行时性能有很大的影响。


* Hashing

对于面向对象编程来说，对象相等性检查的主要用例，就是确定一个对象是不是一个集合的成员。 为了确保在 NSDictionary 和 NSSet 集合中的检查速度，具有自定义相等实现的子类应该以满足以下条件的方式实现 hash 方法：

1. 对象相等具有**交换性**：（[a isEqual:b] ⇒ [b isEqual:a]）
2. 如果对象相等，那么它们的哈希值也必须相等：（[a isEqual:b] ⇒ [a hash] == [b hash]）
3. 但是，反过来则不成立：即两个对象可以具有相同的哈希值，但彼此不相等：（[a hash] == [b hash] ¬⇒ [a isEqual:b]）

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
详细介绍此处就不再翻译，重点说一下：
* 1 bit 用来标识是否是 Tagged Pointer；
* 3 bits 用来标识类型；
* 60 bits 负载数据容量 即存储对象数据；

**注意：** 此处不对 Tagged Pointer 做深入详细介绍，有兴趣的同学可以 Google 一下 Tagged Pointer，有很多优秀的文章介绍的非常详尽。

由于 Tagged Pointer 是一个伪指针，而不是一个真正的对象，因此它并没有 isa 指针。所以当我们通过 LLDB 打印 Tagged Pointer 对应的 `isa` 指针时，程序会报错误提示：
> error: Couldn't apply expression side effects : Couldn't dematerialize a result variable: couldn't read its memory

而当针对 Tagged Pointer 需要使用到类似 Objecttive-C 对象的 isa 指针功能时，可以通过调用 `isKindOfClass` 和 `object_getClass` 实现判断及其他操作。

---
### 拓展知识

---
### 总结


---

### 参考资料：

[Implementing Equality and Hashing](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html)

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系进群学习交流，感谢阅读。