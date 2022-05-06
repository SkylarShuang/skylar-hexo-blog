---
title: JS core简介
cover: "https://pixabay.com/get/g17637b9e4cf5365e43a4133b81e33267028da42d02a4dceb80a7aa7e9f7ebd7f275104e23d1eb52b1673164fe475938e7aa42b9be72f589e42e955d9ae9aa9df_1920.jpg"
categories: 
     - JS原理
---
# JS core

## GC机制：Tracing Garbage Collection

从GCRoot维护的一条引用链，会清除引用链无法触达的地方

## JS Context

熟悉js的人应该都知道，jsContext就是js执行的上下文环境，其中比较重要的一点就是全局的WIndow对象

## JSVirtualMachine（JSVM）

JSVM是一个抽象的虚拟机。不同JSVM执行不同的任务，每个JSContext都从属于一个JSVM，每个JSVM都有自己独立的堆空间，GC也只能处理JSVM内部的对象，不同的JSVM之间无法进行传值。JSVM和JS Context之间的关系如下：

![Untitled](https://github.com/SkylarShuang/blog_images/blob/main/Untitled%201.png?raw=true)

## ****JSExport****

实现JSExport协议可以开放OC类和它们的实例方法，类方法，以及属性给JS调用

## JSValue

JSValue实例是一个指向JS值的引用指针。我们可以使用JSValue类，在OC和JS的基础数据类型之间相互转换。同时我们也可以使用这个类，去创建包装了Native自定义类的JS对象，或者是那些由Native方法或者Block提供实现JS方法的JS对象**。**

## 总结

JSCore给iOS App提供了JS可以解释执行的运行环境与资源。对于我们实际开发而言，最主要的就是JSContext和JSValue这两个类。JSContext提供互相调用的接口，JSValue为这个互相调用提供数据类型的桥接转换。让JS可以执行Native方法，并让Native回调JS，反之亦然。

![Untitled](https://github.com/SkylarShuang/blog_images/blob/main/Untitled.png?raw=true)