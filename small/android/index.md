# Small for Android
> **Small**, a small framework to split app into small parts.

**{Small}** 是一款轻型组件化框架，本Android版本支持对apk、web等组件包进行插件化调用与热更新。

**{Small}** 为App开发而生，我们不需要无所不能，只要恰到好处。

[![Build Status](https://travis-ci.org/facebook/buck.svg)](https://travis-ci.org/facebook/buck)

## 特点

* 完美内置
    - 所有插件支持内置于libs目录，以\*.so形式存在
* 高度透明
    - apk插件包代码编写与独立apk无异
    - web插件页面跳转、window事件与独立web无异
    - 调试插件中代码与整包开发无异（归功于Android Studio module）
* URI连接一切
    - 宿主界面、apk插件界面、本地化网页、远程网页、自定义插件界面，所有的一切均可通过URI来进行跳转与参数传递
* 自动降级
    - 基于URI连接，打开界面优先级从Apk Activity->本地化网页->远程网页递减，这个顺序遵循“高性能、低效率”->“高效率、低性能”
* 高度可扩展
    - 通过自定义插件加载器，你可以启动任意一个任意语言的插件。

## Apk组件
### 功能

* 动态加载
    - 支持.so后缀apk插件（支持四大组件）
* 动态注册
    - 支持Activity组件（另三个组件使用频次低，预设即可）
* 资源访问
    - 支持宿主、共享库、插件三者之间互相访问
* 界面跳转
    - 支持宿主与插件之间相互跳转
    - 支持插件之间相互跳转

### 原理

#### 1. 动态类加载

Android的类是通过_**DexClassLoader**_ 加载的，其动态加载默认只支持\*.apk, \*.zip后缀，为了实现组件包的完美内置，我们必须将apk类型的组件包用\*.so后缀的形式存放于libs目录，从而在程序安装时可以自动复制到应用缓存中。

为此，我们通过反射绕过后缀检查，模拟_**DexClassLoader**_构造pathList：

1. 生成\*.so文件的dexFile
2. 封装成dexElement
3. 并入宿主ClassLoader的pathList。

```
public static boolean expandDexPathList(ClassLoader cl, String dexPath,
                                     String libraryPath, String optDexPath) {
        try {
            File pkg = new File(dexPath);
            DexFile dexFile = DexFile.loadDex(dexPath, optDexPath, 0);
            Object element = makeDexElement(pkg, dexFile);
            fillDexPathList(cl, element);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }
```

#### 2. 动态Activity注册

Activity是Android四大系统组件之一，要实现其完美生命周期，必须要先在宿主 _**AndroidManifest.xml**_ 中进行注册。为此，我们在宿主中预注册占坑Activity，再在运行时通过反射替换的_**Instrumentation**_“偷桃换李”，启动插件Activity。（参考[Android-Plugin-Framework][apf]）

1. Plugin intent -> Stub intent (Instrumentation.execStartActivity)
2. Stub intent -> Plugin activity (Instrumentation.newActivity)

#### 3. 资源分区
    
Android资源访问是通过唯一对应的int型id来定位的，在实际应用场景中，为减小插件包大小，我们需要将公共资源剥离出来。为保证各组件在运行时资源id不重复，我们需要对不同的组件包分配不同区段的id。

资源id格式为0xPPTTNNNN。PP是package id，系统保留0x01与0x02、用户程序默认0x7f。资源分区要做的就是把组件包的package id分配在[0x03, 0x7e]之间。常见的做法有：

1. 定义public.xml
2. 修改aapt源码，重新编译aapt
    
对于(1)，很可惜在gradle 1.3之后不再支持；
对于(2)，是现在很多开源插件化项目的普遍做法，如[AACD][aacd]、[DynamicAPK][dapk]. 但是代价太高，必须下载aapt源码、修改、重新编译，再替换Build Tools下的aapt文件。同时还要跟随Build Tools进行升级维护。

为此，我们寻求并实现了第3种方案：

我们知道，apk文件实际上是个压缩包，解压后我们能看到如下结构：

* resources.arsc
* AndroidManifest.xml
* res/layout
    - activity_main.xml
    - xxx.xml

其中的resources.arsc、以及各个*.xml文件是aapt命令打包生成的二进制文件，资源id被编排写入其中。同时，aapt为此生成了资源id清单R.java文件以供程序访问。

大家应该明白了，既然我们无法预设package id参数，那么我们就在事后进行重写。这个过程并不容易，但原理相对简单，就是定位到0x7f，替换之。

这一工作我hook在了_**processReleaseResources.doLast**_ 上，具体细节可以参看_**small/scripts/coreResIdReassigner.gradle**_。

#### 4. 组件瘦身

在生产环境中，我们要求作为独立更新的组件包越小越好。所以，我们提取了公共类、公共资源作为组件包的依赖库，并且在编译组件包过程中依次剥离，形成“瘦身包”。

剥离的时机很关键，早了无法通过编译，迟了已经被打包进去。

* 剥离公共资源 - after aapt: _**processReleaseResources.doLast**_
* 剥离公共类库 - before dex: _**dexRelease.doFirst**_

编译组件包只需执行gradle task: _**assembleRelease**_，生成的“.so”文件会被自动导出至宿主 _**libs/armeabi**_ 目录下。

特别的，如果要在模拟器下调试运行，需导出至 _**libs/x86**_ 目录下，可以使用以下脚本执行：

```
./gradlew -Dbundle.arm=x86 -p [your bundle name] assembleRelease
```

## Web组件

随着h5的盛行，打通native与web已经是大势所趋

## 组件路由
要实现跨组件之间特别是跨类型组件之间的调用，必须要有一个绝对的标准来规范页面跳转与参数传递。而且这个标准必须足够低、而且通用。很自然的，我们想到了HTTP的URL。其页面标记与参数定义均为字符串，这足以跨平台。

比如：http://m.wequick.net/small/**home**/index.html

更进一步，当我们把“http://m.wequick.net/small/”设为基地址，把“index.html”作为默认页面时，我们的**URI**就可以提取为“**home**”。那么从“**home**”到“http://m.wequick.net/small/**home**/index.html”就是一个默认路由（自动降级的最后跳转方案）。

此时打开该网页可以简单调用如下：

```
Small.openUri("home", context); // (1)
```

再进一步，我们可以通过**assets/bundles.json**来进行高级路由配置。假设有一天，我们认为远程网页的性能太低了，要把**home**做成本地Activity来展示，我们可以新建一个application module - **bundle.home**，包名为“net.wequick.smallandroid.bundle.home”，并在**bundles.json**中追加配置如下：

```
"bundles": [
  ...,
  {
    "uri": "home",
    "pkg": "net.wequick.smallandroid.bundle.home"
  }
]
```
那么现在**(1)**中语句执行后，就会自动跳转到**bundle.home**插件的主界面。

如果不是主界面呢？可以通过“**rules**”字段来定义路由规则，比如：

```
  {
    "uri": "home",
    "pkg": "net.wequick.smallandroid.bundle.home",
    "rules": {
      "about": "MyAbout"
    }
  }
```
此时调用

```
Small.openUri("home/about", context);
```
将会优先跳转**bundle.home**插件的**MyAboutActivity**，如果没有找到该类，则自动降级至网页链接“http://m.wequick.net/small/home/about”。

## License
Apache License 2.0

[apf]: https://github.com/limpoxe/Android-Plugin-Framework
[aacd]: https://github.com/CtripMobile/DynamicAPK
[dapk]: https://github.com/CtripMobile/DynamicAPK