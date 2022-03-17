# iOS Teach Team iOS 集成 Flutter module 解析

### 引言

---
* ### 代码规范

---
* ### 本质

---
* ### 常用介绍

iOS 工程下执行 `pod install` 报错如下：

> [!] Invalid `Podfile` file: undefined local variable or method `privacy' for #<Pod::Podfile:0x00007f906719e930>.

这种情况通常都是因为 `install_all_flutter_pods(privacy)` 的放置位置不当引起的，可以尝试放到 `pod ***` 最末尾解决。


---
### 拓展知识

---
### 总结


---
**最后：期望接下来能再写一篇！**

### 参考资料：