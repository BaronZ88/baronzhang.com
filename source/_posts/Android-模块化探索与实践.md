---
title: Android 模块化探索与实践
date: 2017-05-08 14:04:12
categories: Framework
tags:
- Framework
- Android
---

![](https://resources.baronzhang.com/blog/framework/android/2/header.jpg)

> 首发于《程序员》杂志五月刊
>
> 微信公众号：**BaronTalk**

## 前言

万维网发明人 Tim Berners-Lee 谈到设计原理时说过：“简单性和模块化是软件工程的基石；分布式和容错性是互联网的生命。” 由此可见模块化之于软件工程领域的重要性。

从 2016 年开始，模块化在 Android 社区越来越多的被提及。随着移动平台的不断发展，移动平台上的软件慢慢走向复杂化，体积也变得臃肿庞大；为了降低大型软件复杂性和耦合度，同时也为了适应模块重用、多团队并行开发测试等等需求，模块化在 Android 平台上变得势在必行。阿里 Android 团队在年初开源了他们的容器化框架 Atlas 就很大程度说明了当前 Android 平台开发大型商业项目所面临的问题。

<!-- more -->

## 什么是模块化

那么什么是模块化呢？《 Java 应用架构设计：模块化模式与 OSGi 》一书中对它的定义是：模块化是一种处理复杂系统分解为更好的可管理模块的方式。

上面这种描述太过生涩难懂，不够直观。下面这种类比的方式则可能加容易理解。

我们可以把软件看做是一辆汽车，开发一款软件的过程就是生产一辆汽车的过程。一辆汽车由车架、发动机、变数箱、车轮等一系列模块组成；同样，一款大型商业软件也是由各个不同的模块组成的。

汽车的这些模块是由不同的工厂生产的，一辆 BMW 的发动机可能是由位于德国的工厂生产的，它的自动变数箱可能是 Jatco（世界三大变速箱厂商之一）位于日本的工厂生产的，车轮可能是中国的工厂生产的，最后交给华晨宝马的工厂统一组装成一辆完整的汽车。这就类似于我们在软件工程领域里说的多团队并行开发，最后将各个团队开发的模块统一打包成我们可使用的 App 。

一款发动机、一款变数箱都不可能只应用于一个车型，比如同一款 Jatco 的 6AT 自动变速箱既可能被安装在 BMW 的车型上，也可能被安装在 Mazda 的车型上。这就如同软件开发领域里的模块重用。

到了冬天，特别是在北方我们可能需要开着车走雪路，为了安全起见往往我们会将汽车的公路胎升级为雪地胎；轮胎可以很轻易的更换，这就是我们在软件开发领域谈到的低耦合。一个模块的升级替换不会影响到其它模块，也不会受其它模块的限制；同时这也类似于我们在软件开发领域提到的可插拔。

## 模块化分层设计

上面的类比很清晰的说明的模块化带来的好处：

* 多团队并行开发测试；
* 模块间解耦、重用；
* 可单独编译打包某一模块，提升开发效率。

在[《安居客 Android 项目架构演进》](https://zhuanlan.zhihu.com/p/25420181)这篇文章中，我介绍了安居客 Android 端的模块化设计方案，这里我还是拿它来举例。但首先要对本文中的**组件**和**模块**做个区别定义

* **组件**：指的是单一的功能组件，如地图组件（MapSDK）、支付组件（AnjukePay）、路由组件（Router）等等；

* **模块**：指的是独立的业务模块，如新房模块（NewHouseModule）、二手房模块（SecondHouseModule）、即时通讯模块（InstantMessagingModule）等等；模块相对于组件来说粒度更大。

具体设计方案如下图：

![Android 模块化设计方案](https://resources.baronzhang.com/blog/framework/android/2/modularization.png)

整个项目分为三层，从下至上分别是：

* Basic Component Layer: 基础组件层，顾名思义就是一些基础组件，包含了各种开源库以及和业务无关的各种自研工具库；
* Business Component Layer: 业务组件层，这一层的所有组件都是业务相关的，例如上图中的支付组件 AnjukePay、数据模拟组件 DataSimulator 等等；
* Business Module Layer: 业务 Module 层，在 Android Studio 中每块业务对应一个单独的 Module。例如安居客用户 App 我们就可以拆分成新房 Module、二手房 Module、IM Module 等等，每个单独的 Business Module 都必须准遵守我们自己的 MVP 架构。

我们在谈模块化的时候，其实就是将业务模块层的各个功能业务拆分层独立的业务模块。所以我们进行模块化的第一步就是业务模块划分，但是模块划分并没有一个业界通用的标准，因此划分的粒度需要根据项目情况进行合理把控，这就需要对业务和项目有较为透彻的理解。拿安居客来举例，我们会将项目划分为新房模块、二手房模块、IM 模块等等。

每个业务模块在 Android Studio 中的都是一个 Module ,因此在命名方面我们要求每个业务模块都以 Module 为后缀。如下图所示：
<div align="left"><img src="https://resources.baronzhang.com/blog/framework/android/2/modules.png" width = "40%" alt="图片名称" align=center /></div>

对于模块化项目，每个单独的 Business Module 都可以单独编译成 APK。在开发阶段需要单独打包编译，项目发布的时候又需要它作为项目的一个 Module 来整体编译打包。简单的说就是开发时是 Application，发布时是 Library。因此需要在 Business Module 的 build.gradle 中加入如下代码：

```groovy
if(isBuildModule.toBoolean()){
    apply plugin: 'com.android.application'
}else{
    apply plugin: 'com.android.library'
}
```

> isBuildModule 在项目根目录的 gradle.properties 中定义:
>
> ```java
> isBuildModule=false
> ```

同样 Manifest.xml 也需要有两套：

```groovy
sourceSets {
   main {
       if (isBuildModule.toBoolean()) {
           manifest.srcFile 'src/main/debug/AndroidManifest.xml'
       } else {
           manifest.srcFile 'src/main/release/AndroidManifest.xml'
       }
   }
}
```

如图：
<div align="left"><img src="https://resources.baronzhang.com/blog/framework/android/2/manifest.png" width = "45%" alt="图片名称" align=center /></div>

debug 模式下的 AndroidManifest.xml :

```xml
<application
   ...
   >
   <activity
       android:name="com.baronzhang.android.newhouse.NewHouseMainActivity"
       android:label="@string/new_house_label_home_page">
       <intent-filter>
           <action android:name="android.intent.action.MAIN" />
           <category android:name="android.intent.category.LAUNCHER" />
       </intent-filter>
   </activity>
</application>
```

realease 模式下的 AndroidManifest.xml :

```xml
<application
   ...
   >
   <activity
       android:name="com.baronzhang.android.newhouse.NewHouseMainActivity"
       android:label="@string/new_house_label_home_page">
       <intent-filter>
           <category android:name="android.intent.category.DEFAULT" />
           <category android:name="android.intent.category.BROWSABLE" />
           <action android:name="android.intent.action.VIEW" />
           <data android:host="com.baronzhang.android.newhouse"
               android:scheme="router" />
       </intent-filter>
   </activity>
</application>
```

同时针对模块化我们也定义了一些自己的游戏规则:

* 对于 Business Module Layer，各业务模块之间不允许存在相互依赖关系，它们之间的跳转通讯采用路由框架 Router 来实现（后面会介绍 Router 框架的实现）;
* 对于 Business Component Layer，单一业务组件只能对应某一项具体的业务，个性化需求对外部提供接口让调用方定制;
* 合理控制各组件和各业务模块的拆分粒度，太小的公有模块不足以构成单独组件或者模块的，我们先放到类似于 CommonBusiness 的组件中，在后期不断的重构迭代中视情况进行进一步的拆分;
* 上层的公有业务或者功能模块可以逐步下放到下层，合理把握好度就好；
* 各 Layer 间严禁反向依赖，横向依赖关系由各业务 Leader 和技术小组商讨决定。

## 模块间跳转通讯（Router）

对业务进行模块化拆分后，为了使各业务模块间解耦，因此各个 Bussiness Module 都是独立的模块，它们之间是没有依赖关系。那么各个模块间的跳转通讯如何实现呢？

比如业务上要求从**新房的列表页**跳转到**二手房的列表页**，那么由于是 NewHouseModule 和 SecondHouseModule 之间并不相互依赖，我们通过想如下这种显式跳转的方式来实现 Activity 跳转显然是不可能的实现的。

```Java
Intent intent = new Intent(NewHouseListActivity.this, SecondHouseListActivity.class);
startActivity(intent);
```

有的同学可能会想到用隐式跳转，通过 Intent 匹配规则来实现：

```Java
Intent intent = new Intent(Intent.ACTION_VIEW, "<scheme>://<host>:<port>/<path>");
startActivity(intent);
```

但是这种代码写起来比较繁琐，且容易出错，出错也不太容易定位问题。因此一个简单易用、解放开发的路由框架是必须的了。
<div align="left"><img src="https://resources.baronzhang.com/blog/framework/android/2/router2.png" width = "45%" alt="图片名称" align=center /></div>

我自己实现的路由框架分为<b>路由（Router）</b> 和<b>参数注入器（Injector）</b> 两部分：
<div align="left"><img src="https://resources.baronzhang.com/blog/framework/android/2/router.png" width = "45%" alt="图片名称" align=center /></div>

> Router 提供 Activity 跳转传参的功能；Injector 提供参数注入功能，通过编译时生成代码的方式在 Activity 获取获取传递过来的参数，简化开发。

### Router

路由（Router）部分通过 Java 注解结合动态代理来实现，这一点和 Retrofit 的实现原理是一样的。

首先需要定义我们自己的注解（篇幅有限，这里只列出少部分源码）。

用于定义跳转 URI 的注解 FullUri：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FullUri {
    String value();
}
```

用于定义跳转传参的 UriParam（ UriParam 注解的参数用于拼接到 URI 后面）：

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface UriParam {
    String value();
}
```

用于定义跳转传参的 IntentExtrasParam（ IntentExtrasParam 注解的参数最终通过 Intent 来传递）：

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface IntentExtrasParam {
    String value();
}
```

然后实现 Router ,内部通过动态代理的方式来实现 Activity 跳转：

```java
public final class Router {
    ...
    public <T> T create(final Class<T> service) {

        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

                FullUri fullUri = method.getAnnotation(FullUri.class);
                StringBuilder urlBuilder = new StringBuilder();
                urlBuilder.append(fullUri.value());
                //获取注解参数
                Annotation[][] parameterAnnotations = method.getParameterAnnotations();
                HashMap<String, Object> serializedParams = new HashMap<>();
			    //拼接跳转 URI
                int position = 0;
                for (int i = 0; i < parameterAnnotations.length; i++) {
                    Annotation[] annotations = parameterAnnotations[i];
                    if (annotations == null || annotations.length == 0)
                        break;

                    Annotation annotation = annotations[0];
                    if (annotation instanceof UriParam) {
                        //拼接 URI 后的参数
                        ...
                    } else if (annotation instanceof IntentExtrasParam) {
                        //Intent 传参处理
                        ...
                    }
                }
                //执行Activity跳转操作
                performJump(urlBuilder.toString(), serializedParams);
                return null;
            }
        });
    }
	...
}
```

上面是 Router 实现的部分代码，在使用 Router 来跳转的时候，首先需要定义一个 Interface（类似于 Retrofit 的使用方式）：

```java
public interface RouterService {

    @FullUri("router://com.baronzhang.android.router.FourthActivity")
    void startUserActivity(@UriParam("cityName")
    		String cityName, @IntentExtrasParam("user") User user);

}
```

接下来我们就可以通过如下方式实现 Activity 的跳转传参了：

```java
 RouterService routerService = new Router(this).create(RouterService.class);

 User user = new User("张三", 17, 165, 88);
 routerService.startUserActivity("上海", user);
```

### Injector

通过 Router 跳转到目标 Activity 后，我们需要在目标 Activity 中获取通过 Intent 传过来的参数：

```java
getIntent().getIntExtra("intParam", 0);

getIntent().getData().getQueryParameter("preActivity");
```

为了简化这部分工作，路由框架 Router 中提供了 Injector 模块在编译时生成上述代码。参数注入器（Injector）部分通过 Java 编译时注解来实现，实现思路和 ButterKnife 这类编译时注解框架类似。

首先定义我们的参数注解 InjectUriParam ：

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface InjectUriParam {
    String value() default "";
}
```

然后实现一个注解处理器 InjectProcessor ，在编译阶段生成获取参数的代码：

```java
@AutoService(Processor.class)
public class InjectProcessor extends AbstractProcessor {
    ...
   @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {

        //解析注解
        Map<TypeElement, TargetClass> targetClassMap = findAndParseTargets(roundEnvironment);

        //解析完成后，生成的代码的结构已经有了，它们存在InjectingClass中
        for (Map.Entry<TypeElement, TargetClass> entry : targetClassMap.entrySet()) {
            ...
        }
        return false;
    }
    ...
}
```

使用方式类似于 ButterKnife ，在 Activity 中我们使用 Inject 来注解一个全局变量：

```java
@Inject User user;
```

然后 onCreate 方法中需要调用 inject(Activity activity) 方法实现注入：

```java
RouterInjector.inject(this);
```

这样我们就可以获取到前面通过 Router 跳转的传参了。

> 由于篇幅限制，加上为了便于理解，这里只贴出了极少部分 [Router](https://github.com/BaronZ88/Router) 框架的源码。希望进一步了解 Router 实现原理的可以到 GiuHub 去翻阅源码，[Router](https://github.com/BaronZ88/Router) 的实现还比较简陋，后面会进一步完善功能和文档，之后也会有单独的文章详细介绍。源码地址：[https://github.com/BaronZ88/Router](https://github.com/BaronZ88/Router)

## 问题及建议

### 资源名冲突

对于多个 Bussines Module 中资源名冲突的问题，可以通过在 build.gradle 定义前缀的方式解决：

```groovy
defaultConfig {
   ...
   resourcePrefix "new_house_"
   ...
}
```

而对于 Module 中有些资源不想被外部访问的，我们可以创建 res/values/public.xml，添加到 public.xml 中的 resource 则可被外部访问，未添加的则视为私有：

```xml
<resources>
    <public name="new_house_settings" type="string"/>
</resources>
```

### 重复依赖

模块化的过程中我们常常会遇到重复依赖的问题，如果是通过 aar 依赖， gradle 会自动帮我们找出新版本，而抛弃老版本的重复依赖。如果是以 project 的方式依赖，则在打包的时候会出现重复类。对于这种情况我们可以在 build.gradle 中将 compile 改为 provided，只在最终的项目中 compile 对应的 library ；

其实从前面的安居客模块化设计图上能看出来，我们的设计方案能一定程度上规避重复依赖的问题。比如我们所有的第三方库的依赖都会放到 OpenSoureLibraries 中，其他需要用到相关类库的项目，只需要依赖 OpenSoureLibraries 就好了。

### 模块化过程中的建议

对于大型的商业项目，在重构过程中可能会遇到业务耦合严重，难以拆分的问题。**我们需要先理清业务，再动手拆分业务模块**。比如可以先在原先的项目中根据业务分包，在一定程度上将各业务解耦后拆分到不同的 package 中。比如之前新房和二手房由于同属于 app module，因此他们之前是通过隐式的 intent 跳转的，现在可以先将他们改为通过 Router 来实现跳转。又比如新房和二手房中公用的模块可以先下放到 Business Component Layer 或者 Basic Component Layer 中。在这一系列工作完成后再将各个业务拆分成多个 module 。

模块化重构需要渐进式的展开，不可一触而就，不要想着将整个项目推翻重写。线上成熟稳定的业务代码，是经过了时间和大量用户考验的；全部推翻重写往往费时费力，实际的效果通常也很不理想，各种问题层出不穷得不偿失。对于这种项目的模块化重构，我们需要一点点的改进重构，可以分散到每次的业务迭代中去，逐步淘汰掉陈旧的代码。

各业务模块间肯定会有公用的部分，按照我前面的设计图，公用的部分我们会根据业务相关性下放到业务组件层（Business Component Layer）或者基础组件层（Common Component Layer）。对于太小的公有模块不足以构成单独组件或者模块的，我们先放到类似于 CommonBusiness 的组件中，在后期不断的重构迭代中视情况进行进一步的拆分。过程中完美主义可以有，切记不可过度。

以上就是我在模块化探索实践方面的一些经验，不住之处还望大家指出。

* 模块化示例项目 [ModularizationProject](https://github.com/BaronZ88/ModularizationProject) 源码地址：[https://github.com/BaronZ88/ModularizationProject](https://github.com/BaronZ88/ModularizationProject)
* 路由框架 [Router](https://github.com/BaronZ88/Router) 源码地址：[https://github.com/BaronZ88/Router](https://github.com/BaronZ88/Router)

> 如果你喜欢我的文章，就关注下我的公众号 **BaronTalk** 、 [**知乎专栏**](https://zhuanlan.zhihu.com/baron) 或者在 [**GitHub**](https://github.com/BaronZ88) 上添个 Star 吧！
>   
> * 微信公众号：**BaronTalk**
> * 知乎专栏：[https://zhuanlan.zhihu.com/baron](https://zhuanlan.zhihu.com/baron)  
> * GitHub：[https://github.com/BaronZ88](https://github.com/BaronZ88)
> * 个人博客：[http://baronzhang.com](http://baronzhang.com)

<div align="center"><img src="https://resources.baronzhang.com/blog/common/gzh3.png" width="85%"/></div>