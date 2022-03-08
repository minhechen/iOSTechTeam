# iOS Teach Team iOS instancetype id __kindof __covariant关键词详解

### 引言

---
* ### 代码规范

---
* ### 本质

---
* ### 常用介绍
1. instancetype 关键词作用

 `instancetype` 是 `clang` 提供的一个关键词，从 `clang 3.5` 版本开始使用。它与 `id` 一样，表示某个方法返回的未知类型的 `Objective-C` 对象。它的作用是使那些非关联返回类型的方法返回所在类的类型。

 这里解释一下关联返回类型和非关联返回类型：
 * 关联返回类型


 
2. __kindof 关键词作用
类型延拓符 __kindof

3. __covariant 及 __contravariant 关键词作用
协变 __covariant : 用于数据强制转换类型，子类型指针可以向父类型指针转换

逆变 __contravariant: 用于数据强制转换类型，父类型指针可以向子类型转换


4. ObjectType 通配符的使用
---
### 拓展知识

---
### 总结


---
**最后：期望接下来能再写一篇！**

### 参考资料：

[instancetype](https://nshipster.com/instancetype/)

[Adopting Modern Objective-C](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html)