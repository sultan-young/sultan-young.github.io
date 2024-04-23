---
title: 【译】angular中的ng-template, ng-content, ng-container
categories: [前端, Angular]
tags: [Angular]
date: 2022-09-15
---
![image-20220915142702595](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/image-20220915142702595.png)


> 本文会包含作者一些理解，为了不和原作者意思混淆，会将个人的理解放在【】里。

[原文地址](https://www.freecodecamp.org/news/everything-you-need-to-know-about-ng-template-ng-content-ng-container-and-ngtemplateoutlet-4b7b51223691/)

那是我忙于为我的办公项目创作新功能的日子之一。突然间，某些事情引起了我的注意。
<!--more-->

![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/f2b2dded8281436984afa86e09ef43d9~tplv-k3u1fbpfcp-watermark.png)
上图是angular最终渲染出来的DOM

当我审查DOM时候，我看到这个 `ngcontent` 被Angular应用到了元素上。如果他们包含元素在最终的DOM里，那么`<ng-container>`是用来干什么的？ 那是我很困惑在 `<ng-container>` 和 `<ng-content>`中。

在探寻问题答案的时候我发现了`<ng-template>`和 `*ngTemplateOutlet`的概念。现在我们有四个听起来几乎相同的概念。我开始了探寻它们的旅程。

你曾经也遇到过这种情况吗？如果是，那么你来对地方了。闲话少说，让我们一个一个来讲解他们。

## < ng-template >

顾名思义，`<ng-template>` 是一个模板元素，用于Angular与结构指令结合使用(`*ngIf`, `*ngFor`, `[ngSwitch]`和自定义指令)

<font color='red'>这些模板元素仅工作在结构指令存在的时候</font>。Angular包装了我们应用指令的元素。 考虑下边的一个`*ngIf`的例子。

![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/f200a620eb584e6885af8a100b617f85~tplv-k3u1fbpfcp-watermark.png)

【我们可以看到，在一个普通元素上添加结构指令，angular在解析时候，其实会帮我买创建一个ng-template元素，并将相关的结构指令放到ng-template上进行处理】

上面展示的是Angular对与`*ngIf`的解释。 最终的DOM与我们在本文开头看到的类似。

![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/25797712498342329327e21123dff105~tplv-k3u1fbpfcp-watermark.png)

### 使用方法

我们已经看到了Angular使用`<ng-template>`的方式。但是我们如何使用它呢？由于它只和结构指令一起工作，所以我们可以这样使用：【这种是❌方式，后边会讲到】

![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/dc58bedd21ed447294bc146d15beaf16~tplv-k3u1fbpfcp-watermark.png)

此处的 `home` 设置为 `true`， 上述代码在DOM中的输出为：

![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/52802691912043f7b201bb8256a346dd~tplv-k3u1fbpfcp-watermark.png)

什么都没有被渲染出来！这是为什么呢？ 其实，这正是预期的结果。正如我们已经讨论过的，Angular会用注释替代`ng-template`【译文使用的Angular版本为6.1.10。 在Angular12版本，已经用特殊的className代替注释了】，我们的代码最终会被解释成如下代码。

![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/a7da154d24914dc6b23db7a72232a373~tplv-k3u1fbpfcp-watermark.png)

我们的`<ng-template>`在被Angular包装之后，变成了两个。但是无论怎样，Angular都不对`<ng-template>`的内容进行选中。

以下是两种<font color=green>正确</font>的使用方式： ![image.png](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/8c723fc237bf48e8ae74ea499ad94a97~tplv-k3u1fbpfcp-watermark.png)



#### Method 1

在第一种方法中，你提供给Angular一种不需要进一步处理的方式。这样Angular将仅仅转换 `<ng-template>` 到注释里，并不会改变其内容。（它们不会再像之前的例子一样被放在任何`<ng-template>`中）。因此，它将正确的渲染内容。

要想了解更多如何使用此结构和更多的结构指令，请参考这篇[文章](https://www.concretepage.com/angular-2/angular-4-ng-template-example#ngSwitch)。

#### Method2

这是一种很少使用的方式（使用两个`<ng-template>`）。这里我们在then中给出了一个模板引用，告诉他当条件为真时候应该渲染什么。

像这样使用多`<ng-template>`（你可以使用`<ng-container>`代替）是不被建议的因为这不是Angular的本意。`<ng-template>`应当被使用在多处重复使用的场景下。我们将在文章后面的部分更详细的讨论。



## < ng-container >

你是否曾经写过或看到过类似这样的代码

![1*xSzfSSecltMEvHbwKoTlhQ](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*xSzfSSecltMEvHbwKoTlhQ.png)

我们大部分人写这样代码的原因在于Angular无法再单个元素上使用多个结构指令。现在这个代码正常工作，但是如果item.id 为false，它就会在dom中引入很多空的`<div>`

![1*EZDOC5gDjhx0y-2pMGgP1A](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*EZDOC5gDjhx0y-2pMGgP1A.jpeg)



在简单的例子中可能不需要关心它，但如果在大型复杂的应用中时候（会展示成千上万的数据），它可能变得麻烦，因为有可能会有监听器在这些DOM上。

更糟糕的是你可能必须去嵌套你的CSS。【大概是想表达这些写会在附加样式时候需要额外的选择器】



不要担心！我们有`<ng-container>`可以解决。



`<ng-container>`是一个组元素，它不会干扰样式或布局，因为Angular不会将它们渲染到DOM中。



![1*j-TJRTA11OrLKdLrmrjQjA](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*j-TJRTA11OrLKdLrmrjQjA.png)





以上代码会渲染出这样的DOM结构

![1*7D-if7f35ct3vkY3AnozUQ](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*7D-if7f35ct3vkY3AnozUQ.jpeg)

看，我们摆脱了那些空的div。我们应该使用`<ng-container>`当我们仅仅想使用多个结构指令而不想引入多的DOM时候。



## < ng-content >

它用于创建可配置的组件。这意味着组件可以根据用户的意愿来配置。这就是众所周知的内容投影~



考虑一个简单的使用了`<ng-content>`的组件

![1*gzHVRbeW6JYv3XUxX5tQRA](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*gzHVRbeW6JYv3XUxX5tQRA.png)

![1*HIp3l46s5LRIPS8Cs3ZPlg](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*HIp3l46s5LRIPS8Cs3ZPlg.png)

在`<project-content>`组件的开始标记和结束标记之间是将要被内容投影的内容。【也就是vue和react的slot】

这些内容将被渲染在组件的`<ng-content>`中，这将允许自定义`<project-content>`组件的页脚部分。





### 多重投影

如果你可以决定哪些内容可以被渲染到哪些地方会怎样？您还可以使用`<ng-content>`的select属性来控制内容的投影方式，而不是在单个的`<ng-content>`中投影每个内容。它需要一个元素选择器来决定特定的`<ng-content>`中投影哪些内容.



方法如下：

![1*G6Ruc21MJctpiYqkdD5DjQ](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*G6Ruc21MJctpiYqkdD5DjQ.png)

我们修改`<project-content>`来演示如何使用多内容投影。Select属性选择器将在特定的`<ng-content>`中呈现的内容。这里我们首先选择呈现h1元素，如果投射的内容中没有h1元素，它将不会渲染任何内容。同样，第二个选择查找div。其余的内容在最后一个`<ng-content>`中呈现。



> 以上的方式应该是老版本的Angular了，现在的Angular文档中没有提到，现在的使用方式请查看[最新文档](https://angular.cn/guide/content-projection)



## *ngTemplateOutlet

`*ngTemplateOutlet` 一般被用在两个场景：

1. 不论循环或条件如何，在视图中插入一个公共模板
2. 创建一个高度配置的组件



### 模版重用

考虑一个视图，其必须在多个位置插入模板。例如，要放置在网站中的公司logo。我们可以通过为logo编写一次模板并在视图的任何地方重用来实现它。

下面是代码片段：

![1*M2mxgv1g3VcftdHOFFmTdw](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*M2mxgv1g3VcftdHOFFmTdw.png)

正如你看到的，我们只写了一次模板代码，并使用了它三次！

`<*ngTemplateOutlet>` 也接受一个上下文对象，可以传递改对象来自定义通用模板输出。有关上下文的更多信息，请参阅[官方文档](https://angular.io/api/common/NgTemplateOutlet)



### 自定义组件

`<*ngTemplateOutlet>` 的第二个使用方式是高度订制组件。考虑我们之前的`<project-content>` 的示例。并进行一些修改。

![1*AwPv-pFH7e-Abhr-odvPyQ](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*AwPv-pFH7e-Abhr-odvPyQ.png)



上面是`<project-content>` 组件的修改版本，它接受三个属性 —— `<headerTemplate>` `<bodyTemplate>` `<footerTemplate>` 。下面是其代码片段：



![1*KHFWhtDmaysZMxGDTT_61Q](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*KHFWhtDmaysZMxGDTT_61Q.png)

我们在这里视图实现的是显示从`<project-content>`的父组件接受到的页眉，正文和页脚。如果其中任何一个未提供，我们的组件在其位置展示默认模版。因此，创建了一个高度订制的组件。

现在使用我们刚创建的组件：

![1*13rIyei1HqPdOsQ44l9tug](https://cdn.jsdelivr.net/gh/sultan-young/picture-bed/assets/1*13rIyei1HqPdOsQ44l9tug.png)

这是我们如何传递一个 template refs 给我们的组件的方式。如果其中任何一个没有传递，组件将渲染默认的模板！



## ng-content vs *ngTemplateOutlet

它们都帮我们实现了高度自定义的组件，但是我们该如何选择它们呢？

可以清除的看到，`*ngTemplateOutlet`给了我们更多的能力，例如提供默认的模板...

但 `<ng-content>` 智能按照原样呈现内容，借助select属性，您可以拆分内容并在视图的不同位置呈现它们。您不能有条件的呈现`<ng-content>`的内容。您必须限制从父组件哪里接受到的内容，而无法根据内容做出决定。



无论如何，在这两个间进行选择完全取决于你的用例。至少现在我们有了一个新的武器 `*ngTemplateOutlet`，它提供了更多的控制权对比`<ng-content>`
