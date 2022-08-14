---
title: "Javascript异步编程"
date: 2022-01-26T21:58:57+08:00
draft: false

summary: 初学Javascript，总觉得异步这一块设计的很复杂，简单梳理一下思路
description: 简单梳理一下Javascript异步编程相关知识

tags: ["Javascript", "async"]
categories: ["front-end"]

cover: 
    image: 'img/cover.jpg'
---

第一次接触Javascript这样基于事件模型的单线程语言，简单梳理一下异步编程这块的内容。

## 回调函数

回调函数（Callback）的概念其实很简单，在很多语言里都很常见：

```Javascript
function add(x, y)
{
    return x + y
}

function doSomething(x, y, func)
{
    console.log( func(x, y) )
}

doSomething(1, 2, add)
// 3
```

这里`add()`就是传给`doSomething()`的一个回调函数。说白了，“回调”就是把一个函数作为参数传递给另一个函数。

在C等更接近底层的语言中，可以使用函数指针来达到“传递代码块”的目的；而在Python或Javascript中，直接使用函数名就可以直接实现这一过程。

这里很显然，在调用`doSomething()`之后立即执行了传入的函数`func`，所以这就是一个最简单的同步回调（synchronous callback），同步回调还有一个名字叫“阻塞回调（blocking callback）”，因为如果回调函数过于复杂，它会直接阻塞CPU直到执行完成。

有同步回调，自然有异步回调，异步回调（asynchronous callback）不会立即执行传入的代码，因此不会立即阻塞CPU的运行：

```Javascript
function add(x, y)
{
    return x + y
}

function doSomething(x, y, func)
{
    setTimeout( () => {
        console.log( func(x, y) )
    }, 1000)
}

doSomething(1, 2, add)
console.log("***")

// 输出如下:
// ***
// 随后等待约 1000ms, 输出:
// 3
```

所以异步回调还有一个更形象的名字——延迟回调（deferred callback）

## 异步回调函数

> 异步是异步，回调是回调，异步回调是基于回调实现的一种异步编程手段。
>
> — 周树人

在调用一个普通的函数时，我们可以直接使用函数的返回值来进行后续操作：

```Javascript
let result = Math.pow(4, 0.5)
console.log(result)
```

但是，如果遇到一个耗时较长的操作（读写大文件、网络请求等），这种方式将长时间阻塞程序，无疑是对CPU资源的一种浪费：

```Javascript
// ......

let photo = downloadFile("www.hello.com/test.png")         // 将在这里被阻塞
photo = resize(photo)
show(photo)

// ......
```

其它编程语言（C++, Java, Python...）大多可以通过多线程/多进程来解决这一问题，但由于Javascript不支持多线程/进程操作，因此只能通过异步的方式来间接的达到并发的目的。

在Javascript中如何实现异步编程呢？一种最常用的解决方法是使用异步回调函数：

```Javascript
// ......

function handlePhoto(photo) {
    photo = resize(photo)
    show(photo)
}

downloadFile("www.hello.com/test.png", (photo) => handlePhoto(photo))

// ......
```

在使用常规函数的过程中，是将变量（如`photo`）传入到这些函数中（如`resize()`），再由函数内部的代码对输入进行必要的处理；

而使用异步回调函数的思路与使用普通函数的思路并不完全相同，使用回调函数时是将一段代码（如`handlePhoto()`）作为参数来传入到即将产生变量（如`photo`）的函数内（如`downloadFile()`）。这样，在执行到`downloadFile()`语句时，浏览器会一边下载图片，一边继续执行`downloadFile()`后面的代码；待图片下载完成后，再回过头来调用传入的回调函数`handlePhoto()`（为什么会按这样的异步顺序执行，和`downloadFile()`这类函数的实现有关，这里暂不讨论）。

这就产生了一个新的问题，继续上面的例子，如果想要在回调函数外对`photo`进行其它操作，应该怎样做呢？答案是不可行的，因为**根本无法获取回调函数的返回值**，这是由回调函数本身的特性决定的，**与异步无关**。

```Javascript
// ......

function handlePhoto(photo) {
    photo = resize(photo)
    show(photo)

    return photo                  // 没有任何意义, 无法在函数外拿到返回值
}

downloadFile("www.hello.com/test.png", (photo) => handlePhoto(photo))
save(photo)

// ......
```

换句话说，唯一的解决方法，就是把后续代码中所有涉及变量`photo`的操作全部移动到`handlePhoto()`内。

另外，一个合格的程序，还应当具有异常处理等功能。很显然，这样最后会得到一个非常臃肿的`handlePhoto()`函数：

```Javascript
// ......

function handlePhoto(photo) {
    photo = resize(photo)
    show(photo)
    save(photo)

    // 其它各种操作
    // ......
}

downloadFile("www.hello.com/test.png", (photo) => handlePhoto(photo))

// ......
```

随着程序越来越复杂，还有可能陷入到深层次嵌套的回调地狱（callback hell）中。按照惯例，应当避免两层以上的函数嵌套。

## Promise

为了解决前面提到的回调地狱等问题，ES6中引入了`Promise`类型，利用`Promise`语法，可以轻松的将一个深度嵌套的回调函数改写成一种顺序的、更加易于理解的形式（即没有复杂的嵌套）：

```Javascript
let p = new Promise((resolve, reject) => {            // new 一个 Promise 类, 并向 Promise 类的构造器传递一个函数
    downloadFile("www.hello.com/test.png", (photo) => {
        if(photo) {                                   // 处理下载失败的情况
            reject('Download Failed')
        }
        else {
            resolve(photo)                            // 处理下载成功的情况
        }
    })
})

p.catch(errorMessage => {                             // 如果下载失败, 执行该语句
    console.log(errorMessage)
})

p.then(photo => {                                     // 如果下载成功, 继续执行该语句
    photo = resize(photo)
    return photo
}).then(photo => {                                    // 如果上一部分执行成功, 继续执行该语句
    show(photo)
    save(photo)
    return photo
})

p.then(phoyo => {                                     // 如果上一部分执行成功, 继续执行该语句
    // 其它各种操作
    // ......
})
```

`Promise`的用法也很简单，首先要使用new运算符实例化一个`Promise`对象，`Promise`类的构造器可以被用来包装任意一个返回值不是`Promise`类型的函数，如上面的第一行代码：

```Javascript
let p = new Promise((resolve, reject) => {            // new 一个 Promise 对象, 并向 Promise 类的构造器传递了一个函数
    downloadFile("www.hello.com/test.png", (photo) => {
        if(photo) {                                   // 处理下载失败的情况
            reject('Download Failed')
        }
        else {
            resolve(photo)                            // 处理下载成功的情况
        }
    })
})
```

上面向`Promise`构造器传递了一个具有两个参数的匿名函数，实际上也就等价于下面这段代码：

```Javascript
let p = new Promise(getPhoto)

function getPhoto(resolve, reject) {
    downloadFile("www.hello.com/test.png", (photo) => {
        if(photo) {
            reject('Download Failed')
        }
        else {
            resolve(photo)
        }
    }) 
}
```

简单的说，我们向`Promise`构造器传入了一个回调函数，该回调函数将被传入两个参数`resolve`和`reject`，下面来解释一下这两个参数的含义：

我们在做任何一件事情的过程中，都一定存在且只存在三种情况：

* 这件事情尚未完成（还没开始 or 正在进行）
* 已经执行完成
* 执行失败

> 吃饭了吗？
>
> 还没吃/正在吃、吃完了、吃了，但吃到了奇怪的东西。

前面我们已经把要做的事情（下载文件）包装到一个`Promise`对象里，那现在这个事情（`Promise`对象）执行到哪一步了呢？在`Promise`中，我们使用：

* pending（待定）
* resolve（解决，或fulfilled）
* reject（拒绝）

来描述这三种状态，任意`Promise`对象的状态都一定是这三种状态之一。

这样，当使用`new`运算符创建一个新的`Promise`对象时，该对象的状态就是待定的`pending`状态；而如果执行了`resolve()`函数，`Promise`对象将转变为`fulfilled`状态，表明程序执行成功；同样的，如果调用`reject()`，则将转变为`reject`状态，表明程序执行失败。

那么改变`Promise`的状态又有什么效果呢？在前面的程序中，我们向`.then()`和`.catch()`方法各自传入了一个回调函数。如果`Promise`的状态变为`fulfilled`，就会执行传给`.then()`的函数；如果`Promise`的状态变为`reject`，就会执行传给`.catch()`的函数，这样就优雅的实现了异步函数的顺序执行，从而避免了回调函数的深层次嵌套。

接下来注意到，前面的程序在调用`resolve()`和`reject()`时，还向它们传递了参数：

```Javascript
// ......
        if(photo) {
            reject('Download Failed')
        }
        else {
            resolve(photo)
        }
// ......
```

其实作用也很明显，传给`reject()`和`resolve()`的参数将作为回调函数的参数再次传递给紧跟其后的`.catch()`或`.then()`方法，所以才会有：

```Javascript
p.catch(errorMessage => {         // 如果下载失败, 执行该语句
    console.log(errorMessage)
})

p.then(photo => {                 // 如果下载成功, 继续执行该语句
    photo = resize(photo)
    return photo                  // 将 photo 传递给下一个 .then() 方法
})
```

另外，`.then()`和`.catch()`方法还有一个重要的性质，即它们都会返回一个`Promise`对象，这样就可以随意的实现`Promise`的链式调用：

```Javascript
p.then(photo => {            // 如果下载成功, 继续执行该语句
    photo = resize(photo)
    return photo             // 这里的 return 只是将 photo 作为回调函数的参数 (划重点) 
                             // 传递给下一个 .then() 方法, 而 .then() 返回的仍然是 Promise 对象, 不是 photo
}).then(photo => {           // 如果上一部分执行成功, 继续执行该语句
    show(photo)
    save(photo)
    return photo
})
```

在使用`Promise`编写程序时，`.then()`和`.catch()`方法的调用顺序其实也有讲究，但这里就不对该问题做进一步讨论了。

最后还有一个问题，为什么`Promise`还允许用户传入一个用于异常处理的回调函数`reject()`，而不是让用户直接通过`try...catch()`来捕获异常？这个其实是因为在Javascript中，`try...catch()`语句块无法捕获异步回调中的异常：

```Javascript
try {
    setTimeout(function() {
        throw new Error('You can not catch this error.')
    }, 0)
} catch (e) {
    console.error(e)
}
```

在`setTimeout()`内抛出的异常是无法被`try...catch()`直接捕获的。

## Async/Await

Promise虽然已经解决了回调地狱的问题，但仍不够优雅————大量的`.then()`和`.catch()`语句块依然看着很傻。

于是，Javascript的设计者又设计出新的语法糖————`async`和`await`关键字。

待更。
