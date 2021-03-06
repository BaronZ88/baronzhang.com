---
title: 安居客 Android App 走向平台化
date: 2019-11-09 13:17:03
tags:
---

![](https://resources.baronzhang.com/architecture/平台化/头图.jpg)

> 首发于微信公众号：**BaronTalk**

安居客 Android App 距离上次的模块化/组件化重构已经两年多了，重构之后很好的支撑了两年多以来的业务发展。但这个世界总是在向前走的，没有任何一种架构能够一劳永逸的解决所有问题，外部环境的不断变化相应的也要求项目架构做出改变，以此来应对环境变化所带来的挑战。

本文分享的就是我们安居客 App 团队这次向平台化转型的背景、转型过程中所面临的问题、挑战、我们的解决方案以及我个人在这个过程中的收获和感悟。

<!--more-->

## 一. 背景

自 18 年以来的市场环境大家都知道，各大公司都在缩减开支、提升组织效率，希望用更少的投入带来更多的产出，以此来应对经济下行带的不确定性。在这个大背景下，集团在 18 年底做出了一系列人效提升、业务整合的动作。

拿房产业务举例：58 App 里的房产业务之前是由北京的房产团队开发的，而整个安居客 App 是由我所在的上海安居客团队开发的。在这次调整之后，北京的房产团队作为一条垂直业务线来负责 58 App 和安居客 App 中的租房业务的开发，安居客团队则继续负责整个安居客 App（包含新房、二手房、内容、IM 等业务）的开发，同时还要开发 58 App 中二手房、新房、房产大内容等业务。

![](https://resources.baronzhang.com/architecture/平台化/业务调整.png)

这样一次调整给我们带来了三大挑战：

1. 在人力不变，工作量翻倍的情况下，我们要如何保证业务的迭代速度不受影响？
2. 由以往的「单兵作战」变成了跨团队跨地域协作开发，我们要如何作为一个平台方来和业务方协作开发？
3. 对于 58 App 而言我们是业务方，对于北京房产团队而言我们是平台方，我们要如何同时处理好双重角色？

面对这些挑战我们需要转变开发思维、改进项目架构。

**一次项目重构和架构升级，不只是要解决当下的问题，更要为未来一到两年的业务发展提供支持。**为了解决这一系列的问题，我们联合 58 无线团队、58 房产团队、前端、后端多个团队的同学一起发起了「木星计划」，也开启了我们的平台化转型之路。

## 二. 问题、挑战及解决方案

当公司的业务调整政策下来以后，团队面临的最急迫的一个选择是继续在 58 App 上维护老的房产代码，迭代新功能；还是将安居客现有业务搬迁到 58 App 实现一套代码在两个 App 里运行呢？

* 前者我们需要分一半的人力去开发 58 App，这必然导致团队的需求消化量减半，业务迭代速度受影响。好处是暂时不用做任何技术上的改造。

* 后者需要我们协调 58 无线、58 房产、前后端等多个团队一起对 App 做一次「大手术」，同时还需要说服产品团队容忍改造期间业务迭代速度的放缓。好处是能够一次性将安居客上的房产业务搬迁到 58 App 上，后续我们只需针对房产业务做一次开发就能跑在两个 App 上，用一份人力做了两份的活，开发效率翻倍。

为长远计，我们最终选择了后者。可是要做到一套代码双端运行并不容易，58 App 和安居客 App 隶属于北京、上海两个不同的团队开发，大家底层库不一样，技术方案不一样，开发模式也不一样，要达到我们的目的必然要费一番功夫。接下来我从整体到局部逐步介绍我们在平台化演进过程中的设计思路、遇到的问题和解决方案

### 2.1 整体设计

所谓平台化，就是安居客 App 要作为一个平台来对平台上承载的各种垂直业务提供服务，每个服务都需要对上层提供标准化的接口来支撑平台上各类垂直业务的功能。

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/安居客App平台.png" width = "65%" alt="安居客App平台" align=center /></div>

这一次的项目重构除了要转型平台型 App 支持其他业务的接入和未来业务的发展，还有更重要一点是要做到一套代码在 58 和安居客双平台运行。

要做到这一点，在现有的业务体系和代码体量下虽然工作量巨大，但大的思路确是很清晰简单的：
* 底层库能统一的尽量统一；
* 短时间内无法统一的库以及平台特性通过中间层屏蔽平台差异。

#### 安居客 App 架构调整

针对安居客 App ，我们需要调整架构，引入一个平台中间层并针对平台中间层接口做安居客 App 侧的实现，并将垂直业务中原本调用平台接口的地方改为调用中间层接口。同时将之前的业务组件层细分为「平台级组件」和「业务级组件」，所有垂直业务不再依赖平台级组件，只依赖**平台中间层**和**业务级组件**，并明确业务代码迁移的边界。

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/安居客App架构变化.png" width = "100%" alt="安居客App架构变化" align=center /></div>

#### 58 App 架构调整

同时 58 无线团队的同学也改进了他们的架构，和我们一样引入了平台中间层及中间层在 58 App 上的实现，保证安居客业务迁移进来后能正常编译运行。

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/58App架构变化.png" width = "100%" alt="58App架构变化" align=center /></div>
上面这两张架构图呈现了两个 App 架构调整的大方向，具体做了哪些、遇到了哪些问题我在下面的小节里一一介绍。

### 2.2 平台中间层屏蔽底层差异

计算机领域的任何问题都可以添加一个中间层来解决。58 App 和安居客 App 作为两个不同的平台，对业务层提供的能力是不一样的，接口、方法名都是不一样的。同一块业务代码要跑在两个不同的平台，必然要引入一个中间层来抹平差异、屏蔽底层细节，同时对外提供统一的接口供业务层调用。

因此 58 无线团队、58 房产团队和我们安居客团队三方协商，共同制定了一套平台中间层 API，然后两个平台再针对平台中间层 API 做具体实现。

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/平台中间层设计及示例.png" width = "70%" alt="平台中间层设计及示例" align=center /></div>

为了便于后期管理、划清代码边界，我们将平台中间层服务进一步细化，根据平台差异性将中间层划分为平台公共服务、安居客平台特有服务、58 平台特有服务等。以 Java Package 作为区分,划分到不同的包结构下。一旦后期某一特有服务变成了公共服务，则将其往平台公共服务迁移。

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/平台中间层服务划分.png" width = "60%" alt="平台中间层服务划分" align=center /></div>

在实现上，平台中间层会提供一系列 Service 接口供垂直业务调用，同时提供一个 PlatFormServiceRegistry 类用来注册和获取服务。

```java
public class PlatFormServiceRegistry {

    ...

    /**
     * 注册服务
     */
    private void registService(Class serviceInterface, Class<? extends IService> serviceImpl) {
        if (serviceInterface != null && serviceImpl != null ) {
            classMap.put(serviceInterface.getName(), serviceImpl);
        }
    }


    /**
     * 获取服务
     */
    private <T> T getService(Class<? extends T> service) {
        
				···
          
        IService instance = serviceImplMap.get(service.getName());
        if (null == instance) {
            try {
                Class<? extends IService> serviceClass = classMap.get(service.getName());
                if (serviceClass != null) {
                    instance = serviceClass.getConstructor().newInstance();
                    serviceImplMap.put(service.getName(), instance);
                }
            } catch (Exception e) {
                Log.d(TAG, e.toString());
            }
        }
        return (T) instance;
    }
}
```

在使用方式上，平台方首先要注册服务

```java
//注册平台服务
PlatFormServiceRegistry.registeAppInfoService(AjkAppInfoServiceImpl.class);
```

业务方在使用服务的时候获取到对应的 Service 就可以调用相关方法了。
```java
//使用平台服务
IAppInfoService appInfoService = PlatFormServiceRegistry.getAppInfoService();
appInfoService.getAppName(context);
```

### 2.3 统一 HybridSDK 解决双平台 H5 交互问题

安居客 App 由于历史原因，Android 和 iOS 在与 H5 的交互协议上有很多不一样（虽然后期统一过 JSBridge 协议，但很多历史遗留协议仍旧是不一致的），58 App 上的 Native 和 JS 交互协议更是和安居客不一致。为了解决这些问题我们需要 Android 和 iOS、58 和 安居客有一个统一的 Native 与 JS 的交互协议；同时为了兼容历史协议，让业务平稳过渡，我们还需要设计一套过渡方案。

在未实现协议统一的情况下，一个 H5 页面要上 58 App 和安居客 App 两个平台，需要支持两套协议，再加上安居客之前 Android、iOS 协议的不一致，一个 H5 页面最多可能需要支持 4~5 套协议。

为了实现协议的最终统一，并且让业务平稳过渡，我们引入了一套过渡方案。在保留两个平台现有协议和 JSBridge SDK 的情况下，58 无线团队的同学设计并开发了一个全新的 HybridSDK，过渡阶段三套协议并存，来不及调整的旧业务使用旧协议，新开发及本次要调整的业务使用新协议。

然后随着业务的迭代，不断废弃两个平台的自有协议，最终走向统一。

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/JSBride协议统一.png" width = "90%" alt="中间层设计" align=center /></div>

### 2.4 统一路由协议、API 动态下发路由解决页面交互问题

现有 App 内的页面跳转要么是 intent 跳转，要么是写死的路由跳转。在这次的平台化改造过程中，我们从 58 App 上也学到了很多东西，其中动态路由下发就是我们学习并引入到安居客 App 中的。

简单的说就是 App 给各个页面定义好路由协议，App 在调用 API 的时候返回结果中会包含当前页面跳转到下一页面的路由协议，这就是所谓的动态路由下发。这样做会带来三个好处：

1. **可以做到完善的降级策略**，比如当线上的房源详情页出现了大面积的 Crash，API 可以返回一个 H5 页面的路由协议，临时用 H5 的房源详情页替代；同时 App 上线 HotFix，等 HotFix 覆盖率达到一定程度后 API 再改回下发Native 路由，跳回 Native 页面；
2. **可以支持新功能的效果验证**，利用 H5 和 Native 自由切换这一特性，可以在不发版的情况下验证新功能的数据效果。比如要上线验证某个改动或者某个新功能的效果，之前需要 App 发版，现在只需要做一版 H5 页面，API 直接路由到这个 H5，如果验证下来效果好则可以开发对应的 Native 页面，不好则可以再尝试其他方案；
3. **可以支持平台化改造过程中的灰度上线**，这一点放到下一小节详细说明。

### 2.5 灰度策略保证上线后的稳定性

我们这次的平台化改造，无论对安居客 App 还是 58 App 来说都是一次「大手术」，术后能否保证线上的稳定性、业务数据不受影响是非常重要的。就拿把安居客业务迁移到 58 App 这件事来说，如果一股脑的用安居客迁移过去的房产业务代码替代 58 App 内的房产业务代码，就算我们在技术上做到了绝对的稳定，也难保业务数据不会受影响。

好在得益于上面提到的动态路由下发方案，我们可以做到在少量城市、少量用户上做灰度，让这部分人先试用安居客迁移过去的业务，其它用户继续使用老的房产业务，数据效果好再逐步加量，直到完全替代。这样就能最大限度的降低影响，保证上线后的稳定性。

### 2.6 业务平移带来的包大小问题

虽然在上一次的模块化/组件化改造过程中我们对各项垂直业务做了拆分、解耦，但是各业务还是有很多重叠的业务，于是我们将这些业务下沉到 CommonBusiness 组件中，同时这个 CommonBusiness 组件里还包含了一些 App 平台级别的基础功能，因此这个 CommonBusiness 组件成了个大而全的东西。

在这次的架构调整中，我们为了实现业务的快速平移，将 CommonBusiness 和新房、二手房等几条垂直业务线的代码一股脑的迁移进了 58 App，这就直接导致了 58 App 体积的快速膨胀。于是在后期，我们将 CommonBusiness 按能力拆分成了多个独立的组件，并且分为了「平台级组件」和「业务级组件」。平台级组件属于安居客平台特有，不随业务迁移；业务级组件属于多个垂直业务公用的组件，随业务代码一起迁移到 58 App。这一点在前面的架构图中有体现。

### 2.7 持续演进

前面介绍中间层的时候提到，还有一部分底层库暂时无法统一，现阶段是通过引入中间层来解决的。但从长远来看，整个集团无线体系下，依赖的底层库还是要走向统一。就拿分享组件来说，现阶段安居客和 58 都是使用自己的 ShareSDK，然后中间层定义了一套分享接口，两个 App 分别调用自己的 ShareSDK 来实现接口满足业务的分享需求。为了进一步降低开发成本，避免重复造轮子，后期双方还需要统一使用同一个 ShareSDK，抛弃中间层。最终整个集团体系下所有的 App 的架构应该如下面这张图所示：

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/架构统一.png" width = "60%" alt="中间层设计" align=center /></div>
统一的平台层，不同的宿主搭配上不同的垂直业务，就是不同的 App。这一点还需要我们持续迭代才能做到。

## 三. 收获和感悟

整个木星计划下来，个人有很多的收获和感悟。说实话，这次的技术改造在技术上并没有太大的难度，更谈不上有什么「黑科技」。改造能顺利落地，更多的对于全局的把控、资源的协调沟通以及各个兄弟团队的积极配合。所以这里抛开技术不谈，谈谈我的几点收获：规范化、流程化和全局视角。

### 3.1 规范化、流程化

安居客 Android App 团队在技术上过往基本都属于小团队作战，很少有跨团队、跨地域协同开发的经验，也缺少和集团其它团队的交流，因此规范和流程一直是我们团队所欠缺的。

在之前的小团队开发模式下，就这么一亩三分地，想怎么玩都行，怎么方便怎么来。但是这种方式一旦涉及到跨地域跨团队的协作时就会遇到瓶颈。

比如北京租房团队的需求开发完要集成进安居客，按照我们单纯的想法，定个时间点提交代码给我们就 OK 了，但实际情况是这种方式根本行不通。业务方什么时候开发完、测试完、什么时候交付给我们？业务集成的交付标准是什么？平台方测试和业务方测试如何配合、如何交接？业务代码是以源码方式集成还是 aar 方式集成等等这些都是问题，需要有一套标准化、流程化的规范来约束各方的行为，这样才能保证项目顺利上线。

像上面这样的例子还有很多，我这里就不一一列举了。

### 3.2 全局视角

正所谓**「不谋全局者，不足谋一域」**，本次平台化改造过程中我的另外一个重大收获是让我意识到要以全局视角去看到问题。

前面提到我们在规范和流程上有很多欠缺，很多问题单从自身无法解决，这促使我跳出自己的圈子来思考问题。我们做 App 开发的同学往往容易把视角局限于自己负责的一个页面、一个功能模块、一个业务，能站在整个 App 的角度看待问题的已经寥寥无几了，更别提跳出 App 的视角来看待问题。

但缺少全局视角，很多事情就不能很好的完成。举个例子，假设你接到了一个优化页面响应速度的任务，如果你单从 App 的角度入手你会发现能做的很有限，当你一通操作把 App 的优化点做完了却发现中台部门提供的 SDK 方法耗时两秒，API 返回数据耗时三秒，真的是**一顿操作猛如虎，一看结果二百五**。

上面提到的这个例子还只是最常见、最基础的一点。真正的全局视角是要求我们跳出 App 的限制，去思考整个研发流程的痛点、跨团队协作上还有哪些优化空间等等，这样才能真正提升我们的开发效率、产品性能和用户体验。就拿我业余时间里一直在做的 APM 项目来说，如果只从 App 的角度来看，要实现对线上性能数据的采集会涉及到 Gradle Plugin、字节码、ASM、数据的采集、存储、上报、跨进程通讯等等，每一项都不简单，每一项需要有一定的技术深度才能做好一个 APM 组件。那么是不是每一项都做好了，实现了一个优秀的 APM 组件就能解决线上性能问题、提升用户体验呢？显然不是！

如果你站在更高的角度来看，一个单纯的 APM 组件并没有办法解决任何性能问题。我们需要从前期开发阶段的开发规范和实现方案、QA 验收标准、上线后的性能数据采集、数据上报后的聚合、分析和报警、性能问题的处理流程和规范等各个维度来协同处理，全方位的系统的思考每一个点，从更高的角度入手才能真正解决线上性能问题。

#### 登高望远

现在我们把视角再往上拔高一个层次，站在整个技术团队的角度来看移动开发。

安居客 App 是市面上一类比较典型的互联网应用，它包含了基本的业务呈现、即时通讯、视频播放、直播等面向用户的功能模块，也包含了数据采集、日志记录、网络通讯等用户看不见的功能模块，同时还需持续交付平台、测试平台等等平台的支持。

早期，即使在同一家公司这些内容各个 App 都是独立的，各自自己做一套；

后来慢慢发现这种各自为战的模式效率太低，重复造轮子的现象太严重，于是有了平台化的概念。即时通讯是一个平台、视频直播是一个平台、日志系统又是一个平台，持续交付、测试均是单独的平台。各个平台为各种 App 提供不一样的服务，各个 App 单独对接各个平台就可以了，避免了重复造轮子。

这种平台化方案也有它自己的问题，通过前面的描述你会发现，基本上来一个新业务就要开发一个新系统，形成一个新平台，这样也会给 App 的接入带来困扰，同时不同的平台自然会有不同的团队，这之间的沟通协作成本是巨大的。于是更进一步把各种分散的平台统一成一个更大的平台，统一对各种 App 提供服务。

这就是整个 App 研发体系从蛮荒时代到平台化，再从平台到中台的完整进化史。

> 这里说的平台化和文章标题里的平台化不是同一个概念，这里的平台化是指一套系统作为一个平台为各种 App、Web、小程序等前台应用提供服务；而文章标题里的平台化是指安居客 App 作为一个平台来支持各类房产业务。

那么现在我们再来看研发团队的组织结构，就完全不一样了。请看下图：

<div align="center"><img src="https://resources.baronzhang.com/architecture/平台化/全局视角.png" width = "55%" alt="中间层设计" align=center /></div>

当我们能站在这样一个角度看团队的组织结构、研发流程的时候，很多事就更容易理解了。比如中台部门推出了一套日志系统，各个前端团队要不要替换掉自研的埋点库，使用中台部门的服务，我的看法是当然要。让专业的团队做专业的事，中台为各个前端业务团队赋能，无论是质量上还是效率上都会有极大的提升。同时这样也便于对各业务线的用户数据、行为做统一的聚合、分析、报警等等，然后进一步反哺业务。

这也是为什么之前集团 TEG 团队推出 WMDA（58集团埋点系统）后，我们要顶住巨大压力在安居客内部推广的原因。

## 写在最后

平台化改造能顺利完成并非我们一个团队的功劳，这得益于前后端、产品、测试、58 无线、58 房产等多个团队积极的配合，就比如前面介绍的很多技术方案都是 58 无线团队的同学提出并开发的，因此要在这里说一声感谢，我们从兄弟部门学到了很多。

这次平台化改造的过程中涉及了太多的内容，其中每一个点都能拿出来单独写一篇文章，由于篇幅限制并不能在文中一一详述。我们团队在平台化方面的实践上还缺乏足够的经验，个人能力也有限，未能将细节很好的一一呈现。如果大家发现文章中的错误或者实现方案上的不完美，欢迎在评论区留言交流指正。

------

> 如果你喜欢我的文章，就关注下我的公众号 **BaronTalk** 、 [**知乎专栏**](https://zhuanlan.zhihu.com/baron) 或者在 [**GitHub**](https://github.com/BaronZ88) 上添个 Star 吧！
>
> - 微信公众号：**BaronTalk**
> - 知乎专栏：[https://zhuanlan.zhihu.com/baron](https://zhuanlan.zhihu.com/baron)  
> - GitHub：[https://github.com/BaronZ88](https://github.com/BaronZ88)

<div align="center"><img src="https://resources.baronzhang.com/blog/common/gzh3.png" width="85%"/></div>
