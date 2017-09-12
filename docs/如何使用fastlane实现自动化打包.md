软件项目的自动化是提高工作效率非常重要的一个环节，包括自动化编译打包、自动化发布、自动化测试等等。本文以`Xcode 8`为基础，主要讲如何使用`fastlane`实现 iOS 编译打包发布等，打包服务器的（`Jenkins`）部署不详细说明了。

### Jenkins 安装
假定你已经有了一台本地服务器，首先需要在服务器上安装自动化服务[Jenkins](https://jenkins.io/)。具体安装方法看官网即可，并不麻烦，安装期间可能会出现插件安装时超时失败，基本上是网络问题，重试几次就可以了。

### 配置IPA安装网页
自动化打包之后需要给测试安装打好的 ipa，可以使用第三方的云部署平台，也可以放在本地服务器。这时候就需要一个下载安装的主页，比如这样：

![安装ipa的网页](http://upload-images.jianshu.io/upload_images/1213714-509af5cecc93bd42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最简单的直接在本地服务器启动自带的apache服务器，参考[Mac OS X 上的Apache配置](http://www.jianshu.com/p/7b8d5d6f22c9)。再配置一个本地域名，如果需要的话。

### Xcode 8
这里重点要先提一下 `Xcode 8` 的特殊性。Xcode 8 修改了证书签名方式，引入了苹果“自认为非常牛逼”的自动签名功能。

![自动签名管理](http://upload-images.jianshu.io/upload_images/1213714-d2007952e3427eab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个东西确实应该是很厉害的东西，解决了多年iOS程序员“搞不清楚为什么证书签名又出错”的问题。如果单单使用Xcode来打包和发布，基本上就没有什么问题。但是如果要使用自动化打包，问题就来了。因为自动化打包其实就是使用 Xcode 的命令行工具 `xcodebuild` 来实现编译打包。

![Command Line Tools](http://upload-images.jianshu.io/upload_images/1213714-0200172eab07b808.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而坑爹的是，Xcode 8 的 `xcodebuild` 不支持自动签名。打包签名方式的改变导致了很多 Xcode 7 的自动化项目一升级到 Xcode 8 就失败了，而且官方也没有给出什么解决方案。

直到苹果在今年的 Xcode 9 宣布支持了，也就是说 Xcode 8 的自动签名功能做了一半？！😓 （下图为 [Session 403](https://developer.apple.com/videos/play/wwdc2017/403/)）

![What's New in Signing for Xcode and Xcode Server](http://upload-images.jianshu.io/upload_images/1213714-08cdb362775bc037.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 安装fastlane
[fastlane](https://github.com/fastlane/fastlane)是一个比较成熟的第三方自动化编译和发布的工具，其实应该说是个工具合集，包含了涵盖编译、打包、发布、测试等等各种大小工具。本文主要讲几个常用的命令，其他各种各样的命令自己都可以玩一下[fastlane actions](https://docs.fastlane.tools/actions/)。

首先确保安装了最新的 Xcode 命令行工具：
`xcode-select --install`

最好使用官方给[Homebrew](https://brew.sh/)命令来安装以保证环境的正确性：
`brew cask install fastlane` 

在代码工程根目录执行下面的命令来创建fastlane初始文件夹：
`fastlane init`

创建的过程中会提示输入你的Apple ID等信息，并用你输入的信息登录iTunes Connect以获取app相关的信息。期间如果出现错误并不需要惊慌，其实这里输入的信息最后都用来生成一个名为`Appfile`的文件，所以之后手动再填写信息也是可以的，详情参考[Appfile.md](https://github.com/fastlane/fastlane/blob/master/fastlane/docs/Appfile.md)。
`fastlane init`还会创建一个`fastlane`文件夹，里面有一个默认的`Fastfile`，这个文件是用来保存你自定义的命令的。`metadata`文件夹保存的是从iTunes Connect读取下来的app数据（好像没什么用）。

![fastlane init](http://upload-images.jianshu.io/upload_images/1213714-18e86e9f1cb5716d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 打包方案
自动化打包的目的一般是为了给QA快速便捷的提供测试包，或者按渠道发布包给不同的渠道平台等等。本文主要以测试包为目的讲解，将测试包分成以下两种：

* `Alpha` Dev（开发环境），用于开发期间测试
* `Beta` 正式（发布环境），用于app发布前测试正式环境
* `Testflight` 用于提交 App Store 之前的最终测试和发布

其他测试包如 Testflight 包、分支测试包等，不同的渠道包原理基本上都是一样的，需要根据项目需求来定。比较特别的是企业发布的`Inhouse`包，后面会稍微讲一下。

**创建代码分支**
确定好打包方案之后，就可以按需求建立代码分支来配合 Jenkins 拉取不同的分支代码进行编译打包了。如下图建立了 alpha 和 beta 两个分支对应 Jenkins 打包任务：

![git branch](http://upload-images.jianshu.io/upload_images/1213714-5f51fe4468bbb99a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**创建 Jenkins 任务**

![Jenkins Job](http://upload-images.jianshu.io/upload_images/1213714-ff1a811cd54ce01e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Branches to build](http://upload-images.jianshu.io/upload_images/1213714-ad02d03323668f48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Xcode 配置**
前文已经说明了 Xcode 8 命令行工具不支持自动签名功能，所以我这里采用“手动签名”然后创建不同的`Build Configurations`来配置不同的打包方案。
要注意的是，这个方式并不是唯一的方法，最初我也是通过自动签名然后用字符串替换的方式修改工程配置文件`project.pbxproj`里的编译参数来编译不同的包。但 Xcode 8 里跟签名相关的编译参数非常多，修改起来也很繁琐，一旦出错也很难找到原因，所以最终放弃了这种方式。
还有一种方式是“重签名”，通过同一个配置编译出 IPA 包，然后重签名得到不同的测试包。这也是一种可行的方案，但第一种方案其实是模拟 Xcode app 自己打包的方式，个人觉得更好更靠谱。

To be continue ...


