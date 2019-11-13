---
title: 系列 | 深入理解Java虚拟机二（类文件结构）
date: 2019-08-20 00:25:01
categories: JVM
tags: JVM
---
> 首发于微信公众号：**BaronTalk**，欢迎关注！

之前在阅读 ASM 文档时，对于已编译类的结构、方法描述符、访问标志、ACC_PUBLIC、ACC_PRIVATE、各种字节码指令等等许多概念听起来都是云山雾罩、一知半解，原因就在于对类文件结构和类加载机制不够了解。直到后来细读了《深入理解 Java 虚拟机》中虚拟机执行子系统的相关内容，才建立了清晰的认知。如果你也和我一样，不了解类结构和类加载，但是工作中又涉及到字节码相关内容，相信后面两篇文章会对你有所帮助。
<!-- more -->
我们所编写的每一行代码，要在机器上运行最终都需要编译成二进制的机器码 CPU 才能识别。但是由于虚拟机的存在，屏蔽了操作系统与 CPU 指令集的差异性，类似于 Java 这种建立在虚拟机之上的编程语言通常会编译成一种中间格式的文件来存储，比如我们今天要聊的字节码（ByteCode）文件。

## 一. 语言无关性

Java 虚拟机的设计者在设计之初就考虑并实现了其它语言在 Java 虚拟机上运行的可能性。所以并不是只有 Java 语言能够跑在 Java 虚拟机上，时至今日诸如 Kotlin、Groovy、Jython、JRuby 等一大批 JVM 语言都能够在 Java 虚拟机上运行。它们和 Java 语言一样都会被编译器编译成字节码文件，然后由虚拟机来执行。所以说类文件（字节码文件）具有语言无关性。

<div align="center"><img src="https://resources.baronzhang.com/blog/jvm/2/语言无关性.png" width="80%"/></div>

## 二. Class 文件结构

Class 文件是一组以 8 位字节为基础单位的二进制流，各个数据严格按照顺序紧凑的排列在 Class 文件中，中间无任何分隔符，这使得整个 Class 文件中存储的内容几乎全部都是程序运行的必要数据，没有空隙存在。当遇到需要占用 8 位字节以上空间的数据项时，会按照高位在前的方式分割成若干个 8 位字节进行存储。

Java 虚拟机规范规定 Class 文件格式采用一种类似与 C 语言结构体的微结构体来存储数据，这种伪结构体中只有两种数据类型：无符号数和表。

- **无符号数**属于基本的数据类型，以 u1、u2、u4、u8来分别代表 1 个字节、2 个字节、4 个字节和 8 个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照 UTF-8 编码结构构成的字符串值。
- **表**是由多个无符号数或者其他表作为数据项构成的复合数据类型，所有表都习惯性地以「_info」结尾。表用于描述有层次关系的复合结构的数据，整个 Class 文件就是一张表，它由下表中所示的数据项构成。

| 类型           | 名称                | 数量                  |
| -------------- | ------------------- | --------------------- |
| u4             | magic               | 1                     |
| u2             | minor_version       | 1                     |
| u2             | major_version       | 1                     |
| u2             | constant_pool_count | 1                     |
| cp_info        | constant_pool       | constant_pool_count-1 |
| u2             | access_flags        | 1                     |
| u2             | this_class          | 1                     |
| u2             | super_class         | 1                     |
| u2             | interfaces_count    | 1                     |
| u2             | interfaces          | interfaces_count      |
| u2             | fields_count        | 1                     |
| field_info     | fields              | fields_count          |
| u2             | methods_count       | 1                     |
| method_info    | methods             | methods_count         |
| u2             | attributes_count    | 1                     |
| attribute_info | attributes          | attributes_count      |

Class 文件中存储的字节严格按照上表中的顺序紧凑的排列在一起。哪个字节代表什么含义，长度是多少，先后顺序如何都是被严格限制的，不允许有任何改变。

### 2.1 魔数与 Class 文件版本

每个 Class 文件的头 4 个字节称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接收的 Calss 文件。之所以使用魔数而不是文件后缀名来进行识别主要是基于安全性的考虑，因为文件后缀名是可以随意更改的。Class 文件的魔数值为「0xCAFEBABE」。

紧接着魔数的 4 个字节存储的是 Class 文件的版本号：第 5 和第 6 两个字节是次版本号（Minor Version），第 7 和第 8 个字节是主版本号（Major Version）。高版本的 JDK 能够向下兼容低版本的 Class 文件，虚拟机会拒绝执行超过其版本号的 Class 文件。

### 2.2 常量池

主版本号之后是常量池入口，常量池可以理解为 Class 文件之中的资源仓库，它是 Class 文件结构中与其他项目关联最多的数据类型，也是占用 Class 文件空间最大的数据项目之一，同是它还是 Class 文件中第一个出现的表类型数据项目。

因为常量池中常量的数量是不固定的，所以在常量池入口需要放置一个 u2 类型的数据来表示常量池的容量「constant_pool_count」，和计算机科学中计数的方法不一样，这个容量是从 1 开始而不是从 0 开始计数。之所以将第 0 项常量空出来是为了满足后面某些指向常量池的索引值的数据在特定情况下需要表达「不引用任何一个常量池项目」的含义，这种情况可以把索引值置为 0 来表示。

> Class 文件结构中只有常量池的容量计数是从 1 开始的，其它集合类型，包括接口索引集合、字段表集合、方法表集合等容量计数都是从 0 开始。

常量池中主要存放两大类常量：**字面量**和**符号引用**。

- **字面量**比较接近 Java 语言层面的常量概念，如字符串、声明为 final 的常量值等。
- **符号引用**属于编译原理方面的概念，包括了以下三类常量：
  - 类和接口的全限定名
  - 字段的名称和描述符
  - 方法的名称和描述符

### 2.3 访问标志

紧接着常量池之后的两个字节代表访问标志（access_flag），这个标志用于识别一些类或者接口层次的访问信息，包括这个 Class 是类还是接口；是否定义为 public 类型；是否定义为 abstract 类型；如果是类的话，是否被申明为 final 等。具体的标志位以及标志的含义见下表：

<table class="table table-striped" style="width:100%">
  <tr>
    <th>标志名称</th>
    <th>标志值</th>
    <th>含义</th>
  </tr>
  <tr>
    <td>ACC_PUBLIC</td>
    <td>0x0001</td>
    <td>是否为 public 类型</td>
  </tr>
  <tr>
    <td>ACC_FINAL</td>
    <td>0x0010</td>
    <td>是否被声明为 final，只有类可设置</td>
  </tr>
  <tr>
    <td>ACC_SUPER</td>
    <td>0x0020</td>
    <td>是否允许使用 invokespecial 字节码指令的新语意，invokespecial 指令的语意在 JKD 1.0.2 中发生过改变，微聊区别这条指令使用哪种语意，JDK 1.0.2 编译出来的类的这个标志都必须为真</td>
  </tr>
  <tr>
    <td>ACC_INTERFACE</td>
    <td>0x0200</td>
    <td>标识这是一个接口</td>
  </tr>
  <tr>
    <td>ACC_ABSTRACT</td>
    <td>0x0400</td>
    <td>是否为 abstract 类型，对于接口或者抽象类来说，此标志值为真，其它类值为假</td>
  </tr>
  <tr>
    <td>ACC_SYNTHETIC</td>
    <td>0x1000</td>
    <td>标识这个类并非由用户代码产生</td>
  </tr>
  <tr>
    <td>ACC_ANNOTATION</td>
    <td>0x2000</td>
    <td>标识这是一个注解</td>
  </tr>
  <tr>
    <td>ACC_ENUM</td>
    <td>0x4000</td>
    <td>标识这是一个枚举</td>
  </tr>
</table>

access_flags 中一共有 16 个标志位可以使用，当前只定义了其中的 8 个，没有使用到的标志位要求一律为 0。

### 2.4 类索引、父类索引与接口索引集合

类索引（this_class）和父类索引（super_class）都是一个 u2 类型的数据，而接口索引集合（interfaces）是一组 u2 类型的数据集合，Class 文件中由这三项数据来确定这个类的继承关系。

- 类索引用于确定这个类的全限定名
- 父类索引用于确定这个类的父类的全限定名
- 接口索引集合用于描述这个类实现了哪些接口

### 2.5 字段表集合

字段表集合（field_info）用于描述接口或者类中声明的变量。字段（field）包括类变量和实例变量，但不包括方法内部声明的局部变量。下面我们看看字段表的结构：
<table class="table table-striped" style="width:100%">
  <tr>
    <th>类型</th>
    <th>名称</th>
    <th>数量</th>
  </tr>
  <tr>
    <td>u2</td>
    <td>access_flag</td>
    <td>1</td>
  </tr>
  <tr>
    <td>u2</td>
    <td>name_index</td>
    <td>1</td>
  </tr>
  <tr>
    <td>u2</td>
    <td>descriptor_index</td>
    <td>1</td>
  </tr>
  <tr>
    <td>u2</td>
    <td>attributes_count</td>
    <td>1</td>
  </tr>
  <tr>
    <td>attribute_info</td>
    <td>attributes</td>
    <td>attributes_count</td>
  </tr>
</table>

字段修饰符放在 access_flags 中，它与类中的 access_flag 非常相似，都是一个 u2 的数据类型。
<table class="table table-striped" style="width:100%">
   <tr>
    <th>标志名称</th>
    <th>标志值</th>
    <th>含义</th>
  </tr>
  <tr>
    <td>ACC_PUBLIC</td>
    <td>0x0001</td>
    <td>字段是否为 public</td>
  </tr>
  <tr>
    <td>ACC_PRIVATE</td>
    <td>0x0002</td>
    <td>字段是否为 private</td>
  </tr>
  <tr>
    <td>ACC_PROTECTED</td>
    <td>0x0004</td>
    <td>字段是否为 protected</td>
  </tr>
  <tr>
    <td>ACC_STATIC</td>
    <td>0x0008</td>
    <td>字段是否为 static</td>
  </tr>
  <tr>
    <td>ACC_FINAL</td>
    <td>0x0010</td>
    <td>字段是否为 final</td>
  </tr>
  <tr>
    <td>ACC_VOLATILE</td>
    <td>0x0040</td>
    <td>字段是否为 volatile</td>
  </tr>
  <tr>
    <td>ACC_TRANSIENT</td>
    <td>0x0080</td>
    <td>字段是否为 transient</td>
  </tr>
  <tr>
    <td>ACC_SYNTHETIC</td>
    <td>0x1000</td>
    <td>字段是否由编译器自动生成</td>
  </tr>
  <tr>
    <td>ACC_ENUM</td>
    <td>0x4000</td>
    <td>字段是否为 enum</td>
  </tr>
</table>

### 2.6 方法表集合

Class 文件中对方法的描述和对字段的描述是完全一致的，方法表中的结构和字段表的结构一样。

因为 volatile 关键字和 transient 关键字不能修饰方法，所以方法表的访问标志中没有 ACC_VOLATILE 和 ACC_TRANSIENT。与之相对的，synchronizes、native、strictfp 和 abstract 关键字可以修饰方法，所以方法表的访问标志中增加了 ACC_SYNCHRONIZED、ACC_NATIVE、ACC_STRICTFP 和 ACC_ABSTRACT 标志。

对于方法里的代码，经过编译器编译成字节码指令后，存放在方法属性表中一个名为「Code」的属性里面。

### 2.7 属性表集合

在 Class 文件、字段表、方法表中都可以携带自己的属性表（attribute_info）集合，用于描述某些场景专有的信息。

属性表集合不像 Class 文件中的其它数据项要求这么严格，不强制要求各属性表的顺序，并且只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java 虚拟机在运行时会略掉它不认识的属性。


## 写在最后

为了控制篇幅，这篇文章里丢弃了很多细节，比如常量池的项目类型、方法表、属性表的具体内容等等。建议想要深入了解的同学可以自己动手将 Java 类编译成二进制字节码文件，根据文章里介绍的类文件结构逐个字符去对照和实验，有助于加深理解。

关于「类文件结构」我们就介绍到这里，下一篇我们来聊聊「虚拟机的类加载机制」。

**参考资料：**

- 《深入理解 Java 虚拟机：JVM 高级特性与最佳实践（第 2 版）》

---

> 如果你喜欢我的文章，就关注下我的公众号 **BaronTalk** 、 [**知乎专栏**](https://zhuanlan.zhihu.com/baron) 或者在 [**GitHub**](https://github.com/BaronZ88) 上添个 Star 吧！
>
> - 微信公众号：**BaronTalk**
> - 知乎专栏：[https://zhuanlan.zhihu.com/baron](https://zhuanlan.zhihu.com/baron)  
> - GitHub：[https://github.com/BaronZ88](https://github.com/BaronZ88)

<div align="center"><img src="https://resources.baronzhang.com/blog/common/gzh3.png" width="85%"/></div>