# iOS Teach Team iOS分类与扩展详解

### 引言

> 类别允许你即便没有源代码，仍然可以向现有的类中添加方法。类别的功能很强大，它允许你无需子类化而扩展现有类。使用类别，还可以将类的实现分发到多个文件中。类扩展与此类似，但允许在主类@interface 块内以外的位置为类声明额外的必需 API。
---
* ### 代码规范

**Category:**
```
#import "ClassName.h"
 
@interface ClassName (CategoryName)
// method declarations
@end
```

**Extension:**
```
@interface MyClass : NSObject
@property (retain, readonly) float value;
@end
 
// Private extension, typically hidden in the main implementation file.
@interface MyClass ()
@property (retain, readwrite) float value;
@end
```

---
* ### 本质

您可以通过在接口文件中，以类别名称声明它们，并在实现文件中以相同名称定义它们来将方法添加到类。类别名称表明这些方法是对在别处声明的类的添加，而不是一个新类。但是，不能通过类别添加实例变量到类中。

类别添加的方法成为类类型的一部分。例如，在一个类别中添加到NSArray类中的方法，是编译器期望NSArray实例在其配置表中包含的方法。然而，子类中添加到NSArray类中的方法并不包含在NSArray类型中。(这只对静态类型的对象有影响，因为静态类型是编译器知道对象类的唯一方式。)

---
* ### 常用介绍

**分类：**

分类在经历过编译后，分类里面的内容：对象方法、类方法、协议、属性都转化为类型为category_t的结构体变量：
```
struct category_t {
    const char *name;
    classref_t cls;
    WrappedPtr<method_list_t, PtrauthStrip> instanceMethods;
    WrappedPtr<method_list_t, PtrauthStrip> classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};
```
具体Category都能做什么，常用的大致有如下几个场景：
1. 在不修改原有类的基础上给原有类添加方法，因为分类的结构体指针中没有属性列表，只有方法列表。所以原则来说只能给分类添加方法，不能添加属性，如果需要给分类添加类似属性功能，可以通过关联对象实现；
2. 分类中的方法优先于原有类同名的方法, 即会优先调用分类中的方法, 忽略原有类的方法。即分类与原有类同名方法调用的优先级为： 分类 > 本类 > 父类。开发中尽量不要覆盖本类的方法，如果覆盖会导致本类方法失效；
3. 如果给Category添加@property属性，只会生成setter和getter方法的声明，并不会有具体的代码实现，详细解释可参考历史文章：[iOS探究属性@property](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/iOSTechTeam_01.md)
4. 分类中可以访问原有类中 `.h` 中声明的成员变量；


---
**类的扩展Extension：**

```
@interface Person ()

@end
```
类的Extension看起来很像一个匿名的Category。通常用来声明私有方法，私有属性和私有成员变量。

> extension 在编译期决议， category在运行期决议。

类扩展不能像类别 `Category` 那样拥有独立的实现部分（@implementation部分），也就是说，类的扩展所声明的方法必须依托原类的实现代码部分来实现。

因此，我们不能给系统类添加类扩展。即扩展的方法只能在原类中实现。例如我们扩展NSString，那么只能在 `NSString的.m` 中实现，但我们拿不到 `NSString的.m` 的源码，因此，我们不能给 `NSString` 添加扩展，只能给 `NSString` 添加分类；

定义在 `.m` 文件中的类扩展方法为私有的，如果需要声明私有方法，这种方式特别合适，定义在 .h 文件（头文件）中的类扩展方法为公有的。

---
### **分类Category与扩展Extension的区别**

1. 分类有名字，扩展没有名字，像是一个匿名的分类
2. 分类是运行时决议，而扩展是编译时决议；所以分类中的方法没有实现不会警告，而扩展声明的方法不实现会出现警告。
3. 分类原则上可以增加属性，实例方法，类方法，而且外部类是可以访问的。扩展能添加属性，方法，实例变量，默认是不对外公开的。
4. 分类有自己的实现部分，扩展没有自己的实现部分，只能依赖类本身来实现。
5. 可以为系统类添加分类，而不能为系统类添加扩展。

---
### **+ (void)load与+ (void)initialize区别**
> ```+ (void)load```
>
> ```+ (void)initialize```

* 两者的区别如下：
1. **相同点：**
> * 两个函数都是系统自动调用，因此无需手动调用（如果手动调用则与普通函数调用类似）；
> * 两个函数都会隐士调用各自父类对应的 `+ (void)load` 或 `+ (void)initialize` 方法，即子类调用方法之前，会优先调用其父类对应的方法；
> * 两个函数内部都使用了锁，因此两个函数都是线程安全的；

2. **不同点：**
> 1. 调用时机不同：`+ (void)load` 在 `main` 函数之前执行，即 `objc_init` Runtime初始化时调用，且只会调用一次, `+ (void)initialize` 在类的方法首次被调用时执行，每个类只会调用一次，但父类可能会调用多次；
> 2. 调用方式不同：`+ (void)load` 是根据函数地址直接调用，`+ initialize` 是通过消息发送机制即 `objc_msgSend(id self, SEL _cmd, ...)` 调用；
> 3. 子类父类调用关系不同：
> > * 如果子类没有实现 `+ (void)load`，则不会调用其父类的 `+ (void)load` 方法。
> > * 如果子类没有实现 `+ (void)initialize`，则会调用其父类的方法，因此父类的 `+ (void)initialize` 可能会调用多次；
> 4. 分类 `Category` 对调用的影响不同：
> > * 如果 `Category` 中实现了 `+ (void)load`，则会优先调用原类的的 `+ (void)load`，再调用分类的，即优先级为：父类 > 原类 > 分类
> > > 1. 没有继承关系的不同类中的 `+ (void)load` 的调用顺序跟Compile Sources顺序有关，即在前面的优先编译的类或者分类先调用（ **备注：** 所有类的 `+ (void)load` 优先级大于分类的优先级）；
> > > 2. 同一个类的分类 `+ (void)load` 的调用顺序跟Compile Sources顺序有关，即在前面的优先编译的分类会先调用；
> > > 3. 同一镜像中主工程的 `+ (void)load` 方法优先调用，然后再调用静态库的 `+ (void)load` 方法。有多个静态库时，静态库之间的执行顺序与编译顺序有关，即它们在Link Binary With Libraries中的顺序；
> > > 4. 不同镜像中，动态库的 `+ (void)load` 方法优先调用，然后再调用主工程的 `+ (void)load`，多个动态库的 `+ (void)load` 方法的调用顺序跟编译顺序有关，即它们在Link Binary With Libraries中的顺序；

> > * 如果 `Category` 中实现了 `+ (void)initialize`，则原类的 `+ (void)initialize` 将不会再调用
> > > 1. 多个 `Category` 中同时实现了 `+ (void)initialize` 方法时，Compile Sources中顺序最下面的一个，即最后一个被编译分类的 `+ (void)initialize` 会执行；

### 分类 `Category` 中添加关联对象

`Category` 中添加属性 `@property` 在前文已做过简单介绍，具体可查看 [iOS探究属性@property](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/iOSTechTeam_01.md)，这里我们重点说一下关联对象的实现原理：

操作关联对象有三个核心方法：
1. 设置关联对象方法：
> objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key, id _Nullable value, objc_AssociationPolicy policy)
> > * 1. id _Nonnull object: 给哪个对象添加关联对象，通常是当前对象，即用 `self` 即可；
> > * 2. const void * _Nonnull key: 关联对象的 `key`，作为管理对象的唯一标识存在，它只要是一个非空指针即可；
> > * 3. id _Nullable value: 关联对象的值，通过关联 `key` 进行设值及获取值，如果需要清除一个已存在的关联对象，将其值设置为 `nil` 即可；
> > * 4. objc_AssociationPolicy policy: 关联策略，即关联对象的存储形式，其可选枚举值如下：
``` 
public enum objc_AssociationPolicy : UInt {
    case OBJC_ASSOCIATION_ASSIGN = 0 // 指定对关联对象的弱引用
    case OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1 // 指定对关联对象的强引用，非原子性
    case OBJC_ASSOCIATION_COPY_NONATOMIC = 3 // 指定复制关联的对象，非原子性
    case OBJC_ASSOCIATION_RETAIN = 769 // 指定对关联对象的强引用，原子性
    case OBJC_ASSOCIATION_COPY = 771 // 指定复制关联的对象，原子性
} 
```
根据源码，我们可以知道 `objc_setAssociatedObject` 实际调用的是 `_object_set_associative_reference`:
```
void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    // This code used to work when nil was passed for object and key. Some code
    // probably relies on that to not crash. Check and handle it explicitly.
    // rdar://problem/44094390
    if (!object && !value) return;

    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));

    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value};

    // retain the new value (if any) outside the lock.
    association.acquireValue();

    bool isFirstAssociation = false;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());

        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
            if (refs_result.second) {
                /* it's the first association we make */
                isFirstAssociation = true;
            }

            /* establish or replace the association */
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            auto refs_it = associations.find(disguised);
            if (refs_it != associations.end()) {
                auto &refs = refs_it->second;
                auto it = refs.find(key);
                if (it != refs.end()) {
                    association.swap(it->second);
                    refs.erase(it);
                    if (refs.size() == 0) {
                        associations.erase(refs_it);

                    }
                }
            }
        }
    }

    // Call setHasAssociatedObjects outside the lock, since this
    // will call the object's _noteAssociatedObjects method if it
    // has one, and this may trigger +initialize which might do
    // arbitrary stuff, including setting more associated objects.
    if (isFirstAssociation)
        object->setHasAssociatedObjects();

    // release the old value (outside of the lock).
    association.releaseHeldValue();
}
```
根据上述源码可以发现，`ObjcAssociation` 根据 传入的 `value` 及 `policy` 创建对象，并经过 `acquireValue` 函数处理生成新的 `_value` 。`acquireValue` 函数内部是通过对策略 `policy` 的判断进行相应处理，生成新值，其实现如下：
```
inline void acquireValue() {
    if (_value) {
        switch (_policy & 0xFF) {
        case OBJC_ASSOCIATION_SETTER_RETAIN:
            _value = objc_retain(_value);
            break;
        case OBJC_ASSOCIATION_SETTER_COPY:
            _value = ((id(*)(id, SEL))objc_msgSend)(_value, @selector(copy));
            break;
        }
    }
}
```

接下来我们首先需要了解一下 `AssociationsManager` 和 `AssociationsHashMap`
```
class AssociationsManager {
    using Storage = ExplicitInitDenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap>;
    static Storage _mapStorage;

public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }

    AssociationsHashMap &get() {
        return _mapStorage.get();
    }

    static void init() {
        _mapStorage.init();
    }
};
```
由源码可以知道，`AssociationsManager` 是以 `DisguisedPtr<objc_object>` 即一个指针地址作为 `key`，以 `ObjectAssociationMap` 即一个关联表作为 `value` 的哈希表来使用的。其内部是使用一个全局静态变量 `static Storage _mapStorage` 来存储程序中所有的关联对象。

这里重点介绍一下全局静态变量 `static Storage _mapStorage` 的初始化时机，App启动过程中，在 `_objc_init` 函数中会调用 `void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped  unmapped)`，具体如下：

```
void _objc_init(void)
{
    //...
    // 此处仅保留谈到的函数
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
    //...
}
```
在 `dyld` 源码中可以看到，函数 `_dyld_objc_notify_register` 中的三个参数为三个回调函数的指针，如下图：

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/dyld_objc_notify_register.png)

回调函数会在所有镜像文件初始化完成之后，回调 `map_images(unsigned count, const char * const paths[], const struct mach_header * const mhdrs[])` 函数，详细调用流程如下图：


**备注：图中已对无关代码进行删减，仅用来展示调用流程**
![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/objc_associations_init.png)

从上图我们就知道了，在App启动过程中 `AssociationsManager` 中的静态变量 `static Storage _mapStorage` 的初始化时机。在App启动之后，所有用到关联对象的地方，程序都是从这个全局静态变量 `_mapStorage` 中获取 `AssociationsHashMap` 来对关联对象进行进一步处理。

在 `AssociationsManager` 中，我们可以看到是由一个 `AssociationsManagerLock` 叫做 `spinlock_t` 的互斥锁：

> using spinlock_t = mutex_tt<LOCKDEBUG>;

它是用来保障 `AssociationsManager` 中对 `AssociationsHashMap` 操作的线程安全。

```
AssociationsHashMap &get() {
    return _mapStorage.get();
}
```

对于 `AssociationsHashMap` 这个哈希表，则是由全局静态变量 `_mapStorage` 获取而来，因此不管任何时候操作关联对象，程序始终都是在操作这个 `AssociationsHashMap` 全局唯一的哈希表。

**再回到上面 `_object_set_associative_reference` 源码中**，当我们添加一个关联对象时，`AssociationsHashMap` 会调用如下函数：
```
auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
```
`try_emplace` 函数的源码如下：

```
  template <typename... Ts>
  std::pair<iterator, bool> try_emplace(const KeyT &Key, Ts &&... Args) {
    BucketT *TheBucket;
    if (LookupBucketFor(Key, TheBucket))
      return std::make_pair(
               makeIterator(TheBucket, getBucketsEnd(), true),
               false); // Already in map.

    // Otherwise, insert the new element.
    TheBucket = InsertIntoBucket(TheBucket, Key, std::forward<Ts>(Args)...);
    return std::make_pair(
             makeIterator(TheBucket, getBucketsEnd(), true),
             true);
  }
```

首先根据传来的 `key` 即 `disguised` 在 `AssociationsHashMap` 中查找对应的 `ObjectAssociationMap` 是否已在映射表中，如果不在则将键插入，如果键不在，则创建一个 `BucketT` 即一个空的桶。在第二次调用 `try_emplace` 时将 `ObjcAssociation` （里面包含了 `_policy` 和 `_value` ）存储到这个 `BucketT` 空桶中。

当设置的关联 `value` 为空的时候会进入 `if` 判断的 `else` 里面：
```
auto refs_it = associations.find(disguised);
if (refs_it != associations.end()) {
    auto &refs = refs_it->second;
    auto it = refs.find(key);
    if (it != refs.end()) {
        association.swap(it->second);
        refs.erase(it);
        if (refs.size() == 0) {
            associations.erase(refs_it);

        }
    }
}
```
先去 `AssociationsHashMap` 里面查找 `disguised` ，如果找到则根据 `key` 查找到指定的关联对象，然后进行清除 `erase` 操作。之后判断当前 `object` 的关联对象是否为0，如果为0，则将当前关联对象从全局的 `AssociationsHashMap` 中移除。

2. 获取关联对象方法：
> objc_getAssociatedObject(id _Nonnull object, const void * _Nonnull key)
> > * 1. id _Nonnull object: 获取哪个对象里面的关联对象；
> > * 2. const void * _Nonnull key: 关联对象的 `key` ，与 `objc_setAssociatedObject` 中的 `key` 相对应，通过 `key` 值取出 `value` 即关联对象；

其内部调用的是 `_object_get_associative_reference` 内部具体实现如下：

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/object_get_associative_reference.png)

如果我们理解了设置关联对象的过程，上面的代码理解起来就比较简单了，从全局的 `AssociationsHashMap` 中取得 `object` 对象对应的 `ObjectAssociationMap` ，然后根据 `key` 从 `ObjectAssociationMap` 获取对应的 `ObjcAssociation` ，然后根据关联策略 `_policy` 判断是否需要对 `_value` 执行 `retain` 操作，最后根据关联策略 `_policy` 判断是否需要将 `_value` 添加到自动释放池，并返回 `_value`。

3. 移除关联对象：
上面已经提到，如果想要清除某一个特定关联对象，设置关联对象的 `value` 为 `nil` 即可。如果想要移除所有关联对象，则可以使用：
> objc_removeAssociatedObjects(id _Nonnull object)
> > * 1. id _Nonnull object: 移除给定对象的所有关联对象

其内部实现代码如下：

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/objc_removeAssociatedObjects.png)

当调用移除关联对象操作时，会先判断 `object` 是否为空及是否有关联对象存在，如果存储则会调用 `_object_remove_assocations` 函数。

从上图其内部实现代码可以看到，程序会获取全局的 `AssociationsHashMap` 然后从中获取对象对应的 `ObjectAssociationMap` ，注释说如果不是 `deallocating`，则系统的关联对象将会保留。而 `objc_removeAssociatedObjects` 函数传入的 `deallocating` 参数为 `false`，因此我们可以推断，解除关联必定不是在调用 `objc_removeAssociatedObjects` 时。

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/objc_destructInstance.png)

于是，我搜索了一下 `_object_remove_assocations`，发现了真正的调用时机，即在 `objc_destructInstance` 函数调用时，如上图。

那什么时候会调用 `objc_destructInstance` 函数呢？带着这个疑问，我查了一下源码，这里简单说一下调用流程，后续会专门针对 `dealloc` 写相关文章，其大体流程如下：

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/dealloc.png)
图中函数调用流程非常清晰，此处不做过多解释，所以，我们知道解除关联对象是在源对象 `dealloc` 时进行的。


---
### 拓展知识
1. iOS中变量修饰词@public、@protected、@package、@private的作用：
> @package // 常用于框架类的实例变量，使用@private太限制，使用@protected或者@public又太开放，这时可以使用@pakage
>
> @private // 作用范围只能在自身类，即使子类也无法使用，但分类及扩展类中可以使用
>
> @protected // 系统默认为@protected，作用范围在自身类及子类
>
> @public // 公开类型，作用域大，只要能拿到所属实例对象就可以使用
>

实例变量范围图（`@package` 的范围图中未展示）

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/scopeinstancevariables.jpg)

```
@interface Person : NSObject {
@package
    NSString *_country; // 框架内拿到Person及其子类的实例变量都可以使用

@protected
    NSString *_birthday; // 只能在自身类及子类中使用，包括分类及扩展

@private
    NSString *_weight; // 只能在自身类中使用，包括分类及扩展

@public
    NSString *_height; // 全局任意拿到Person及其子类实例变量的地方都可以使用
}
```
具体实例如下： `Son` 继承自 `Person`：
```
@interface Son : Person

@end
```
![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/son_test_code_error.png)
@protected
从上图示例代码可以看到，在子类中是可以访问父类的 `@protected _birthday` 成员变量，但不能访问父类的 `@private _weight` 成员变量。

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_03/view_controller_code_error.png)
从上图示例代码可以看到，在其他类中是可以访问父类的 `@protected _birthday` 成员变量，但不能访问父类的 `@private _weight` 成员变量。

---
### 总结

分类 `Category` 和扩展 `Extension` 涉及到的东西还是挺多的，这里仅对其核心要关注的一些点进行了详细介绍，另外还有关于 `Category` 装载的过程，希望对你我能有所帮助，感谢阅读。

---
### **参考资料：**

* [Categories and Extensions](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocCategories.html)

* [Defining a Class](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocDefiningClasses.html#//apple_ref/doc/uid/TP30001163-CH12-SW1)

* [Associated Objects](https://nshipster.com/associated-objects/)

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系交流，感谢阅读。