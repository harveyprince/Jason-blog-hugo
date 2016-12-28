+++
math = false
date = "2016-12-15T10:18:34+08:00"
title = "Angular1 Source Code Overview"
tags = [
    "analyze",
    'source code',
    'AngularJS'
]
image = ""

+++

基于版本：1.6.0

## 概念解读
---
先来搬运一下官方的doc

其中介绍了一些框架内的重要概念
<!--more-->

|Concept|Description|
|:------ |:------ |
|Template|HTML with additional markup|
|Directives|extend HTML with custom attributes and elements|
|Model|the data shown to the user in the view and with which the user interacts|
|Scope|context where the model is stored so that controllers, directives and expressions can access it|
|Expressions|access variables and functions from the scope|
|Compiler|parses the template and instantiates directives and expressions|
|Filter|formats the value of an expression for display to the user|
|View|what the user sees (the DOM)|
|Data Binding|sync data between the model and the view|
|Controller|the business logic behind views|
|Dependency Injection|Creates and wires objects and functions|
|Injector|dependency injection container|
|Module|a container for the different parts of an app including controllers, services, filters, directives which configures the Injector|
|Service|reusable business logic independent of views|

## 这些概念存在的意义
---
### 一个简单的例子

<embed width="100%" height="500" src="https://embed.plnkr.co/beedjLQy1AqlCcgcixuN/" />

这看起来很像普通的html，加上一些新的标记。在angular中，以这样的形式组织的语句叫做模版。
当Angular启动应用时，它会使用`compiler`来转化和处理`template`中的新标记。
加载、转化和渲染后的DOM称之为`view`。

第一种新标记称之为`directives`。它们将特殊的行为应用到HTML中的attributes或者elements中。
在这个例子中，我们使用了`ng-app`，它用来连接一个`directive`来实例化我们的应用。
Angular自身还为`input`元素定义了一个directive来扩展该元素的行为。
`ng-model`这个directive用来在变量和input filed中同步数据展示，即一个双向绑定。
值得注意的是，在Angular中，`directive`是唯一一个可以直接接触DOM的地方。

第二种新的标记形如`{{ expression | filter }}`，当compiler遇到这样的标记时，会用最终的计算结果来代替它。
expression是一个js形式的代码语句，可以读写在该`scope`中存活的变量。存储在scope中的变量称之为`model`。
这个例子还包含了一个`filter`，一个过滤器将数据格式化后展现给用户。

![concepts-databinding1](/blog/img/post/AngularjsSourceCodeOverview/concepts-databinding1.png)

这个例子展现了Angular的一个重要的绑定概念：无论何时当input的value发生了变化，expression中的value也会自动重新计算，
并且将结果更新到DOM中去，这个概念称之为双向数据绑定。

### 一个controller的例子

<embed width="100%" height="500" src="https://embed.plnkr.co/ICBcjB5SPnwqOUaegQQg/" />

这里我们引入了`controller`，使用controller的目的是为了将一些变量和功能性的方法暴露给`expressions`和`directives`。
我们在HTML中使用了`ng-controller`这个directive，这个directive告诉Angular一个新的`InvoiceController`负责处理存在该directive的元素以及其下的所有子元素。
语句`InvoiceController as invoice`告诉Angular实例化该controller并且用变量`invoice`来存储到当前的scope中。
于是可以在template里通过invoice来调用controller的变量和方法。

![concepts-databinding2](/blog/img/post/AngularjsSourceCodeOverview/concepts-databinding2.png)

### service：与视图无关的业务逻辑

<embed width="100%" height="500" src="https://embed.plnkr.co/bJ7Ci5uRFycAnuup7n9I/" />

在上一个例子中，`controller`包含了所有的逻辑。然而当我们的应用不断增长壮大之后，一个良好的实践是将与视图无关的业务逻辑
抽离到`service`中去，这使得这些方法可以被应用的其他部分进行复用。
我们将`convertCurrency`和相关的定义移到了service中，但是controller是怎样获取并使用到一个已经分离出去的方法的呢？

这就是`Dependency Injection`发挥作用的地方了。
DI是一种软件设计模式，用来处理对象和方法的创建以及确保它们得到它们的依赖。
Angular中的一切（directives,filters,controllers,services,...）都是通过DI创建并且管理的。
在Angular里，DI的容器称之为`Injector`。

![concepts-module-injector](/blog/img/post/AngularjsSourceCodeOverview/concepts-module-injector.png)

为了使用DI，需要有一个地方使得所有事物一起工作并且注册，这就是在Angular中使用`modules`的目的。
当Angular启动时，它会使用具有`ng-app`这一directive定义的名称的module的配置，
同时包含了该module所依赖的所有modules的配置。

在上述的例子中，这个template包含了directive `ng-app="invoice2"`。这告诉Angular使用`invoice2`
module作为本应用的主module。`angular.module("invoice2",["finance2"])`表明了`invoice2`module依赖于
`finance2`模块，通过这个，Angular使用`InvoiceController`以及`currencyConverter`服务。

现在Angular知道了应用的所有部分，需要去创建它们。在前面我们知道controllers通过构造方法来创建。
至于services，有很多种方法来指定它们的创建方式，在上述的例子中，我们使用`factory`来创建`currencyConverter`，
此函数应该要返回currencyConverter service实例。

> 这里牵扯到 factory和service的区别，factory最终需要显式返回一个实例，而service会以构造函数的形式创建服务

![concepts-module-service](/blog/img/post/AngularjsSourceCodeOverview/concepts-module-service.png)

回到最初的问题，InvoiceController是怎样获得currencyConverter的引用的？
在Angular中，通过简单的在构造方法中定义arguments就可以做到。通过这种方式，`injector`就能够以正确的顺序
创建对象，并且将先前创建的对象传递到依赖于它们的对象的工厂中。在上述的例子中，InvoiceController有一个argument叫做
currencyConverter，通过这个，Angular就知道了controller和service之间的依赖关系，并且通过该service的实例来调用controller的构造函数进行实例化。

至于我们在module.controller中传递了一个数组，是为了使得代码在经过压缩和uglify之后仍然能够正常工作的机制，因为这样arguments可能会被`a`这样的变量名替代。

### Data Binding
数据绑定在传统的模板系统中

![One_Way_Data_Binding](/blog/img/post/AngularjsSourceCodeOverview/One_Way_Data_Binding.png)

数据绑定在Angular模板系统中

![Two_Way_Data_Binding](/blog/img/post/AngularjsSourceCodeOverview/Two_Way_Data_Binding.png)

但也因此要特别注意不要造成死循环