# iOS Teach Team iOS探究属性@property

### 引言

> @property应该是面试过程中被问到最多的一个知识了，既能考察一个人的基础，又能挖掘一个人对知识细节的熟练度，本文着重全面，细致的介绍一下@property都有哪些知识点值得我们关注。

* ### 代码规范

声明@property时，注意关键词及字符间的空格，规范虽然对功能没有直接影响，但一个人，一个群体的规范性会提升整个行业的基本素养。

> @property (nonatomic, copy) NSString *name;

* ### 本质
@property的默认本质其实是：ivar(实例变量) + getter + setter；

* ### 常用关键词
```
// 存取器方法
1. getter=getterName
2. setter=setterName

// 读写权限
1. readonly
2. readwrite

// 内存管理
1. strong
2. assign
3. copy
4. weak
5. retain
6. unsafe_unretained

// 原子性
1. nonatomic
2. atomic
```
---
接下来逐个介绍一下，每个关键词的作用:

#### **存取器方法**
> 1. getter=getterName 
> 2. setter=setterName:

指定获取属性对象的名字为"getterName"，如果你没有使用`getter`指定getterName，系统默认直接使用propertyName访问即可。通常来说，只有所指属性需要我们指定isPropertyName对应的Bool值时，才使用指定getterName，一般直接用PropertyName即可。`setter=setterName:`则是用来指定设置属性所使用的的setter方法，即设置属性值时使用`setterName:`方法，此处setterName是一个方法名，因此要以":"结尾，具体示例如下：
```
// 指定getter访问名为`isHappy`
@property (nonatomic, assign, getter=isHappy) BOOL happy;

// 使用系统默认getter/setter方法
@property (nonatomic, assign) NSInteger age;

// 指定setter方法名为`setNickName:`
@property (nonatomic, copy, setter=setNickName:) NSString *name;

```

---
#### **读写权限**
> 1. readwrite
> 2. readonly

* readwrite
表示自动生成对应的getter和setter方法，即可读可写权限，readwrite是编译器的默认选项。
* readonly
表示只生成getter，不需要生成setter，即只可读，不可以修改。

---
#### **内存管理**
> 1. strong             // 强引用，引用计数+1
> 2. assign             // assign是指针赋值，不对引用计数操作，对象销毁后不会自动置为nil
> 3. copy               // copy出一个新对象，引用计数为1
> 4. weak               // 弱引用，不对引用计数操作，对象销毁时自动置为nil
> 5. retain             // 强引用，对象引用计数+1
> 6. unsafe_unretained  // 弱引用，不对引用计数操作，对象销毁时不会自动置为nil

* **strong**

表示强引用关系，即修饰对象的引用计数会+1，通常用来修饰对象类型，可变集合及可变字符串类型，当对象引用计数为0，即不被任何对象持有，并且此对象不再显示在列表中，则此对象就会从内存中释放。

* **assign**

对象不进行retain操作，即不改变对象引用计数。通常用来修饰基本数据类型（NSInteger, CGFloat, Bool, NSTimeInterval等），内存在栈上由系统自动回收。

assign也可以用来修饰NSObject类型对象，因为assign不会改变修饰对应的引用计数，所以当修饰对象的引用计数为0，对象销毁的时候，对象指针不会被自动清空，而此时对象指针指向的地址已被销毁，这时再访问该属性会产生野指针错误`:EXC_BAD_ACCESS`，因此assign通常用来修饰基本数据类型。

* **copy**

当调用修饰对象的setter方法时，会建立一个引用计数为1的新对象，然后释放旧对象，即对象会在内存里拷贝一份副本，两个指针指向不同的内存地址。一般用于修饰字符串（NSString）和集合类(NSArray, NSDictionary)的不可变变量，Block也是用copy修饰。

针对copy，这里又牵涉到了深copy和浅copy的问题，这里做一下简单介绍，后续会有文章专门探讨这个问题:
> 浅copy：是对指针的copy，指针指向的内容是同一个地址，对象的引用计算+1;<br>
> 深copy：是对内容的copy，会开辟新的内存空间，将内容重新copy一份；

* 非集合对象的copy与mutableCopy:
> 不可变对象：copy操作为浅copy，mutableCopy操作为深copy。<br>
> 可变对象：copy操作为深copy，mutableCopy操作也为深copy。

* 集合类对象的copy与mutableCopy:
> 不可变对象：copy操作为深copy，mutableCopy操作为深copy。<br>
> 可变对象：copy操作为深copy，可变对象的mutableCopy也为深copy。

注意：当使用copy修饰属性赋值时，copy出来的是一份不可变对象。因此当对象是一个可变对象时，切记不要使用copy进行修饰，如果这时使用copy修饰，对象在使用可变对象所特有的方法时，会因为找不到对应的方法而Crash。

* **weak**

表示弱引用关系，修饰对象的引用计数不会增加，当修饰对象被销毁的时候，对象指针会自动置为nil，防止出现野指针。weak也用来修饰delegate，避免循环引用。weak只能用来修饰对象类型，且是在ARC下新引入的修饰词，MRC下使用assign。

**weak的底层实现原理**
weak的底层实现是基于Runtime底层维护的SideTables的hash数组，里面存储的是一个SideTable的数据结构：
> spinlock_t slock; // 确保原子性操作的锁，虽然名字还叫spinlock_t，其实本质已经是mutex_t，具体可见objc-os源码（`typedef mutex_t spinlock_t;`）<br>
>
> RefcountMap refcnts; // 用来存储对象引用计数的hash表<br>
> weak_table_t weak_table // 存储对象weak引用指针的hash表;

* weak功能实现核心的数据结构weak_table_t：
```
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries; // 存储weak对象信息的hash数组
    size_t    num_entries; // 数组中元素的个数
    uintptr_t mask;        // 计数辅助量
    uintptr_t max_hash_displacement; // hash元素最大偏移值
};
```

* weak_entry_t也是一个hash结构
```
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        // 动态数组
        struct {
            weak_referrer_t *referrers; // 弱引用该对象的对象指针的hash数组
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        // 定长数组，最大值为4，苹果考虑到一半弱引用的指针个数不会超过这个数，因此为了提升运行效率，一次分配一整块的连续内存空间
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    // 判断当前是动态数组，还是定长数组
    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```
这里重点说一下weak_entry_t定长数组-->动态数组的切换，会先将原来定长数组中的内容转移到动态数组中，然后再在动态数组中插入新的元素。

而对于动态数组中元素个数大于或等于总空间的3/4时，则会对动态数组进行总空间 * 2的扩容
```
    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        return grow_refs_and_insert(entry, new_referrer);
    }
```
每次动态数组扩容，都会将原先数组中的内容重新插入到新的数组中。

当对象的引用计数为0时，底层会调用_objc_rootDealloc方法对对象进行释放，而在_objc_rootDealloc方法里面会调用rootDealloc方法，如果对象有被weak引用，则会进入`object_dispose`, 之后会在`objc_destructInstance`函数里面调用`obj->clearDeallocating();`根据对象地址获取所有weak指针地址数组，遍历数组找到对应的值，将其值为nil，然后将entry从weak表中移除，最后从引用计数表中删除以废弃对象的地址为键值的记录。

**备注：** 此处省略了weak底层实现的很多细节，具体详细实现，后续会单独发文介绍。

* retain是在MRC下常用的修饰词：

ARC下已不再使用retain，而是使用strong。retain同strong类似，用来修饰对象类型，强引用对象，其修饰对象的引用计数会+1，不会对对象分配新的内存空间。

* unsafe_unretained同weak类似：

unsafe_unretained不会对对象的引用计数+1，只能用来修饰对象类型，修饰的对象在被销毁的时，其指针不会自动清空，指向的仍然是已销毁的对象，这时再调用该指针时会产生野指针错误`:EXC_BAD_ACCESS`

---
### **原子性**
> atomic原子性：系统会自动给生成的getter/setter方法会进行加锁操作
> nonatomic非原子性：系统自动生成的getter/setter方法不会进行加锁操作

设置属性函数`static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)`的原子性非原子性实现如下：
```
    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }
```
获取属性函数`id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic)`的内部实现如下：
```
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
```

由此可见，对属性对象的加锁操作仅限于对象的get/set操作，如果是get/set以外的操作，该加锁并没有意义，因此atomic的原子性，仅能保障的是对象的get/set的线程安全，并不能保障多线程下对对象的其他操作安全，如一个线程在get/set操作，一个线程进行release操作，可能会导致crash。此种场景的线程安全，还需要由开发者自己进行处理。

---
### 拓展知识

* Category中添加属性@property

在Category中添加@property，只会生成setter和getter方法的声明，并不会有具体的代码实现，具体原因是因为Category在运行期，对象的内存布局已经确定，此时如果添加实例变量就会破坏对象的内存布局，这将会是灾难性的，因此Category无法添加实例变量。那如何给Category实现类似实例变量功能呢，此处仅列举两种方式，因为可讲内容过多，因此此处暂时不做具体详解，后续会有文章单独介绍：
1. 使用临时全局变量代替成员变量，并在属性的set/get函数中进行存取值操作；
2. 通过Runtime添加管理对象实现成员变量，并在属性的set/get函数中进行存取值操作，关键调用有两个：
> objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key, id _Nullable value, objc_AssociationPolicy policy) // 设置关联对象值调用
>
> objc_getAssociatedObject(id _Nonnull object, const void * _Nonnull key) // 获取关联对象值调用

* Protocol中添加属性@property

在protocol中添加属性@property，其实就是声明该属性的setter/getter方法，在实现该protocol时，并没有生成对应的成语变量，此时有动态实现和自动实现两种方式：
1. 动态实现，需要在实现类中添加protocol中声明属性对应的setter/getter方法，并声明一个私有成语变量用来进行存取操作。**备注：** 实现类如果没有实现对应的setter/getter方法，在调用protocol属性的setter/getter方法是时会引起`Crash`
2. 自动实现，可以通过`@synthesize propertyName;`告诉编译器自动添加对应的setter/getter方法，并生成对应的成员变量；

* @synthesize作用

`@synthesize`的作用是告诉编译器，自动创建属性的setter/getter方法，同时生成成员变量，并且可以给属性指定别名，如：
>`@synthesize name = nickName;`

* @dynamic作用

`@dynamic`作用是告诉编译器，无需自动创建属性的setter/getter方法，这个时候就需要我们工程师自己手动实现相应的setter/getter，否则，在使用到相应属性的setter/getter方法时，会因找不到相应方法而Crash。

* null相关的一些关键词
```
nullable // 可以为空
nonnull // 不可以为空
null_unspecified // 未知类型
null_resettable // get不能为空，set可以为空
__nullable // 可以为Null或nil
__nonnull // 不可以为空
```
* Swift下的unowned与weak区别：

Swift下weak的使用同Objective-C。
unowned标记对象，即使它的原来引用已经被释放，它仍然会保持对被已经释放了的对象的一个 "无效的" 引用，它不是 Optional ，也不会被指向 nil。所以，当我们试图访问这样的 unowned 引用时，程序就会发生错误。而weak在引用的内容被释放后，标记为 weak 的成员将会自动地置为 nil。
> Define a capture in a closure as an unowned reference when the closure and the instance it captures will always refer to each other, and will always be deallocated at the same time.
>
>Conversely, define a capture as a weak reference when the captured reference may become nil at some point in the future. Weak references are always of an optional type, and automatically become nil when the instance they reference is deallocated. This enables you to check for their existence within the closure’s body.
>
> **NOTE:**
>
> **If the captured reference will never become nil, it should always be captured as an unowned reference, rather than a weak reference.**

根据苹果的官方文档的建议。如果捕获的引用永远不会变为 nil，我们应该使用 unowned，否则应该使用weak。

---
### 总结
@property延展相关的知识可以聊很多，如：copy相关的NSCopying协议，weak底层详细的实现原理，如何保障对象的多线程安全，还有很多知识点都跟Runtime、Runloop有关等等。

知识点完整说下来就是一整套系统的协同运转，各个环节紧密相扣，最终才成为我们现在看到的样子。本文及以后的文章都会尽可能的收缩一下单片文章探讨的范围，以期能够让话题更加紧密。

---
**最后：期望接下来能再写一篇！**


### 参考资料：
[The Objective-C Programming Language](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocProperties.html#//apple_ref/doc/uid/TP30001163-CH17-SW2)