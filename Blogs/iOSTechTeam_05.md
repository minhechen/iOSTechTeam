# iOS Teach Team iOS 依赖管理工具 CocoaPods、 Carthage 及 Git submodules 的使用及原理

### **引言**
iOS Dependency manager 即 iOS 依赖管理器，几乎每个项目都会使用第三方库或者框架，也有许多管理依赖项的方法。有些人主张自己实现每个功能，而另一些人则选择使用现有的库。当然，没有一个答案是最好的，真相介于两者之间，适合自己的才是最好的。但无论是选用第三方库、框架，还是选择自己重新造轮子，都离不开代码的依赖管理，这篇文章我们就介绍一下 iOS 常用的几种依赖管理工具及其实现原理。

---
### **常用的三种依赖管理器优缺点对比**

1. CocoaPods：

优点：
* 使用方便，除了编写 Podfile 之外基本都是命令行完成；
* 支持 iOS 8 及以上的 framework，当然也支持旧的静态编译（现在很少App支持iOS 8了）；
* 软件包数量比较多，主流支持；

缺点：
* 每个默认更新都要更新 podspec 仓库，很耗时；
* 开发者使用比较简单，但是如果创建兼容 CocoaPods 的库，就会相对繁琐一些（尽管有了命令行）；
* 每次 clean 之后编译，都要重新编译所有第三方库；


2. Carthage：

优点：
* 使用 Carthage 的话，所有的第三方库依赖，除非是更新的需要，不然平常干净编译 Project，它是不需要再次编译的，大大加快平常编译及 Archive 的时间；
* 它是去中心化的，没有中心服务器. 这意味着每次配置和更新环境，只会去更新具体的库，而不会有一个向中心服务器获取最新库的索引这么个过程，如此又省了很多时间；
*  CocoaPods 无缝集成！一个项目可同时使用两套包管理工具, 当前 CocoaPods 管理主要 Framework 的配置下, 将少量其他 Framework 交给了 Carthage 管理, 二者可以和谐地共存；
* 结构标准的项目天然就是 Carthage 库；

缺点：
* 库依然不如 CocoaPods 丰富：尽管很多库不需要声明并改造就直接可以被 Carthage 用，但依然有大量 CocoaPods 能用的库不支持；
* 只支持 Framework ，iOS 8 以上（现在已很少有App支持iOS 8了）；
* 无法在 Xcode 里定位到源码，无法进行断点调试：如果你在写代码过程中，想跳转到一个第三方库去看具体的实现，这是无法办到的，Carthage 的配置只能让你看到一个库的头文件；


3. Git Sumboudles：

优点：
* 允许开发者将一个子项目的 git 仓库作为主项目 git 仓库的子项目，并让子项目保持提交的独立；
* 子项目无需多余操作，只需要在主项目指向子项目相应的 commitId 即可，这个即是优点，也是缺点；

缺点：
* 主项目仓库不记录子项目仓库的文件变动，只记录子项目仓库的commitId，在这个背景下，团队协作开发过程中容易产生子项目版本不一致的问题；
* 每次更新子项目仓库，子项目仓库都会回到游离状态；
* 子项目仓库总是需要手动更新(可以通过自动化工具编写帮助解决此问题)；


---
* ### CocoaPods 使用及原理


---
* ### Carthage 使用及原理

---
* ### Git Sumboudles 使用及原理


---
### 拓展知识
* 1. 什么是库，什么是框架？
库是指具有特定功能实现的代码库，例如：`SnapKit` 帮助我们实现自动布局，或像 `lottie-ios` 帮助我们实现自定义动画的动画库。

---
### 总结


---
**最后：期望接下来能再写一篇！**

### 参考资料：

[Differences between Carthage and CocoaPods](https://github.com/Carthage/Carthage#differences-between-carthage-and-cocoapods)
[Mastering Git submodules](https://medium.com/@porteneuve/mastering-git-submodules-34c65e940407)

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系交流，感谢阅读。