---
layout:     post
title:      "JavaScript学习一"
date:       2017-04-29
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - JavaScript
---

移动端狗发现大前端才是未来的方向,所以学习一下Js。其实个人很讨厌Js,很久以前也学过发现毫无兴趣,感觉非常没有逻辑(不要打我),但都开始改RN了还是先巩固一下基础吧> <

### 对象

在Js中,对象是属性的集合。

```javascript
//||可以填充默认值
var middle = a["xx"]||"unknown"
```

#### 原型

[这个系列讲的太好了,哭泣](http://www.cnblogs.com/wangfupeng1988/p/3977987.html)

**对象都是通过函数创建的,函数也是对象**

每个对象都有一个隐藏的属性——`“__proto__”`，这个属性引用了创建这个对象的函数的`prototype`。即：`fn.__proto__ === Fn.prototype`

```javascript
function Fn() {
    this.name = '王福朋';
    this.year = 1988;
}
var fn1 = new Fn();
```

>Object和Array都是函数

![](/img/2017-04-29-javascript-one/14934546630375.png)

每个函数都有一个属性叫做prototype

```js
function Fn() { 
    Fn.prototype.name = '王福朋';
    Fn.prototype.getYear = function () {
        return 1988;
    };
    var fn = new Fn();
    console.log(fn.name);
    console.log(fn.getYear());
}
```


![](/img/2017-04-29-javascript-one/14921573274630.png)

* 自定义函数的prototype也是对象,其的__proto__指向的就是Object.prototype。
* Object.prototype确实一个特例——它的__proto__指向的是null
* 函数是被Function创建的
* 函数Foo.__proto__指向Function.prototype，Object.__proto__指向Function.prototype,Function也是一个函数，函数是一种对象，也有__proto__属性。既然是函数，那么它一定是被Function创建。所以——Function是被自身创建的。所以它的__proto__指向了自身的Prototype。


总结一下,函数的__proto__ = Function.prototype

prototype也是对象,其__proto__ = Object.prototype

>Instanceof运算符的第一个变量是一个对象，暂时称为A；第二个变量一般是一个函数，暂时称为B。
Instanceof的判断队则是：沿着A的__proto__这条线来找，同时沿着B的prototype这条线来找，如果两条线能找到同一个引用，即同一个对象，那么就返回true。如果找到终点还未重合，则返回false。

>访问一个对象的属性时，先在基本属性中查找，如果没有，再沿着__proto__这条链向上找，这就是原型链。使用`hasOwnProperty`可以只在自己的基本属性中找


#### 执行上下文

##### 准备工作

* 变量、函数表达式——变量声明，默认赋值为undefined；
* this——赋值；
* 函数声明——赋值；

这三种数据的准备情况我们称之为“执行上下文”或者“执行上下文环境”。

![](/img/2017-04-29-javascript-one/14934560033963.png)


>javascript在执行一个代码段之前，都会进行这些“准备工作”来生成执行上下文。这个“代码段”其实分三种情况——全局代码，函数体，eval代码。

![](/img/2017-04-29-javascript-one/14934680400123.jpg)

##### this

[直接看这个吧](http://www.cnblogs.com/wangfupeng1988/p/3988422.html)

##### 作用域

javascript除了全局作用域之外，只有函数可以创建的作用域。
自由变量: 在A作用域中使用的变量x，却没有在A作用域中声明（即在其他作用域中声明的），对于A作用域来说，x就是一个自由变量。

要到创建这个函数的那个作用域中取值——是“创建”

假设a是自由量

* 现在当前作用域查找a，如果有则获取并结束。如果没有则继续；
* 如果当前作用域是全局作用域，则证明a未定义，结束；否则继续；
* 不是全局作用域，那就是函数作用域）将创建该函数的作用域作为当前作用域；

Js缺少块级作用域,所以应该在顶部声明所有变量。

函数内部声明变量的时候，一定要使用var命令。如果不用的话，你实际上声明了一个全局变量！

![](/img/2017-04-29-javascript-one/14934665457550.jpg)
![](/img/2017-04-29-javascript-one/14934669219870.jpg)


因为没看到第二个scope赋值我差点又要怀疑人生了,其实因为第二个scope的作用域是整个function,但执行到var前还没有被赋值所以是undefined,看过前面推荐的系列就很容易懂了


##### 闭包

函数作为返回值，函数作为参数传递。闭包中,有些情况下，函数调用完成之后，其执行上下文环境不会接着被销毁。

![](/img/2017-04-29-javascript-one/14934627858999.png)


出于种种原因，我们有时候需要得到函数内的局部变量。但是，前面已经说过了，正常情况下，这是办不到的，只有通过变通方法才能实现。

```js
function f1(){
　　　var n=999;
　　　function f2(){
　　　　　alert(n); 
　　　}
　　　return f2;
}
var result=f1();
result(); // 999
```

这里f2就是闭包,闭包就是能够读取其他函数内部变量的函数。
它的最大用处有两个，一个是前面提到的可以读取函数内部的变量，另一个就是让这些变量的值始终保持在内存中。

来一个很恶心的for循环例子:

```js
function box(){
    var arr = [];
    for(var i=0;i<5;i++){
        arr[i] = i;        
    }
    return arr;
}
alert(box())   
```

输出[4，4，4，4，4] QAQ

[知乎这个解释很强](https://www.zhihu.com/question/33468703)

![](/img/2017-04-29-javascript-one/14934645101371.png)

改造后:

```js
for(var i=0,arr=[];i<=3;i++) {
		arr.push(
			(function(i){
				return function(){
					alert(i);
				}
			})(i)
		);
	}
```

![](/img/2017-04-29-javascript-one/14934646959028.png)


![](/img/2017-04-29-javascript-one/14934651332269.jpg)

##### Array & Object


Array和Object都是函数。

##### 私有变量


任何在函数中定义的变量，都可以认为是私有变量，因为不能在函数的外部访问这些变量。私有变量包括函数的参数、局部变量和函数内定义的其他函数。


```js
function car(){ 
      var wheel = 3;//私有变量 
      this.wheel = 4;//公有变量 
 } 
var car1 = new car(); 
alert(car1.wheel);结果：4
```

访问私有变量用闭包吧:

```js
function car(){ 
    var wheel = 3;//私有变量 
    this.wheel = 4;//公有变量 
    this.getPrivateVal = function(){ 
        return wheel; 
    } 
} 
var car1 = new car(); 
alert(car1.getPrivateVal());结果：3 
```





