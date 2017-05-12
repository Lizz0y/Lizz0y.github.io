---
layout:     post
title:      "Build Your Own Angularjs-预备工作"
date:       2017-05-12
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - 阅读笔记
    - 翻译
    - Build Your Own Angularjs
---

开个`Build Your Own Angularjs`的翻译系列吧。个人目前熟练移动端，前端是刚入门的状态，所以有什么问题请大家多多指出。学习前端到了有点迷茫的阶段，之前android基本把一些库的源码都能很流畅的撸一遍，但直接撸前端库发现难度有点大摔，所以借着还有两个月毕业的时间，强迫自己把这本书翻译一遍，算是给自己的一个毕业礼物吧。所有代码见![github](https://github.com/Lizz0y/myAngularjs).

从今天开始，我们要完成一个强大的`JavaScript`框架。我们需要建立一个工程，并配上自动化测试机制。幸运的是，现在已经有很多现成的工具供我们使用。在这节，我们使用`NPM`和`Grunt`建立一个`JavaScript`库，并使用`JSHint`进行静态分析，`Jasmine`进行单元测试。

首先介绍一下相关工具：

* `Node.js`: 流行的服务端JS平台。`Nodejs`使用`Npm`作为模块管理器，现在请先搭建好搭建`Node`和`Npm`环境，具体过程请自行完成。
* Grunt:我们将使用它来作为JS的构建工具。现在请使用`Npm`安装`grunt-cli`包。

安装`Grunt`时，需要同时安装`grunt`与`grunt-cli`，并使用`npm -g`进行全局安装。

到这一步时，你需要安装好`node`,`npm`,`grunt`与`grunt-cli`

### 创建工程目录

现在让我们为我们的库创建基本的目录结构吧！我们需要以下几个目录，包括`src`源目录和用来进行单元测试的`test`目录。

![](/img/2017-05-12-myAngular-preface/14945186718073.jpg)

### 为Npm创建package.json

为了使用`Npm`,我们需要使用`package.json`文件。该文件让`Npm`了解我们工程的一些基本情况，主要提供所依赖的包信息。

![](/img/2017-05-12-myAngular-preface/14945193612186.jpg)


### 创建grunt的构建文件。
这个文件只是一段JS文件，包括要运行的任务和配置。可以用来测试和打包项目。你可以把他想象成`C`中的`Makefile`,或者`android`中的`build.gradle/ant.xml`


![](/img/2017-05-12-myAngular-preface/14945193943496.jpg)
一切在该文件中都被包装成函数，并被模块导出，是不是和`Node.js`的模块机制很像。`grunt.initConfig`将把我们的项目配置传递给`Grunt`

### Hello World!

我们先来一段`Hello World`开心一下吧~ 创建一个`hello.js`文件

>src/hel lo.js

```js
function sayHello() {    return "Hello, world!";
}
```

### 使用`JSHint`来静态分析

`JSHint`是一个阅读你的`JS`代码并给出存在的语法或结构问题。和`android`中的`linter`很像，都是代码分析工具。

使用Npm安装吧 `npm install grunt-contrib-jshint --save-dev`

当你在工程目录下安装时，你的工程中会出现`node_modules`,

![](/img/2017-05-12-myAngular-preface/14945195987333.jpg)

新安装的模块也会出现在其中,哦你会发现你的`package.json`也发生了一些变化

![](/img/2017-05-12-myAngular-preface/14945196429889.jpg)

多了`devDependencies`,因为你加了`--save-dev`，所以这些变化会自动写到`package.json`,不然就要你手动加啦。

现在，我们把`gruntfile.js`改动一下:

![](/img/2017-05-12-myAngular-preface/14945196904936.jpg)
`all`代表只检查`src`下所有`js`文件，并且我们会引用两个不在我们自己代码中定义的变量。即`_ from Lo-Dash` 和`$ from jQuery`, 我们启用了`browser`和`devel JSHint`环境，这使得当我们在引用来自浏览器的`Global`变量例如`setTimeout`和`console`不会报错

`grunt.loadNpmTasks`将`jshint`的任务传递到我们自己的构建任务中。


我们使用`grunt jshint`来进行代码检查

![](/img/2017-05-12-myAngular-preface/14945608192848.jpg)
可以看到`lint free`!


### 使用Jasmine,Sinon，Testem进行单元测试

单元测试在平时开发过程中非常常见，我们需要有一个好的测试框架。我们使用`Jasmine`，因为他有一个好又简单的`API`并且可以和`Grunt`良好交互。

我们使用`Testem`跑测试用例，它提供了一个很好的`UI`界面让我们看测试结果,使用`Sinon.JS`作为测试辅助库，当我们要用到`http`时`sinon`变得非常有用。

我们首先安装`grunt`与`Testem`进行交互整合的库。他会帮我们同时引入`Jasmine`框架的依赖

`npm install grunt-contrib-testem —save-dev`

同时安装`Sinon`,这两步都在当前目录下进行 `npm install sinon —save-dev`

然后在`Gruntfile.js`中配置`Testem`,

![](/img/2017-05-12-myAngular-preface/14945609013120.jpg)

同时在最后一行增加
`grunt.loadNpmTasks(‘grunt-contrib-testem’)`

在`initConfig`中我们定义了`testem`任务，并把它叫做`unit`。它会为在`src`目录下的所有js文件运行测试用例,测试文件在`test`目录下,运行时,`Testem`会一直观察这些文件是否变化了,如果变化了就自动重新运行测试用例。


使用`launch_in_dev`,我们告诉`testem`我们希望建立一个`PhantomJs`的浏览器来运行测试用例。`PhantomJS`是一个基于`webkit`的`javascript API`。它让我们运行测试用例时不需要管理外部浏览器,就是说就算你没安装浏览器它也可以模拟一个浏览器来给你运行测试用例。现在全局安装它吧

`npm install -g phantoms`

我们使用`before_task`钩子让`JSHint`先进行检查

在测试代码中我们会引用在`Jasmine`中定义的全局变量,所以把他们加入到`JSHint`的`global`对象中
![](/img/2017-05-12-myAngular-preface/14945609278998.jpg)
我们新建`hello_spec.js`文件作为测试用例,新建在`test`目录下:

![](/img/2017-05-12-myAngular-preface/14945609336952.jpg)

如果你用过`ruby`的`rSpec`,那这段代码应该很熟悉,`describe`块代表一系列测试用例,每个测试用例都有一个名字和函数

`Jasmine`的idea来自`Behavior-Driven Development`,行为驱动开发。`describe`和`it`描述了我们代码的行为,这些测试用例代表我们的代码应该具有的输出。这也是为什么使用`_spec.js`作为后缀。

我们来运行吧,`grunt testsem:run:unit`

![](/img/2017-05-12-myAngular-preface/14945609390005.jpg)

在浏览器中打开它说的`url`可以测试其他浏览器，这里可以看到测试通过，

![](/img/2017-05-12-myAngular-preface/14945609451436.jpg)
这一步其实花了很多功夫才成功,首先Phantomjs装不成功,一直报错,后来github的issue上有人说卸了去他的官网直接装二进制文件,果然成功了。

在`Web Storm`的`terminal`里出不来。。。不懂为什么


尝试编辑`src/hello.js`或是`test/hello_spec.js`,如果`termina`l一直运行着的话是会自动刷新重新测试的。当你学习这本书的时候，我们希望`TestemUI`永远运行着，这样就能一直进行test。


### Include Lo-Dash和jQuery

`Angular`本身不需要`require`任何第三方库,除了`jQuery`,然而我们有必要在我们自己的`Angular`中引入一些库,让`Angular`更酷一些?

以下两个低级别的操作可以委派给第三方库

* 数组和对象操作,比如`Equal`和`Clone` ,委托给`Lo-Dash` 
* `dom`操作和处理,委托给`jQuery`

首先安装:

`npm install lodash —save`
`npm install jquery —save`

与之前安装的区别在于`--save`,意味着这些包在运行时被依赖，不是开发期依赖

我们现在来尝试一些新鲜的东西：

这个函数把`to`当做参数，使用`Lo-Dash`模板构造了一个结果字符串，我们同时修改测试用例:

```js
function sayHello(to) {    return _.template("Hello, <%= name %>!")({name: to}); 
}
```

测试用例:

```js
describe("Hello", function() {    it("says hello to receiver", function() {         expect(sayHello( 'Jane' )).toBe("Hello, Jane!");    }); 
});
```

如果我们现在尝试运行测试用例，是不会成功的，因为`Testem`找不到`Lo-Dash`,因为我们把`Lo-Dash`安装在`node_packages`中，没有把加载到我们的测试用例中，所以`_`变量是找不到的，我们可以修复他 通过改变`Gruntfile.js`,把`Lo-Dash`和`jQuery`一起包含进来:

![](/img/2017-05-12-myAngular-preface/14946029801466.jpg)

ok,我这么做了,测试失败，说我`redefination`....,重复定义了,还是在github的issue上找到解决方案,天下码农一家亲，

```js
        testem: {
            unit: {
                options: {
                    framework:  'jasmine2' ,
                    launch_in_dev: [ 'PhantomJS' ],
                    before_tests:  'grunt jshint ',
                    serve_files: [
                        'node_modules/lodash/lodash.min.js',
                        'node_modules/jquery/dist/jquery.min.js',
                        'node_modules/sinon/pkg/sinon.js',
                        'src/**/*.js',
                        'test/**/*.js'

                    ],
                    watch_files: [
                        'src/**/*.js' ,
                        'test/**/*.js'
                    ] }
} }
```

测试成功！

![](/img/2017-05-12-myAngular-preface/14945609583639.jpg)



### 定义默认的Grunt Task

在`Gruntfile.js`最后加一句:

`grunt.registerTask( 'default' , [ 'testem:run:unit' ]);` 
在命令行直接`grunt`即可运行测试用例。






