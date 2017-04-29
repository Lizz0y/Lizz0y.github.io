---
layout:     post
title:      "Gradle实战与Nuwa顺带分析"
date:       2017-04-28
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Gradle
    - HotFix
---






[很棒的总结](http://jiajixin.cn/2015/08/07/gradle-android/)

[很好的语法总结](http://blog.csdn.net/innost/article/details/48228651)



### Gradle基本

![](/img/2017-04-29-gradleAndNuwa/14932951429578.jpg)


我们知道,一个Task的生命周期分三部分:初始化,配置,执行。

* 初始化阶段，会去读取根工程中setting.gradle中的include信息，决定有哪几个工程加入构建，创建project实例，比如下面有三个工程：`include ':app', ':lib1', ':lib2'`

* 配置阶段，会去执行所有工程的build.gradle脚本，配置project对象，一个对象由多个任务组成，此阶段也会去创建、配置task及相关信息。

* 运行阶段，根据gradle命令传递过来的task名称，执行相关依赖任务

![](/img/2017-04-29-gradleAndNuwa/14933572585310.jpg)


注意两种task的区别,第一种写法代表是配置,就算执行另一个task,Hello world也还是会打印出来。第二种写法代表是真正的执行体。

#### 设置task的类型和依赖

```java
task copyFile(type: Copy) {
   from 'xml'
   into 'destination'
}
task taskA(dependsOn: taskB) {
   //do something
}
```

#### 设置Property


![](/img/2017-04-29-gradleAndNuwa/14932981971835.jpg)


### 一些重要语法

```java
//无类型的函数定义，必须使用def关键字  
def  nonReturnTypeFunc(){  
    last_line   //最后一行代码的执行结果就是本函数的返回值  
}  
//如果指定了函数返回类型，则可不必加def关键字来定义函数  
String getString(){  
   return"I am a string"  
}  
```

#### closure

```java
def aClosure = {//闭包是一段代码，所以需要用花括号括起来..  
    Stringparam1, int param2 ->  //这个箭头很关键。箭头前面是参数定义，箭头后面是代码  
    println"this is code" //这是代码，最后一句是返回值，  
   //也可以使用return，和Groovy中普通函数一样  
}  
aClosure.call("this is string",100)  或者  
aClosure("this is string", 100)  
```

如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫it，和this的作用类似。it代表闭包的参数。

```java

public static <T> List<T>each(List<T> self, Closure closure)  
/**
上面这个函数表示针对List的每一个元素都会调用closure
做一些处理。这里的closure，就有点回调函数的感觉。但是，在使用这个each函数的时候，我们传递一个怎样的Closure进去呢？比如：  
def iamList = [1,2,3,4,5]  //定义一个List  
iamList.each{ //调用它的each，这段代码的格式看不懂了吧？each是个函数，圆括号去哪了？  
      println it  
}**/
```
Groovy中，当函数的最后一个参数是闭包的话，可以省略圆括号

  
### 自定义Gradle插件

[自定义Gradle插件](http://blog.csdn.net/liuhongwei123888/article/details/50541759)


![](/img/2017-04-29-gradleAndNuwa/14933703969513.jpg)



自定义插件的文件结构如上图。

#### 插件主要代码

```java
package com.micky.gradle;  
import org.gradle.api.*;  
  
class MyCustomPlugin implements Plugin<Project> {  
    void apply(Project project) {  
        project.task('myTask') << {  
            println "Hi this is micky's plugin"  
        }  
    }  
}  
```

在`META-INF`的`plugins`下创建`property`文件,名字需要跟plugin的ID匹配,比如这里应该是`包名+ mycustom + properties`,这个是以后我们在`apply plugin` 使用的名字

内容为: `implementation-class=包名+MyCustomPlugin`

在根目录下建立`setting.gradle`目录,设置插件名字:
`rootProject.name='gradle-micky'`

#### Maven发布

```java
apply plugin: 'groovy'  
apply plugin: 'maven'  
  
dependencies {  
    compile gradleApi()  
    compile localGroovy()  
}  
  
repositories {  
    mavenCentral()  
}  
  
group='com.micky'  
version='1.0.0'  
  
uploadArchives {  
    repositories {  
        mavenDeployer {  
            repository(url: uri('../repo'))  
        }  
    }  
}  
```

or

```java
apply plugin: 'maven'
uploadArchives {
    repositories {
        mavenDeployer {
            pom.groupId = GROUP_ID
            pom.artifactId = ARTIFACT_ID
            pom.version = VERSION
            repository(url: RELEASE_REPOSITORY_URL) {
                authentication(userName: USERNAME, password: PASSWORD)
            }
        }
    }
}
```


#### 执行

`gradle uploadArchives`就把插件jar文件存储在本地maven库里，本地maven库即我们在脚本里创建的"../repo"目录

#### 应用

```java
buildscript {  
    repositories {  
        maven {  
            url uri('../repo')  
        }  
    }  
  
    dependencies {  
        classpath group: 'com.micky',  name: 'gradle-micky',  version: '1.0.0'  
    }  
}  
  
apply plugin: 'com.micky.mycustom'  
```

可以看到url就是本地repo,设置依赖,还要设置apply plugin,代表引入这个插件


#### 加入任务

```java
afterEvaluate {
    android.applicationVariants.each { variant ->
        def dx = tasks.findByName("dex${variant.name.capitalize()}")
        def hello = "hello${variant.name.capitalize()}"
        task(hello) << {
			println "hello"
        }
        tasks.findByName(hello).dependsOn dx.taskDependencies.getDependencies(dx)
        dx.dependsOn tasks.findByName(hello)
    }
}
```

我们知道,variant = productFlavors+ buildTypes 所以每个variant的每个dex是不一样的
### 自定义Gradle Task

```java
class MyCustomTask extends DefaultTask {  
    @TaskAction  
    void output() {  
        println "Hello this is my custom task output"  
    }  
}  
class MyCustomPlugin implements Plugin<Project> {  
    void apply(Project project) {  
        project.task('customTask', type: MyCustomTask)  
    }  
} 
```

#### Extension

```java
class MyCustomPluginExtension {  
    def message = "From MyCustomPluginExtention"  
    def sender = "MyCustomPluin"  
} 
class MyCustomPlugin implements Plugin<Project> {  
    void apply(Project project) {  
        project.extensions.create('myArgs', MyCustomPluginExtension)  
        project.task('customTask', type: MyCustomTask)  
    }  
} 
void output() {  
        println "Sender is ${project.myArgs.sender},\nmessage: ${project.myArgs.message}"  
} 
```

在`build.gradle`内也可以重新定义message,sender。所以叫扩展,就是用户可以自由定义:

```java
myArgs {  
    sender='Micky Liu'  
    message='Gradle is so simple.'  
    nestArgs {  
        receiver='David Chen'  
        email='David@126.com'  
    }  
  
} 
```
#### 多渠道打包

##### flavor

flavor + buildType :

buildType更多是内部的不同,例如debug or release
flavor更多是外部的不同,例如多个渠道

Gradle会为不同flavor关联对应的sourceSet，默认位置为src/<flavorName>目录

```java
android {
    ....

    productFlavors {
        flavor1 {
            minSdkVersion 14
        }
    }
}

android {
    defaultConfig {
        buildConfigField "boolean", "AUTO_UPDATES", "true"
    }

    productFlavors {
        wandoujia {
            buildConfigField "boolean", "AUTO_UPDATES", "false"
        }        
    }

}
```
>Gradle会在generateSources阶段为flavor生成一个BuildConfig.java文件。BuildConfig类默认提供了一些常量字段，比如应用的版本名（VERSION_NAME），应用的包名（PACKAGE_NAME）等。更强大的是，开发者还可以添加自定义的一些字段。
 
 ![](/img/2017-04-29-gradleAndNuwa/14933646908691.jpg)

 
 Gradle在构建应用时，会优先使用flavor所属dataSet中的同名资源。所以可以进行资源覆盖,例如换包名等。
 
##### 修改包名

[有借鉴意义的文章](http://hugozhu.myalert.info/2014/08/03/50-use-gradle-to-customize-apk-build.html) 
[很多总结性知识](http://toastdroid.com/2014/03/28/customizing-your-build-with-gradle/)

让我想起了我悲伤的换包,我要跟导师控诉

* 修改debug版的包名
* 修正资源文件里的包名
* 定制APK的应用名称
* 修改Cp的authority

```java
android.applicationVariants.all { variant ->
    def buildType = variant.buildType
    def encoding = java.nio.charset.Charset.defaultCharset().toString()
    if (buildType.applicationIdSuffix) {
        def defaultPackageId = variant.packageName.replaceAll(buildType.applicationIdSuffix,'')
        variant.mergeResources.doLast {
            def dir = file("${buildDir}/intermediates/res/${variant.dirName}/layout")
            dir.listFiles().each { f->
                String content = f.getText(encoding)
                content = content.replaceAll("res/"+defaultPackageId, "res/"+variant.packageName)
                f.write(content, encoding)
            }
        }
    }
}
```


#### 杂

```java
inputs.dir sources
outputs.file destination
```

指定输入和输出时,可以获得增量编译的优势。


![](/img/2017-04-29-gradleAndNuwa/14933009927487.jpg)

![](/img/2017-04-29-gradleAndNuwa/14933577361720.jpg)


```java
 tasks.whenTaskAdded { theTask ->       if (theTask.name.equals("packageRelease")) {           theTask.dependsOn "getReleasePassword"       } }
``` 

`whenTaskAdded`后面的闭包会在gradle配置阶段完成。
`afterEvaluate`是在配置阶段要结束，项目评估完会走到这一步。

### 遗留问题

据说facebook的Buck构建速度很快,到时候研究一下

### Nuwa源码分析
[很好的实践文章](http://www.cnblogs.com/xgjblog/p/5483220.html)
热补丁,具体原理基于ClassLoader,烂大街了没啥好说的。操作也是:打完patch.jar直接推入sdcard即可。

所以今天来分析一下源码。

Nuwa分成两个部分,`sdk`和`gradle plugin`部分。前者实现`dex`合并过程,后者实现字节码注入过程。所以unbelievable,后者才是nuwa真正的精华部分。

#### sdk

注入代码不再赘述,看一下build.gradle结构:

```
nuwa
----build.gradle ①
sample
----build.gradle ②
build.gradle ③
```

最外层build.gradle③


![](/img/2017-04-29-gradleAndNuwa/14934312843869.jpg)

看到直接引入`maven`,`nuwa`的依赖。注意这里是gradle插件依赖,和后面的`compile project(:nuwa)引入的是sdk依赖不同`

再看①:

```
apply plugin: 'com.android.library'
apply plugin: 'com.github.dcendents.android-maven'
apply plugin: 'com.jfrog.bintray'
```

apply依赖的插件,然后把它作`library`上传到maven仓库中

最后看②:

```
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile project(":nuwa")
    compile 'com.android.support:appcompat-v7:22.2.1'
    compile 'com.android.support:design:22.2.1'
}

apply plugin: "cn.jiajixin.nuwa"

```

看到这里`compile project(:nuwa)`,说明是依赖于工程中另一个项目`nuwa`,`apply plugin: "cn.jiajixin.nuwa"`把gradle插件进行应用。

这里我一直有个疑问,什么时候既要compile&apply,什么时候只需要compile。我个人认为是如果库作为插件那么是前者,如果是作为Library那么是后者。知道的同学请告知一下,感激不尽。

#### gradle plugin


![](/img/2017-04-29-gradleAndNuwa/14934319361168.jpg)
![](/img/2017-04-29-gradleAndNuwa/14934319878836.jpg)


##### Extension

```java
class NuwaExtension {
    HashSet<String> includePackage = []
    HashSet<String> excludeClass = []
    boolean debugOn = true

    NuwaExtension(Project project) {
    }
}
```

声明了Extension,用户可自由配置。

##### Plugin

```groovy
class NuwaPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        project.extensions.create("nuwa", NuwaExtension, project)
```

首先创建`extension`

```groovy
project.afterEvaluate {
            def extension = project.extensions.findByName("nuwa") as NuwaExtension
            includePackage = extension.includePackage
            excludeClass = extension.excludeClass
            debugOn = extension.debugOn

```

在配置完后,把一些用户配置写入变量。

```groovy
project.android.applicationVariants.each { variant ->
    if (!variant.name.contains(DEBUG) || (variant.name.contains(DEBUG) && debugOn)) {
        Map hashMap
        File nuwaDir
        File patchDir
        def preDexTask = project.tasks.findByName("preDex${variant.name.capitalize()}")
        def dexTask = project.tasks.findByName("dex${variant.name.capitalize()}")
        def proguardTask = project.tasks.findByName("proguard${variant.name.capitalize()}")
        def processManifestTask = project.tasks.findByName("process${variant.name.capitalize()}Manifest")
        def manifestFile = processManifestTask.outputs.files.files[0]
        /**
        * 这个属性是从控制台输入的,代表之前release版本生成的混淆文件和hash文件目录,这两个文件发版时需要保持
         * ./gradlew clean nuwaQihooDebugPatch -P NuwaDir=/Users/jason/Documents/nuwa
 */
        def oldNuwaDir = NuwaFileUtils.getFileFromProperty(project, NUWA_DIR)
        if (oldNuwaDir) {
        //如果mapping文件用户自己定义了,那就应用吧
                def mappingFile = NuwaFileUtils.getVariantFile(oldNuwaDir, variant, MAPPING_TXT)
                NuwaAndroidUtils.applymapping(proguardTask, mappingFile)
        }
        if (oldNuwaDir) {
        //把hashFile load到内存
            def hashFile = NuwaFileUtils.getVariantFile(oldNuwaDir, variant, HASH_TXT)
            hashMap = NuwaMapUtils.parseMap(hashFile)
        }
```

遍历variants,找到各种Task。然后创建closure:

```grrovy
def dirName = variant.dirName
nuwaDir = new File("${project.buildDir}/outputs/nuwa")
def outputDir = new File("${nuwaDir}/${dirName}")
def hashFile = new File(outputDir, "hash.txt")
Closure nuwaPrepareClosure = {
    //已经定义了application类,则加入excludeClass列表,不执行字节码修改
    def applicationName = NuwaAndroidUtils.getApplication(manifestFile)
    if (applicationName != null) {
        excludeClass.add(applicationName)
    }
    outputDir.mkdirs()
    if (!hashFile.exists()) {
       hashFile.createNewFile()
    }
    //创建patch文件夹,到时候hash有变动的就放到那里
    if (oldNuwaDir) {
        patchDir = new File("${nuwaDir}/${dirName}/patch")
        patchDir.mkdirs()
        patchList.add(patchDir)
    }
}
def nuwaPatch = "nuwa${variant.name.capitalize()}Patch"
project.task(nuwaPatch) << {
//执行patch的dex操作
    if (patchDir) {
        NuwaAndroidUtils.dex(project, patchDir)
    }
}
def nuwaPatchTask = project.tasks[nuwaPatch]
Closure copyMappingClosure = {
    if (proguardTask) {
        def mapFile = new File("${project.buildDir}/outputs/mapping/${variant.dirName}/mapping.txt")
        def newMapFile = new File("${nuwaDir}/${variant.dirName}/mapping.txt");
        // 将构建产生的mapping文件拷贝至目标nuwa目录
        FileUtils.copyFile(mapFile, newMapFile)
    }
}
```


```groovy
//创建一个task名为nuwaJarBeforeDex
def nuwaJarBeforeDex = "nuwaJarBeforeDex${variant.name.capitalize()}"
project.task(nuwaJarBeforeDex) << {
//获得所有输入文件
    Set<File> inputFiles = dexTask.inputs.files.files
    inputFiles.each { inputFile ->
        def path = inputFile.absolutePath
        if (path.endsWith(".jar")) {
        //对jar进行字节码注入,dex任务之前会生成一个jar
            NuwaProcessor.processJar(hashFile, inputFile, patchDir, hashMap, includePackage, excludeClass)
        }
    }
}
//nuwaJarBeforeDexTask在dexTask之前,在dexTask原来之前所有task之后
def nuwaJarBeforeDexTask = project.tasks[nuwaJarBeforeDex]
nuwaJarBeforeDexTask.dependsOn dexTask.taskDependencies.getDependencies(dexTask)
dexTask.dependsOn nuwaJarBeforeDexTask
nuwaJarBeforeDexTask.doFirst(nuwaPrepareClosure)//创建文件夹等初始化
nuwaJarBeforeDexTask.doLast(copyMappingClosure)//copy mapping 善后工作
nuwaPatchTask.dependsOn nuwaJarBeforeDexTask//等注入字节码完成后,dex patch
beforeDexTasks.add(nuwaJarBeforeDexTask)
```
先看没有开启multiDex即没有preTask的过程,整个依赖为 dex之前的所有task->获得dex之前的所有输入文件->字节码注入->dex patch -> dex


再看有preDex的:

```groovy
//preDex会在dex任务之前把所有的库工程和第三方jar包提前打成dex，
//下次运行只需重新dex被修改的库，以此节省时间。
if (preDexTask) {
    def nuwaJarBeforePreDex = "nuwaJarBeforePreDex${variant.name.capitalize()}"
    project.task(nuwaJarBeforePreDex) << {
        Set<File> inputFiles = preDexTask.inputs.files.files
        inputFiles.each { inputFile ->
            def path = inputFile.absolutePath
            if (NuwaProcessor.shouldProcessPreDexJar(path)) {
                //向classes.jar注入字节码
                NuwaProcessor.processJar(hashFile, inputFile, patchDir, hashMap, includePackage, excludeClass)
            }
        }
    }
    def nuwaJarBeforePreDexTask = project.tasks[nuwaJarBeforePreDex]
    nuwaJarBeforePreDexTask.dependsOn preDexTask.taskDependencies.getDependencies(preDexTask)
    preDexTask.dependsOn nuwaJarBeforePreDexTask
    nuwaJarBeforePreDexTask.doFirst(nuwaPrepareClosure)
    def nuwaClassBeforeDex = "nuwaClassBeforeDex${variant.name.capitalize()}"
    project.task(nuwaClassBeforeDex) << {
        Set<File> inputFiles = dexTask.inputs.files.files
        inputFiles.each { inputFile ->
            def path = inputFile.absolutePath
            if (path.endsWith(".class") && !path.contains("/R\$") && !path.endsWith("/R.class") && !path.endsWith("/BuildConfig.class")) {
                if (NuwaSetUtils.isIncluded(path, includePackage)) {
                    if (!NuwaSetUtils.isExcluded(path, excludeClass)) {
                        def bytes = NuwaProcessor.processClass(inputFile)
                        path = path.split("${dirName}/")[1]
                        def hash = DigestUtils.shaHex(bytes)
                        hashFile.append(NuwaMapUtils.format(path, hash))
                        //与上一个release版本hash值不一样则复制出来,作为patch.jar的组成部分
                        if (NuwaMapUtils.notSame(hashMap, path, hash)) {
                            NuwaFileUtils.copyBytesToFile(inputFile.bytes, NuwaFileUtils.touchFile(patchDir, path))
                        }
                    }
                }
            }
        }
    }
    def nuwaClassBeforeDexTask = project.tasks[nuwaClassBeforeDex]
    nuwaClassBeforeDexTask.dependsOn dexTask.taskDependencies.getDependencies(dexTask)
    dexTask.dependsOn nuwaClassBeforeDexTask
    nuwaClassBeforeDexTask.doLast(copyMappingClosure)
    nuwaPatchTask.dependsOn nuwaClassBeforeDexTask
    beforeDexTasks.add(nuwaClassBeforeDexTask)
```

有无preDex:当有preDex时,dex任务会把preDex生成的jar文件和主工程中的class文件一起生成class.dex，没有时直接是一个jar

最后:

```groovy
 project.task(NUWA_PATCHES) << {
    patchList.each { patchDir ->
        NuwaAndroidUtils.dex(project, patchDir)
    }
}
beforeDexTasks.each {
    project.tasks[NUWA_PATCHES].dependsOn it
}
```

dex patch依赖于字节码注入。

##### 字节码注入

使用asm进行字节码注入:

```groovy
static processJar(File hashFile, File jarFile, File patchDir, Map map, HashSet<String> includePackage, HashSet<String> excludeClass) {
        if (jarFile) {
            def optJar = new File(jarFile.getParent(), jarFile.name + ".opt")

            def file = new JarFile(jarFile);
            Enumeration enumeration = file.entries();
            JarOutputStream jarOutputStream = new JarOutputStream(new FileOutputStream(optJar));

            while (enumeration.hasMoreElements()) {
                JarEntry jarEntry = (JarEntry) enumeration.nextElement();
                String entryName = jarEntry.getName();
                ZipEntry zipEntry = new ZipEntry(entryName);

                InputStream inputStream = file.getInputStream(jarEntry);
                jarOutputStream.putNextEntry(zipEntry);
                //枚举jar包中的文件
                if (shouldProcessClassInJar(entryName, includePackage, excludeClass)) {
                    def bytes = referHackWhenInit(inputStream);
                    jarOutputStream.write(bytes);

                    def hash = DigestUtils.shaHex(bytes)
                    hashFile.append(NuwaMapUtils.format(entryName, hash))

                    if (NuwaMapUtils.notSame(map, entryName, hash)) {
                        NuwaFileUtils.copyBytesToFile(bytes, NuwaFileUtils.touchFile(patchDir, entryName))
                    }
                } else {
                    jarOutputStream.write(IOUtils.toByteArray(inputStream));
                }
                jarOutputStream.closeEntry();
            }
            jarOutputStream.close();
            file.close();

            if (jarFile.exists()) {
                jarFile.delete()
            }
            optJar.renameTo(jarFile)
        }

    }

```

最关键的注入代码:

```groovy
//refer hack class when object init
    private static byte[] referHackWhenInit(InputStream inputStream) {
        ClassReader cr = new ClassReader(inputStream);
        ClassWriter cw = new ClassWriter(cr, 0);
        ClassVisitor cv = new ClassVisitor(Opcodes.ASM4, cw) {
            @Override
            public MethodVisitor visitMethod(int access, String name, String desc,
                                             String signature, String[] exceptions) {

                MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
                mv = new MethodVisitor(Opcodes.ASM4, mv) {
                    @Override
                    void visitInsn(int opcode) {
                    //如果是构造函数
                        if ("<init>".equals(name) && opcode == Opcodes.RETURN) {
                            super.visitLdcInsn(Type.getType("Lcn/jiajixin/nuwa/Hack;"));
                        }
                        super.visitInsn(opcode);
                    }
                }
                return mv;
            }

        };
        cr.accept(cv, 0);
        return cw.toByteArray();
    }

```

dex patch:

`dx --dex --output=patch.jar classDir` classDir是注入字节码后的补丁目录

```groovy public static dex(Project project, File classDir) {
        if (classDir.listFiles().size()) {
            def sdkDir
  
            Properties properties = new Properties()
            File localProps = project.rootProject.file("local.properties")
            if (localProps.exists()) {
                properties.load(localProps.newDataInputStream())
                sdkDir = properties.getProperty("sdk.dir")
            } else {
                sdkDir = System.getenv("ANDROID_HOME")
            }
            if (sdkDir) {
                def cmdExt = Os.isFamily(Os.FAMILY_WINDOWS) ? '.bat' : ''
                def stdout = new ByteArrayOutputStream()
                project.exec {
                    commandLine "${sdkDir}/build-tools/${project.android.buildToolsVersion}/dx${cmdExt}",
                            '--dex',
                            "--output=${new File(classDir.getParent(), PATCH_NAME).absolutePath}",
                            "${classDir.absolutePath}"
                    standardOutput = stdout
                }
                def error = stdout.toString().trim()
                if (error) {
                    println "dex error:" + error
                }
            } else {
                throw new InvalidUserDataException('$ANDROID_HOME is not defined')
            }
        }
    }
```

#### 注意点

* 不能注入字节码的类是Application的子类，因为Hack.apk在程序运行之前没有被加载，所以如果Application类中引用了Hack.apk中的Hack.class文件，则会报Class找不到的异常，之后也永远找不到了。所以这个类不能注入字节码，但是需要提前加载初始化方法中动态加载该Hack.apk。
* 发版时的mapping文件以及所有class文件的hash值的文件需要保持下来打patch使用。


[坑](http://blog.csdn.net/sbsujjbcy/article/details/51028027)

* Nuwa处理的是混淆后的jar，混淆后的jar包名和类名发生了变化，你再使用配置进去的excludeClass是无法主动不进行字节码注入处理的，除非你加进去的是混淆后的类名。可以拿到Build目录下的mapping.txt然后反混淆即可。


![](/img/2017-04-29-gradleAndNuwa/14934363009939.jpg)


最后,很好的开源项目:https://github.com/Livyli/AndHotFix,进行了完善与改造,值得研究


#### Transform

[gradle1.5.0以后的新特性](http://blog.csdn.net/sbsujjbcy/article/details/50839263)

>The Dex class is gone. You cannot access it anymore through the variant API (the getter is still there for now but will throw an exception)
Transform can only be registered globally which applies them to all the variants. We'll improve this shortly.

也就是说,以后我们不能直接通过`variant API`去操作`dex`,通过Transform吧！

作用就是第三方插件在class文件转为为dex文件前操作编译好的class文件，所以只有这一步我已开始还想了半天它到底怎么指定的插入的位置的。。

![](/img/2017-04-29-gradleAndNuwa/14934406958131.jpg)


可以看到由transform + (contentType)[Classes | Resources] + with + [name] + for + flavor|buildType


##### 注册

```groovy
//插件的apply方法最前面注册
def isApp = project.plugins.hasPlugin(AppPlugin)
if (isApp) {
      def android = project.extensions.getByType(AppExtension)
      def transform = new TransformImpl(project)
      android.registerTransform(transform)
} class TransformImpl extends Transform {
    Project project
    public TransformTest(Project project) {
        this.project = project
    }

    @Override
    String getName() {
        return "TransformImpl"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }


    @Override
    Set<QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }


    @Override
    boolean isIncremental() {
        return false;
    }


    @Override
    void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs, TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
        //todo 在这里进行字节码注入即可
    }

}
```





