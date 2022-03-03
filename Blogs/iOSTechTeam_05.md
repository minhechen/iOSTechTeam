# iOS Teach Team iOS中的空 nil, Nil, NULL, NSNull, kCFNulL 及空修饰符 nullable, __nullable, _Nullable, nonnull, __nonnull, __Nonnull 详细解读

### 引言

日常开发过程中，我们经常会碰到空值、空指针、空对象、空的占位对象等。在一些情况下，如果判断不好或者处理方式不对，可能会引起程序运行异常，有些特殊情况甚至会导致 Crash ，因此，熟练了解掌握它们之间的区别，将有助于我们写出更高质量的代码，本文就详细介绍一下它们之间的差别与注意事项。

---
### **常用介绍**

#### 空值相关
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

#### 空值修饰符相关
* nullable
表示对象可以是 NULL 或 nil
2. __nullable
3. _Nullable
4. nonnull
5. __nonnull
表示对象不应该为空
6. __Nonnull



---
### 拓展知识

---
### 总结


---
**最后：期望接下来能再写一篇！**

### 参考资料：

[nil / Nil / NULL / NSNull](https://nshipster.cn/nil/)