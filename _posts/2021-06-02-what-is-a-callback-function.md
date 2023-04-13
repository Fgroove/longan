---
title: What is a callback function?
author: Mista
date: 2021-06-02 00:32:00 +0800
categories: [Question, others]
tags: [callback]
---

什么是回调函数呢？

## what is a callback function?

回调函数，听名字很让人不解。回调函数是一个函数：

* 可以被另外的函数访问，以及
* 在第一个函数执行完成后被调用，

可以把回调函数理解为在caller（回调函数被当作参数传入的）的结尾被调用的函数。也可以叫做`call after`函数。

这个结构对异步操作非常有用，可以在前一个事件完成时触发一个活动。

Pseudocode:

```java
// A function which accepts another function as an argument
// (and will automatically invoke that function when it completes - note that there is no explicit call to callbackFunction)
funct printANumber(int number, funct callbackFunction) {
    printout("The number you provided is: " + number);
}

// a function which we will use in a driver function as a callback function
funct printFinishMessage() {
    printout("I have finished printing numbers.");
}

// Driver method
funct event() {
   printANumber(6, printFinishMessage);
}
```

如果调用`event()`，

```
The number you provided is: 6
I have finished printing numbers.
```

注意这里的输出顺序，回调函数`printFinishMessage`在`printANumber`执行完成之后被调用。

回调函数和指针语言一起使用，它只是用来描述作为参数提供给另一个函数的方法，在父方法被调用时（例如按钮点击，计时器等）且父方法完成后，调用回调函数。



To be continued ...

> In computer programming, a callback is a reference to executable code, or a piece of executable code, that is passed as an argument to other code. This allows a lower-level software layer to call a subroutine (or function) defined in a higher-level layer.
>
> ---[Wikipedia](https://en.wikipedia.org/wiki/Callback_(computer_science))

## Reference

[What is a callback function?](https://stackoverflow.com/questions/824234/what-is-a-callback-function)

