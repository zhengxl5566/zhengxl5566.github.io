---
layout: post
title:  "Freemarker 教程(一)-模板开发手册"
date:   2021-2-2 08:00:00 +0800
category: Java
author: Java课代表
excerpt: 本文是Freemarker系列的第一篇，面向模板开发人员，主要介绍 FreeMarker 所使用的 FTL语法，了解 Freemarker 的基本概念
---

本文是Freemarker系列的第一篇，面向模板开发人员，主要介绍 FreeMarker 所使用的 FTL(FreeMarker Template Language) 语法，了解 Freemarker 的基本概念，介绍基本的 FTL 术语 及内置函数，内置指令，方便作为开发手册速查（文中演示所用版本为 2.3.30，实际使用中请根据自己项目版本自查[官网](https://freemarker.apache.org/)）。

本文不会罗列官网API，只在必要时演示其语法，代码工程中有课代表整理的 freemarker api 思维导图，配合此文食用可使功力大增！请到 [课代表的 github](https://github.com/zhengxl5566/springboot-demo)自取。

## 1.FreeMarker 是什么

`Freemarker`是一款纯 `Java`编写的模板引擎软件，可以用来生成各种文本，包括但不限于：`HTML`，`E-Mail`以及各种源代码等等。

它的主要任务就是：把模板和数据组装在一起，生成文档，这个过程又叫渲染(Render)。流程如图：

![freemarker-overview.png](https://zhengxl5566.github.io/img/article-img/2021-2/freemarker-overview.png)

由于大部分模板开发人员都是用它来生成HTML页面，所以本文将基于 `SpringBoot(2.4.1)+Freemarker(2.3.30)+SpringWeb`演示 HTML 页面的渲染

## 2.最简单的模板

假设我想要一个简单页面用来欢迎当前用户，模板代码：
```html
<html>
<head>
  <title>index</title>
</head>
<body>
    <p>你好，${userName}</p>
</body>
</html>
```

`${userName}`是 FTL 的**插值**语法，他会把`userName`的值替换到生成的 `HTML` 中，从而根据当前登录者，显示不同的用户名，这个值由后端代码放到Model中，对应的 Controlelr 代码：

```java
@Controller
public class HelloWorld {
    @GetMapping("hello")
    public String hello(Model model) {
        model.addAttribute("userName","Java 课代表");
        // 返回模板名称
        return "index";
    }
}
```

访问页面：
![freemarker-hello.png](https://zhengxl5566.github.io/img/article-img/2021-2/freemarker-hello.png)

数据由后端代码通过数据模型（Model）传递，模板只关心数据如何展示（View），二者的关联关系由 Controller 来控制，这就是 MVC。

## 3.数据模型(data-model)

Controller中添加到 model 中的数据是如何组织的呢？这就需要了解一下FTL的数据模型(data-model)。

 FTL 的数据模型在结构上是一个树形：

```json
(root)
  |
  +- animals
  |   |
  |   +- mouse
  |   |   |
  |   |   +- size = "small"
  |   |   |
  |   |   +- price = 50
  |   |
  |   +- elephant
  |   |   |
  |   |   +- size = "large"
  |   |   |
  |   |   +- price = 5000
  |   |
  |   +- python
  |       |
  |       +- size = "medium"
  |       |
  |       +- price = 4999
  |
  +- message = "It is a test"
  |
  +- misc
      |
      +- foo = "Something"
```
其中的`root` 可以理解为 Controller 中的 model ，通过 `model.addAttribute("userName","Java 课代表");`就可以往数据模型中添加数据。

数据模型中可以像目录一样展开的变量，如：`root, animals, mouse, elephant, python, misc`称之为**哈希(hash)**。哈希的 key 就是变量名，value 就是变量存储的值，通过`.`分隔的路径可以访问变量值，比如访问 mouse 的 price :`animals.mouse.price`.

像`animals.mouse.price`这样存储单个值的变量叫做**标量(scalar)**，标量有四种具体类型：string，boolean，date-like，number；

还有一种变量叫做**序列(sequence)**,可以类比为 Java 中的数组，序列中的每个项没有名字，可以通过遍历，或者下标的方式访问（后面会演示序列的访问），它的数据结构看起来是这样的：
```java
(root)
  |
  +- animals
  |   |
  |   +- (1st)
  |   |   |
  |   |   +- name = "mouse"
  |   |   |
  |   |   +- size = "small"
  |   |   |
  |   |   +- price = 50
  |   |
  |   +- (2nd)
  |   |   |
  |   |   +- name = "elephant"
  |   |   |
  |   |   +- size = "large"
  |   |   |
  |   |   +- price = 5000
  |   |
  |   +- (3rd)
  |       |
  |       +- name = "python"
  |       |
  |       +- size = "medium"
  |       |
  |       +- price = 4999
  |
  +- misc
      |
      +- fruits
          |
          +- (1st) = "orange"
          |
          +- (2nd) = "banana"
```

FTL 里**常用的数据类型**就这三类：哈希（hashe）,标量（scalar），序列（sequence）。

有了数据，还要有语法来组织这些数据，下面介绍 FTL 中的常用语法。

## 4.FTL 语法

FreeMarker 只认如下三种语法：
1. 插值：${...} ，Freemarker 会将里面的变量替换为实际值
2. FTL 标签(tags)：结构上类似HTML的标签，都是用`<>`包裹起来，普通标签以`<#`开头，用户自定义标签以`<@`开头，如`<#if true>true thing<#/if>`，`<@myDirect></@myDirect>`
> 你会看到两种叫法，1：标签（tags），2：指令（directive）。举个例子：`<#if></#if>`叫标签； 标签里面的`if`是指令，可以类比于html中的标签（如：`<table></table>`）和元素（如：`table`）。不过，把标签和指令认为是**同义词**也没有问题。

3. 注释(Comments)：FTL 中的注释是这样的：`<#-- 被注释掉的内容 -->`，对于注释，FTL会自动跳过，所以不会显示在生成的文本中（这点有别于 HTML 的注释）

> 注意：除以上三种语法之外的所有内容，皆被 FreeMarker 视为普通文本，普通文本会被原样输出

插值就是单纯的替换变量的值，注释更没啥好说的，下面主要介绍几个**最常用**的 FTL 标签（指令）并结合代码演示其用法。

### if 指令

if 可以根据条件跳过模板中的某块代码，以前文为例，当 `userName` 值为 "Java课代表" 或`zhengxl5566`时，用特殊样式展示，相关模板代码如下：
```html
<p>你好，
    <#if userName == "Java 课代表">
        <strong>${userName}</strong>
    <#elseif userName == "zhengxl5566">
        <h1>${userName}</h1>
    <#else>
        ${userName}
    </#if>
</p>
```
### list 指令

list用来遍历序列，其语法为：

```html
<#list sequence as loopVariable>
    repeatThis
</#list>
```
比如后台往 model 里放入一个 allUsers 的集合
```java
model.addAttribute("allUsers",userService.getAllUser());
```
可以直接使用下标访问集合中的某个元素：`${allUsers[0].name}`

也可以在模板中直接遍历展示：
```html
<ol>
<#list allUsers as user>
    <li>
        姓名：${user.name}，年龄：${user.age}
    </li>
</#list>
</ol>
```
实际渲染出来的 HTML：
```html
<ol>
  <li>
    姓名：zxl，年龄：18
  </li>
  <li>
    姓名：ls，年龄：19
  </li>
  <li>
    姓名：zs，年龄：16
  </li>
</ol>
```

> 注意：假设 allUsers 是空的，渲染出来的页面会是`<ol></ol>`，如果需要规避这个情况，可以使用 items 标签

```html
<#list allUsers>
    <ol>
        <#items as user>
            <li>
                姓名：${user.name}，年龄：${user.age}
            </li>
        </#items>
    </ol>
</#list>
```
此时，假设 allUsers是空的，list 标签中的 html 内容就不会被渲染出来。

### include 指令

include 指令可以把一个模板的内容插入到另一个模板中（官方建议使用 import 代替，参见下文的最佳实践）。
假设我们每个页面都需要一个 footer，可以写一个公共的footer.ftlh模板，其余需要footer的页面只需要引用footer.ftlh模板即可：

```html
<#include "footer.ftlh">
```
### import 指令

import 可以将模板中定义的变量引入当前模板，并在当前模板中使用。它和 include 的主要区别就是 import 可以将变量封装到新的命名空间中（后文会介绍 import 和 include 的对比）。

例如：模板 /libs/commons.ftl 里面写了很多公共方法，想在其他模板里引用，只需要在其他模板的开头写上：

```
<#import "/libs/commons.ftl" as com>
```

后续想使用/libs/commons.ftl 中的 copyright 方法，可以直接使用：

```html
<@com.copyright date="1999-2002"/>
```

### assign 指令

assign 可以用来创建新的变量并为其赋值，语法如下：

```html
<#assign name1=value1 name2=value2 ... nameN=valueN>
or
<#assign name1=value1 name2=value2 ... nameN=valueN in namespacehash>
or
<#assign name>
  capture this
</#assign>
or
<#assign name in namespacehash>
  capture this
</#assign>
```

举例：

```html
<#--创建字符串-->
<#assign myStr = "Java 课代表">
<#--使用插值语法显示字符串-->
myStr:${myStr}
```

### macro 指令

macro 用来从模板上创建用户自定义指令（Java后端可以通过实现`TemplateDirectiveModel`接口自定义指令，将在下一篇：《Freemarker 教程(二)-后端开发指南》中介绍）

macro 创建的也是变量，该变量可以做为用户自定义指令使用，比如下面的模板定义了 `greet`指令：

```html
<#macro greet>
  <h1>hello 课代表</h1>
</#macro>
```

使用 greet 指令

```html
<@greet></@greet>
或者
<@greet/>
```

指令还可以附带参数：

```html
<#macro greet person>
  <h1>hello ${person}</h1>
</#macro>
```

使用时传入 person 变量：

```html
<@greet person="Java课代表"/> and <@greet person="zhengxl5566"/>
```

## 5.内置函数（Built-ins）

所谓内置函数，就是 FreeMarker 针对不同数 据类型为我们提供的一些内置方法，有了这些方法，可以让数据在模板中的展示更加方便。使用内置函数时，只需要在变量后面使用`?`加相应函数名即可。篇幅有限，这里不打算罗列所有内置函数，只挑几个简单例子展示其语法。

### 例子一：字符串的内置函数

```html
<#--创建字符串-->
<#assign myStr = "Java 课代表">
<#--首字母小写-->
${myStr?uncap_first}
<#--保留指定字符后面的字符串-->
${myStr?keep_after("Java")}
<#--替换指定字符-->
${myStr?replace("Java","Freemarker")}
```

### 例子二：时间类型内置函数

```html
<#--获取当前时间(如果是后端将时间传入data-model，只需要传Date类型即可)-->
<#assign currentDateTime = .now>
<#--展示日期部分-->
${currentDateTime?date}<br>
<#--展示时间部分-->
${currentDateTime?time}<br>
<#--展示日期和时间部分-->
${currentDateTime?datetime}<br>
<#--按指定格式展示时间日期-->
${currentDateTime?string("yyyy-MM-dd HH:mm a")}<br>
```

### 例子三：序列的内置函数

```html
<#--序列类型内置函数样例-->
<#assign mySequence = ["Java 课代表","张三","李四","王五"]>
<#--将所有元素以指定分隔符分割，输出字符串-->
${mySequence?join(",")}<br>
<#--取序列中的第一个元素-->
${mySequence?first}<br>
<#--将序列排序后使用逗号分割输出为字符串-->
${mySequence?sort?join(",")}<br>
```

通过以上三个例子的简单演示，相信你已经能掌握内置函数的使用技巧了，就是在变量后面用 `?`加变量数据类型所支持的函数。FTL 的内置函数极其丰富，官网按数据类型详细罗列了各自支持的内置函数及其用法，可自行查看[官网的内置函数参考](https://freemarker.apache.org/docs/ref_builtins.html)。

为了方便大家快速查阅相关内置函数(built-ins)和指令(directives)，课代表从官网翻译，并使用 xmind 做了个思维导图，每个函数(指令)都可以点进去查看功能描述和样例，可以极大提高模板开发效率：

![freemarker-xmind.png](https://zhengxl5566.github.io/img/article-img/2021-2/freemarker-xmind.png)

原始 xmind 文件放在 [课代表的github上](https://github.com/zhengxl5566/springboot-demo)，读者可以按需自取。

> 课代表划重点！这个思维导图是全文精华，一定要下载下来看看！



## 6.命名空间(Namespaces)

所谓命名空间，就是在同一个模板里，所有使用 assign，macro，function 指令所创建的变量集合，它的主要作用就是唯一标识一个变量。

有两种方式可以创建命名空间：

1、同一个模板中的变量在同一个命名空间中。

以如下的`index.ftlh`为例，这里面创建的变量都在一个命名空间下，同名变量的值会相符覆盖

```html
<#assign myName = "Java 课代表">
<#assign myName = "课代表">
<#--实际输出的是“课代表”-->
${myName}
```

2、不同模板中的变量可以通过 import 指令来区分不同的命名空间变量

模板A中想使用模板B中的变量，可以使用 import 指令，给引入的模板定义一个新的命名空间，通过 `as` 后面指定的 key 访问该新命名空间。

比如模板 `lib/example.ftlh`中定义了`copyright`：

```html
<#macro copyright date>
  <p>Copyright (C) ${date} Someone. All rights reserved.</p>
</#macro>

<#assign mail = "user@example.com">
```

在另一个模板`index.ftlh`中使用`copyright`：

```html
<#import "lib/example.ftlh" as e>
<@e.copyright date="1999-2002"/>
${e.mail}
```
### 命名空间的生命周期（The life-cycle of namespaces）

命名空间由 import 指令中的 path 确定（绝对路径），如果相同的 path 引入了多次，只有第一次调用 import 的时候才会触发相应命名空间的创建。后面对于相同模板路径的 import，指代的都是同一个命名空间，举例：

```html
<#import "/lib/example.ftl" as e>
<#import "/lib/example.ftl" as e2>
<#import "/lib/example.ftl" as e3>
${e.mail}, ${e2.mail}, ${e3.mail}
<#assign mail="other@example.com" in e>
${e.mail}, ${e2.mail}, ${e3.mail}
```

将`/lib/example.ftl` `import` 后赋给 `e,e2,e3`三个命名空间，修改`e`中的 `mail`， 对`e2,e3`同样生效

输出：

```html
user@example.com, user@example.com, user@example.com
other@example.com, other@example.com, other@example.com
```

### import 和 include 的区别

`<#import "lib/example.ftlh" as e>` 会创建一个全新的命名空间，并把`lib/example.ftlh`中定义的变量封装到新命名空间中，提供访问，如：`<@e.copyright date="1999-2002"/>`。

`<#include  "lib/example.ftlh">` 只是单纯把`example.ftlh`的内容插入到当前模板，并不会对命名空间产生影响

> Freemarker官方建议：
>
> 所有使用 include 的地方都应该被 import 替代

使用 import 好处如下：

* import 引入的模板只会被执行一次，重复引用多次并不会重复执行模板。而对于 include 而言，每次 include 都会执行一次模板内容；
* import 可以创建该模板的命名空间，引用变量时可以清晰表达出变量来源，减少命名冲突概率；
* 如果定义了很多通用方法，可以将`auto-import`配置为懒加载，用到哪个加载哪个。而`auto-include`无法实现懒加载，必须全量加载；
* import 指令不会有任何输出，而 include 可能会根据模板内容输出相应字符。

## 7.最佳实践

### 1.空值处理

**变量空值的处理**

对于不存在的变量和值为 null 的变量，freemarker 统一认为是不存在的值，对其调用将会报错。为了避免这种情况，可以设置默认值，示例：`<h1>Welcome ${user!"visitor"}!</h1>`，当user不存在时，将显示`visitor`字符串。

另一种方式是在变量后面使用 `??`表达式，如：`user??`如果user存在，则返回 true，否则返回 false，示例：`<#if user??><h1>Welcome ${user}!</h1></#if>`，当`user`不存在时，就不展示欢迎标语了。

**list中的空值处理**

遍历序列的时候，假设序列中有空值，freemarker并不会直接报错或显示空值，而是像更上一级作用域去搜索同名变量值，这种行为可能会导致错误的输出，举例：

```html
<#assign x = 20>
<#list xs as x>
  ${x!'Missing'}
</#list>
```

本意是当遇到序列 xs 中的空元素是显示“missing” 字符串，但由于 freemarker 向上查找的特性， 这里的空值将会显示为 20。要关闭这个特性，可以在服务端配置：`configuration.setFallbackOnNullLoopVariable(false);`

### 2.使用 import 代替 include

前文在介绍到 include 的时候提到过，官方建议应该将所有用到 include 的地方都用 import 实现

我们平时用到 include 指令，主要就是用来把一段内容插入到当前模板，那如何用 import 实现 include 的功能呢？

很简单，把需要插入的内容封装成 自定义指令就好了。

比如我们从common.ftlh 里定义一个自定义指令 myFooter

```html
<#macro myFooter>
    <hr>
    <p>这里是 footer</p>
</#macro>
```

在需要使用的地方，引入 common.ftlh 并调用 myFooter 指令

```html
<#import "lib/common.ftlh" as common>
<#--将include使用import代替-->
<#--<#include "footer.ftlh">-->
<@common.myFooter/>
```

###  3.隔行变色

数据展示的时候经常遇到使用表格展示数据的情况，为了增加辨识度，一般会让奇数行和偶数行颜色不同以区分，这就是隔行变色

下面以遍历序列为例：

```html
<#assign mySequence = ["Java 课代表","张三","李四","王五"]>
<#list mySequence as name>
    <span style="color:  ${name?item_cycle("red","blue")}">   ${name}<br/> </span>
</#list>
    
```

这里的 item_cycle 是循环变量的内置函数，至于他的详细用法，再次推荐你去看一下课代表整理的思维导图。

## 8.总结

本文介绍了 Freemarker 的基本概念和基础语法，意在让刚接触的萌新能对 Freemarker 有个全局性认识，了解Freemarker的数据模型、内置函数、指令。只要能区分这几个概念，实际开发中现用现查即可，不要一开始就迷失在海量的API中。

总体来说，Freemarker 是一个比较简单，容易上手的模板引擎，只要掌握了本文所提及的基本概念，直接上手开发是完全没问题的。

在本文写作过程中，课代表意识到纯文字表达的局限性，又不能全文罗列和翻译API，于是整理了一个思维导图，将Freemarker 的各类指令，内置函数及其官网示例全都翻译整理了进去。在平时开发过程中极大提升了开发效率，需要的同学请到 [课代表的 github](https://github.com/zhengxl5566/springboot-demo)自取。

最后附上思维导图镇楼：

![freemarker-api-xmind.png](https://zhengxl5566.github.io/img/article-img/2021-2/freemarker-api-xmind.png)



