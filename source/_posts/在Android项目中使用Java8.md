---
title: 在Android项目中使用Java8
date: 2017-02-05 23:37:48
categories: Java
tags:
- Java
- Android
---

> 首发于公众号：**BaronTalk**

## 前言

在过去的文章中我介绍过Java8的一些新特性，包括：

1. Java8新特性第1章(Lambda表达式)
2. Java8新特性第2章(接口默认方法)
3. Java8新特性第3章(Stream API)

之前由于Android平台不支持Java8，如果我们想在Android项目中使用Lambda表达式、Stream API等Java8中的新特性就必须使用Retrolambda、Lightweight-Stream-API等第三方开源库来实现。现在Google爸爸终于让Android平台支持Java8了，这篇文章中便来和大家聊聊如何在Android项目中配置使用Java8。

遗憾的是目前Android平台仅支持Java8的部分新特性，当我们在开发面向Android N及以上版本的应用时(即minSdkVersion>=24)，可以使用如下新特性：

* [Lambda表达式\(Lambda Expressions\)](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)（也可以在minSdkVersion<24的情况下使用）
* [方法引用\(Method References\)](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html)（也可以在minSdkVersion<24的情况下使用）
* [Stream API\(Streams\)](http://www.oracle.com/technetwork/articles/java/ma14-java-se-8-streams-2177646.html)
* [接口默认方法\(Default Methods\)](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html) 
* [重复注解\(Repeating Annotations\)](https://docs.oracle.com/javase/tutorial/java/annotations/repeating.html)

简单的说就是现在你的项目要想使用Stream API、接口默认方法和重复注解就要求你的minSdkVersion>=24，而Lambda表达式和方法引用则对minSdkVersion无要求。关于这些新特的使用及分析可以看看我之前的文章。

<!--more-->

## Jack(Java Android Compiler Kit)

要想在Android项目中使用Java8的新特性，需要将你的Android Studio升级到2.1及以上版本，并采用新的Jack(Java Android Compiler Kit)编译。新的 Android 工具链将 Java 源语言编译成 Android 可读取的 Dalvik 可执行文件字节码，且有其自己的 .jack 库格式，在一个工具中提供了大多数工具链功能：重新打包、压缩、模糊化以及 Dalvik 可执行文件分包。

以下是构建 Android Dalvik 可执行文件可用的两种工具链的对比：

* 旧版 javac 工具链：  
  <font color="ff0000"> `javac (.java --> .class) --> dx (.class --> .dex)` </font>
* 新版 Jack 工具链：  
  <font color="ff0000"> `Jack (.java --> .jack --> .dex)` </font>
	
## 配置

为了在项目中使用Java8，我们还需要项目module中的gradle.build文件中加入如下代码：

```Groovy
android {

  compileSdkVersion 24
  buildToolsVersion "24.0.3"
    
  defaultConfig {
    
    applicationId "me.baron.hellojava8"
    minSdkVersion 24
    targetSdkVersion 24
    versionCode 1
    versionName "1.0"
        
    jackOptions {
      enabled true
    }
  }
  
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
```

## 使用

进行上述配置后大家就可以在Android项目中尽情的探索使用Java8的新特性了。比如之前我们实现button的点击事件时需要这这样写：

```java
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
		v.setText("lalala");
   	}
});
```

现在我们便可以使用Java8的Lambda表达式来实现了：

```java
button.setOnClickListener(v -> v.setText("lalala"));
```

如果你项目的minSdkVersion>=24，我们还可以使用Stream API。比方说有一个形状集合shapes，现在我们想把所有蓝色的形状提取到新的List里。通过Stream API则可以很轻易的办到：

```java
List<Shape> blue = shapes.stream()
						  .filter(s -> s.getColor() == BLUE)
						  .collect(Collectors.toList());
```

## 总结

Java8的新特性并不是本文的重点，对此有兴趣的同学可以去翻看我之前的文章。当前Jack编译器还有诸多限制，比如在使用新的Jack工具链时会禁用Instant Run以及前面提到的新特性对我们的最低支持版本和编译版本有要求等等(我猜想Jack对Buck、Layoutcast、Freeline等编译方案也会有影响，没做过验证，有了解的同学可以在评论区留言和大家交流下)；总之要想在Android项目中愉快的使用Java8全部的新特性还需时日。期待Google爸爸尽快优化吧！

参考资料：
* [https://developer.android.com/guide/platform/j8-jack.html](https://developer.android.com/guide/platform/j8-jack.html)
* [https://medium.com/@sergii/java-8-in-android-n-preview-76184e2ab7ad](https://medium.com/@sergii/java-8-in-android-n-preview-76184e2ab7ad)

------

> 如果你喜欢我的文章，就关注下我的公众号 **BaronTalk** 、 [**知乎专栏**](https://zhuanlan.zhihu.com/baron) 或者在 [**GitHub**](https://github.com/BaronZ88) 上添个 Star 吧！
>
> - 微信公众号：**BaronTalk**
> - 知乎专栏：[https://zhuanlan.zhihu.com/baron](https://zhuanlan.zhihu.com/baron)  
> - GitHub：[https://github.com/BaronZ88](https://github.com/BaronZ88)

<div align="center"><img src="https://resources.baronzhang.com/blog/common/gzh3.png" width="85%"/></div>