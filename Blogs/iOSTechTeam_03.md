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
4. 

---
### 拓展知识

---
### 总结


---
**最后：？！**

### 参考资料：

[Categories and Extensions](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocCategories.html)