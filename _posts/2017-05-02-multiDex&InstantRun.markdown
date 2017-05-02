---
layout:     post
title:      "MultiDex & Instant Run"
date:       2017-05-02
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - 插件化
    - 热补丁
---

### ClassLoader


![](/img/2017-05-02-multidex/14932770207728.png)

* `PathClassLoader`是Android应用中的默认加载器，`PathClassLoader`只能加载`/data/app`中的apk，也就是已经安装到手机中的apk。这个也是`PathClassLoader`作为默认的类加载器的原因，因为一般程序都是安装了，在打开，这时候`PathClassLoader`就去加载指定的apk(解压成dex，然后在优化成odex)就可以了。

* `DexClassLoader`可以加载任何路径的`apk/dex/jar`，`PathClassLoader`只能加载已安装到系统中（即/data/app目录下）的apk文件。

#### DexPathList

```java
// definingContext对应的就是当前classLoader
// dexPath对应的就是上面传进来的apk/dex/jar的路径
// libraryPath就是上面传进来的加载的时候需要用到的lib库的目录，这个一般不用
// optimizedDirectory就是上面传进来的dex的输出路径
public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                       suppressedExceptions);
}
// files是一个ArrayList<File>列表，它对应的就是apk/dex/jar文件，因为我们可以指定多个文件。
// optimizedDirectory是前面传入dex的输出路径
// suppressedExceptions为一个异常列表
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,ArrayList<IOException> suppressedExceptions) {
    ArrayList<Element> elements = new ArrayList<Element>();
    /*
     * Open all files and load the (direct or contained) dex files
     * up front.
     */
    for (File file : files) {
        File zip = null;
        DexFile dex = null;
        String name = file.getName();

        // 如果是一个dex文件
        if (name.endsWith(DEX_SUFFIX)) {
            // Raw dex file (not inside a zip/jar).
            try {
                dex = loadDexFile(file, optimizedDirectory);
            } catch (IOException ex) {
                System.logE("Unable to load dex file: " + file, ex);
            }
        // 如果是一个apk或者jar或者zip文件
        } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                || name.endsWith(ZIP_SUFFIX)) {
            zip = file;

            try {
                dex = loadDexFile(file, optimizedDirectory);
            } catch (IOException suppressed) {
                /*
                 * IOException might get thrown "legitimately" by the DexFile constructor if the
                 * zip file turns out to be resource-only (that is, no classes.dex file in it).
                 * Let dex == null and hang on to the exception to add to the tea-leaves for
                 * when findClass returns null.
                 */
                suppressedExceptions.add(suppressed);
            }
        } else if (file.isDirectory()) {
            // We support directories for looking up resources.
            // This is only useful for running libcore tests.
            elements.add(new Element(file, true, null, null));
        } else {
            System.logW("Unknown file type for: " + file);
        }

        if ((zip != null) || (dex != null)) {
            elements.add(new Element(file, false, zip, dex));
        }
    }

    return elements.toArray(new Element[elements.size()]);
}
```

* 在`DexClassLoader`我们指定了加载的`apk/dex/jar`文件和`dex`输出路径`optimizedDirectory`，它最终会被解析得到`DexFile`文件。 
* 将`DexFile`文件对象放在`Element`对象里面，它对应的就是`Element`对象的`dexFile`成员变量。 
* 将这个`Element`对象放在一个`Element[]`数组中，然后将这个数组返回给`DexPathList`的`dexElements`成员变量。 
* `DexPathList`是`BaseDexClassLoader`的一个成员变量。


![](/img/2017-05-02-multidex/14932823251895.jpg)

### MultiDex

主要只有两个文件:`MultiDex`和`MultiDexExtractor`

* `Context.getFilesDir()`：返回`/data/data/<packagename>/files`目录，一般通过`openFileOutput`方法输出文件到该目录
* `ApplicationInfo.dataDir`: 返回`/data/data/<packagename>`目录

```java
public static void install(Context context) {
        Log.i("MultiDex", "install");
        if(IS_VM_MULTIDEX_CAPABLE) {
        //虚拟机自身支持,就不用我们瞎BB了
            Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
        } else if(VERSION.SDK_INT < 4) {
        //小于4不能用
            throw new RuntimeException("Multi dex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
        } else {
            try {
                ApplicationInfo e = getApplicationInfo(context);
                if(e == null) {
                    return;
                }
                Set var2 = installedApk;
                synchronized(installedApk) {
                    String apkPath = e.sourceDir;
                    if(installedApk.contains(apkPath)) {
                        return;
                    }
                    installedApk.add(apkPath);
                    if(VERSION.SDK_INT > 20) {
                        Log.w("MultiDex", "MultiDex is not guaranteed to work in SDK version " + VERSION.SDK_INT + ": SDK version higher than " + 20 + " should be backed by " + "runtime with built-in multidex capabilty but it\'s not the " + "case here: java.vm.version=\"" + System.getProperty("java.vm.version") + "\"");
                    }
                    ClassLoader loader;
                    try {
                        loader = context.getClassLoader();
                    } catch (RuntimeException var9) {
                        Log.w("MultiDex", "Failure while trying to obtain Context class loader. Must be running in test mode. Skip patching.", var9);
                        return;
                    }
                    if(loader == null) {
                        Log.e("MultiDex", "Context class loader is null. Must be running in test mode. Skip patching.");
                        return;
                    }
                    try {
                        clearOldDexDir(context);
                    } catch (Throwable var8) {
                        Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var8);
                    }

                    File dexDir = new File(e.dataDir, SECONDARY_FOLDER_NAME);
                    ////把dex文件缓存到/data/data/<packagename>/code_cache/secondary-dexes/目录
                    List files = MultiDexExtractor.load(context, e, dexDir, false);             
                    //检查是否为zip文件
                    if(checkValidZipFiles(files)) {
                        installSecondaryDexes(loader, dexDir, files);
                    } else {
                        Log.w("MultiDex", "Files were not valid zip files.  Forcing a reload.");
                        files = MultiDexExtractor.load(context, e, dexDir, true);
                        if(!checkValidZipFiles(files)) {
                            throw new RuntimeException("Zip files were not valid.");
                        }
                        installSecondaryDexes(loader, dexDir, files);
                    }
                }
            } catch (Exception var11) {
                Log.e("MultiDex", "Multidex installation failure", var11);
                throw new RuntimeException("Multi dex installation failed (" + var11.getMessage() + ").");
            }
            Log.i("MultiDex", "install done");
        }
    }
```

#### load

解压apk文件中的classes2.dex、classes3.dex等文件解压到dexDir目录中,当然如果已经有了直接加载好啦


```java
static List<File> load(Context context, ApplicationInfo applicationInfo, File dexDir, boolean forceReload) throws IOException {
///data/app/com.sina.weibo.sdk.demo-1.apk
    Log.i("MultiDex", "MultiDexExtractor.load(" + applicationInfo.sourceDir + ", " + forceReload + ")");
    File sourceApk = new File(applicationInfo.sourceDir);
    long currentCrc = getZipCrc(sourceApk);
    List files;
        //使用SharedPreferences存时间戳和crc来决定是否更新，其实就是用一个生命周期与dex文件相同的位置去存储数据就好
    if(!forceReload && !isModified(context, sourceApk, currentCrc)) {
        try {
            files = loadExistingExtractions(context, sourceApk, dexDir);
        } catch (IOException var9) {
            Log.w("MultiDex", "Failed to reload existing extracted secondary dex files, falling back to fresh extraction", var9);
            files = performExtractions(sourceApk, dexDir);
            putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
        }
    } else {
        Log.i("MultiDex", "Detected that extraction must be performed.");
        files = performExtractions(sourceApk, dexDir);
        putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
    }
    Log.i("MultiDex", "load found " + files.size() + " secondary dex files");
    return files;
}
```

可以看到,如果没有更改,直接loadExisting:

```java
private static List<File> loadExistingExtractions(Context context, File sourceApk, File dexDir) throws IOException {
    Log.i("MultiDex", "loading existing secondary dex files");
    String extractedFilePrefix = sourceApk.getName() + ".classes";
    //有多少个dex
    int totalDexNumber = getMultiDexPreferences(context).getInt("dex.number", 1);
    ArrayList files = new ArrayList(totalDexNumber);
    for(int secondaryNumber = 2; secondaryNumber <= totalDexNumber; ++secondaryNumber) {
        String fileName = extractedFilePrefix + secondaryNumber + ".zip";
        File extractedFile = new File(dexDir, fileName);
        if(!extractedFile.isFile()) {
            throw new IOException("Missing extracted secondary dex file \'" + extractedFile.getPath() + "\'");
        }
        files.add(extractedFile);
        if(!verifyZipFile(extractedFile)) {
            Log.i("MultiDex", "Invalid zip file: " + extractedFile);
            throw new IOException("Invalid ZIP file.");
        }
    }
    return files;
}

```

所有`secondary dex`输出为`zip`文件，这样是为了保持和`DexClassLoader`和`DexPathList`的兼容

```java
private static List<File> performExtractions(File sourceApk, File dexDir) throws IOException {
    String extractedFilePrefix = sourceApk.getName() + ".classes";
    prepareDexDir(dexDir, extractedFilePrefix);
    ArrayList files = new ArrayList();
    ZipFile apk = new ZipFile(sourceApk);
    try {
        int e = 2;
        for(ZipEntry dexFile = apk.getEntry("classes" + e + ".dex"); dexFile != null; dexFile = apk.getEntry("classes" + e + ".dex")) {
            String fileName = extractedFilePrefix + e + ".zip";
            File extractedFile = new File(dexDir, fileName);
            files.add(extractedFile);
            Log.i("MultiDex", "Extraction is needed for file " + extractedFile);
            int numAttempts = 0;
            boolean isExtractionSuccessful = false;
            while(numAttempts < 3 && !isExtractionSuccessful) {
                ++numAttempts;
                    //extract得到的是zip文件,简言之，这个方法做的事情就是，将Apk文件解压后得到的classes2.dex, …, classesN.dex文件的内容依次写入到/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes2.zip, …, /data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classesN.zip压缩文件的classes.dex文件中。
                extract(apk, dexFile, extractedFile, extractedFilePrefix);
                isExtractionSuccessful = verifyZipFile(extractedFile);
                Log.i("MultiDex", "Extraction " + (isExtractionSuccessful?"success":"failed") + " - length " + extractedFile.getAbsolutePath() + ": " + extractedFile.length());
                if(!isExtractionSuccessful) {
                    extractedFile.delete();
                    if(extractedFile.exists()) {
                        Log.w("MultiDex", "Failed to delete corrupted secondary dex \'" + extractedFile.getPath() + "\'");
                    }
                }
            }
            if(!isExtractionSuccessful) {
                throw new IOException("Could not create zip file " + extractedFile.getAbsolutePath() + " for secondary dex (" + e + ")");
            }
            ++e;
        }
    } finally {
        try {
            apk.close();
        } catch (IOException var16) {
            Log.w("MultiDex", "Failed to close resource", var16);
        }
    }
    return files;
}
```

#### install

install的输入是zip,并根据api分V4/V14/V19三种进行注册。

注册过程就是classLoader的dexPath过程。

```java
private static void install(ClassLoader loader, List<File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
    Field pathListField = MultiDex.findField(loader, "pathList");
    Object dexPathList = pathListField.get(loader);
    MultiDex.expandFieldArray(dexPathList, "dexElements",makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory));
}

private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
    Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", new Class[]{ArrayList.class, File.class});
    return (Object[])((Object[])makeDexElements.invoke(dexPathList, new Object[]{files, optimizedDirectory}));
}
```

有空可以看一下dx的`-multi-dex`参数是如何拆包的。

### Instant-run

和Nuwa一样,也是一个Gradle插件一个`sdk:gradle plugin 2.0.0-alpha1`和`instant-run.jar`
`instance-run.jar`在build期间被打包到apk里面,避免我们自己去compile

* `hot swap`是三种类型中最快生效的，它的可以作用在一般代码的修改上
* `warm swap`是针对资源的修改，需要你重启对应的Activity。
* `cold swap`是最慢的一种，它需要你重启整个app，并且需要你的Android API在21或者以上，对于API20以下的，则会和原来一样，重新构建并部署应用。


![](/img/2017-05-02-multidex/14934468299959.jpg)
我随便选了个weibo sdk的test工程做了修改,`instant-run`后可以看到确实生成了`AppPatchesLoaderImpl`和一些`@override`文件,至于有三个是因为有一些内部类。


![](/img/2017-05-02-multidex/14934470238756.jpg)

![](/img/2017-05-02-multidex/14934471773422.jpg)

![](/img/2017-05-02-multidex/14934471953188.jpg)
所有方法都被asm注入为以下套路:

```
IncrementalChange var2 = $change;
if(var2 != null){
    var2.access$dispatch(...。。)
}else{
    original code
}  
```
 

![](/img/2017-05-02-multidex/14934473559925.jpg)

![](/img/2017-05-02-multidex/14934473803670.jpg)
这些都是由`plugin`通过`transform api`实现的。

#### sdk


![](/img/2017-05-02-multidex/14934474788035.jpg)
##### createResources

```java
  private void createResources(long apkModified)
  {
   //"/data/data/" + applicationId + "/files/instant-run/inbox"
    FileManager.checkInbox();
    
    File file = FileManager.getExternalResourceFile();
    this.externalResourcePath = (file != null ? file.getPath() : null);
    if (Log.isLoggable("InstantRun", 2)) {
      Log.v("InstantRun", "Resource override is " + this.externalResourcePath);
    }
    if (file != null) {
      try
      {
        long resourceModified = file.lastModified();
        if (Log.isLoggable("InstantRun", 2))
        {
          Log.v("InstantRun", "Resource patch last modified: " + resourceModified);
          Log.v("InstantRun", "APK last modified: " + apkModified + " " + (apkModified > resourceModified ? ">" : "<") + " resource patch");
        }
        if ((apkModified == 0L) || (resourceModified <= apkModified))
        {
          if (Log.isLoggable("InstantRun", 2)) {
            Log.v("InstantRun", "Ignoring resource file, older than APK");
          }
          this.externalResourcePath = null;
        }
      }
      catch (Throwable t)
      {
        Log.e("InstantRun", "Failed to check patch timestamps", t);
      }
    }
  }
```

##### setupClassLoaders

[非常棒的文章👍](http://w4lle.github.io/2016/05/02/从Instant%20run谈Android替换Application和动态加载机制/)


![](/img/2017-05-02-multidex/14934475142698.jpg)
再看inject方法:


```java
public static ClassLoader inject(ClassLoader classLoader, String nativeLibraryPath, String codeCacheDir, List<String> dexes){
    IncrementalClassLoader incrementalClassLoader = new IncrementalClassLoader(classLoader, nativeLibraryPath, codeCacheDir, dexes); 
    setParent(classLoader, incrementalClassLoader); 
    return incrementalClassLoader;
 }
```
`BootStrapApplication`由`IncrementalClassLoader`加载

```java
public class IncrementalClassLoader
  extends ClassLoader
{
  public static final boolean DEBUG_CLASS_LOADING = false;
  private final DelegateClassLoader delegateClassLoader;
  //cache是optimized dir
  public IncrementalClassLoader(ClassLoader original, String nativeLibraryPath, String codeCacheDir, List<String> dexes)
  {
    super(original.getParent());
    //创建真正的classLoader
    this.delegateClassLoader = createDelegateClassLoader(nativeLibraryPath, codeCacheDir, dexes, original);
  }
  
  public Class<?> findClass(String className)
    throws ClassNotFoundException
  {
    try
    {
      return this.delegateClassLoader.findClass(className);
    }
    catch (ClassNotFoundException e)
    {
      throw e;
    }
  }
  
  private static class DelegateClassLoader
    extends BaseDexClassLoader
  {
    private DelegateClassLoader(String dexPath, File optimizedDirectory, String libraryPath, ClassLoader parent)
    {
      super(optimizedDirectory, libraryPath, parent);
    }
    
    public Class<?> findClass(String name)
      throws ClassNotFoundException
    {
      try
      {
        return super.findClass(name);
      }
      catch (ClassNotFoundException e)
      {
        throw e;
      }
    }
  }
  
  private static DelegateClassLoader createDelegateClassLoader(String nativeLibraryPath, String codeCacheDir, List<String> dexes, ClassLoader original)
  {
    String pathBuilder = createDexPath(dexes);
    return new DelegateClassLoader(pathBuilder, new File(codeCacheDir), nativeLibraryPath, original, null);
  }
  
  private static String createDexPath(List<String> dexes)
  {
    StringBuilder pathBuilder = new StringBuilder();
    boolean first = true;
    for (String dex : dexes)
    {
      if (first) {
        first = false;
      } else {
        pathBuilder.append(File.pathSeparator);
      }
      pathBuilder.append(dex);
    }
    if (Log.isLoggable("InstantRun", 2)) {
      Log.v("InstantRun", "Incremental dex path is " + 
        BootstrapApplication.join('\n', dexes));
    }
    return pathBuilder.toString();
  }
  //设置parent,双亲委派模式
  private static void setParent(ClassLoader classLoader, ClassLoader newParent)
  {
    try
    {
      Field parent = ClassLoader.class.getDeclaredField("parent");
      parent.setAccessible(true);
      parent.set(classLoader, newParent);
    }
    catch (IllegalArgumentException e)
    {
      throw new RuntimeException(e);
    }
    catch (IllegalAccessException e)
    {
      throw new RuntimeException(e);
    }
    catch (NoSuchFieldException e)
    {
      throw new RuntimeException(e);
    }
  }
}
```

##### dexPath

`instant-run`会把所有`classes.dex`放到一个压缩包里,所以我们要自定义`classloader`去解压加载

`dexPath`是`/data/data/package_name/files/instant-run/dex`目录下的dex列表

```java
 public static List<String> getDexList(Context context, long apkModified)
  {
  //"/data/data/" + applicationId + "/files/instant-run"
    File dataFolder = getDataFolder();
   //hotSwap回留下临时文件 
    long newestHotswapPatch = getMostRecentTempDexTime(dataFolder);
    //dexFolder还没有,那就先创建吧
    File dexFolder = getDexFileFolder(dataFolder, false);
    
    boolean extractedSlices = false;
    File[] dexFiles;
    if ((dexFolder == null) || (!dexFolder.isDirectory()))
    {
      if (Log.isLoggable("InstantRun", 2)) {
        Log.v("InstantRun", "No local dex slice folder: First run since installation.");
      }
      dexFolder = getDexFileFolder(dataFolder, true);
      if (dexFolder == null)
      {
        Log.wtf("InstantRun", "Couldn't create dex code folder");
        return Collections.emptyList();
      }
      File[] dexFiles = extractSlices(dexFolder, null, -1L);
      extractedSlices = dexFiles.length > 0;
    }
    else
    {
      dexFiles = dexFolder.listFiles();
    }
    if ((dexFiles == null) || (dexFiles.length == 0))
    {
      if (Log.isLoggable("InstantRun", 2)) {
        Log.v("InstantRun", "Cannot find dex classes, not patching them in");
      }
      return Collections.emptyList();
    }
    long newestColdswapPatch = apkModified;
    if ((!extractedSlices) && (dexFiles.length > 0))
    {
      long oldestColdSwapPatch = apkModified;
      for (File dex : dexFiles)
      {
        long dexModified = dex.lastModified();
        oldestColdSwapPatch = Math.min(dexModified, oldestColdSwapPatch);
        newestColdswapPatch = Math.max(dexModified, newestColdswapPatch);
      }
      if (oldestColdSwapPatch < apkModified)
      {
        if (Log.isLoggable("InstantRun", 2)) {
          Log.v("InstantRun", "One or more slices were older than APK: extracting newer slices");
        }
        //解压instant-run.zip中的dex
        dexFiles = extractSlices(dexFolder, dexFiles, apkModified);
      }
    }
    else if (newestHotswapPatch > 0L)
    {
      purgeTempDexFiles(dataFolder);
    }
    if (newestHotswapPatch > newestColdswapPatch)
    {
      String message = "Your app does not have the latest code changes because it was restarted manually. Please run from IDE instead.";
      if (Log.isLoggable("InstantRun", 2)) {
        Log.v("InstantRun", message);
      }
      Restarter.showToastWhenPossible(context, message);
    }
    List<String> list = new ArrayList(dexFiles.length);
    for (File dex : dexFiles) {
      if (dex.getName().endsWith(".dex")) {
        list.add(dex.getPath());
      }
    }
    Collections.sort(list, Collections.reverseOrder());
    
    return list;
  }
  
  ```
  

![](/img/2017-05-02-multidex/14935234377075.png)
所以前面使用`delegateClassLoader`是用来加载这些dex的。

##### createRealApplication


![](/img/2017-05-02-multidex/14935170794564.jpg)

如果用户自定义了`Application`,在`Instant-run-bootstrap.jar`中,`AppInfo.applicationClass`为真实`Application`,因此把它赋值给`realApplication`变量。

#### onCreate

```java
public void onCreate(){
    if (!AppInfo.usingApkSplits){
      MonkeyPatcher.monkeyPatchApplication(this, this, this.realApplication, this.externalResourcePath);
      MonkeyPatcher.monkeyPatchExistingResources(this, this.externalResourcePath, null);
    }else{
      MonkeyPatcher.monkeyPatchApplication(this, this, this.realApplication, null);
    }
    super.onCreate();
```

##### MonkeyPatcher.monkeyPatchApplication

很长的函数

1. 把对应的`application`替换成我们真正的`applictaion`。
2. 替换资源路径，老的资源路径是`/data/app/[package name]-1.apk`，新的资源路径是`/data/data/[applicationId]/files/instant-run/resources.ap_`。
3. 把真正`application`的`LoadedApk`替换成`BootstrapApplication`的，为什么要这么做呢，因为`LoadedApk`中持有了`ClassLoader`，这样替换以后，我们程序中加载类都会使用`BootstrapApplication`的`LoadedApk`，从而使用它的`ClassLoader`，而在之前我们已经把`ClassLoade`的父`loader`设置成了`IncrementalClassLoader`，绕了这么一大圈，其实就是为了［使用`IncrementalClassLoader`去加载类］。


* 替换`ActivityThread`的`mInitialApplication`为`realApplication`
* 替换`mAllApplications` 中所有的`Application`为`realApplication`


```java
public static void monkeyPatchApplication(Context context, Application bootstrap, Application realApplication, String externalResourceFile){
    try{
      Class<?> activityThread = Class.forName("android.app.ActivityThread");
      Object currentActivityThread = getActivityThread(context, activityThread);   
      Field mInitialApplication = activityThread.getDeclaredField("mInitialApplication");
      mInitialApplication.setAccessible(true);
      Application initialApplication = (Application)mInitialApplication.get(currentActivityThread);
      //设置mInitialApplicaion
      if ((realApplication != null) && (initialApplication == bootstrap)) {
        mInitialApplication.set(currentActivityThread, realApplication);
      }
      if (realApplication != null)
      {
        Field mAllApplications = activityThread.getDeclaredField("mAllApplications");
        mAllApplications.setAccessible(true);
        
        List<Application> allApplications = (List)mAllApplications.get(currentActivityThread);
        for (int i = 0; i < allApplications.size(); i++) {
          if (allApplications.get(i) == bootstrap) {
            allApplications.set(i, realApplication);
          }
        }
      }
      Class<?> loadedApkClass;
      try
      {
        loadedApkClass = Class.forName("android.app.LoadedApk");
      }
      catch (ClassNotFoundException e)
      {
        Class<?> loadedApkClass;
        loadedApkClass = Class.forName("android.app.ActivityThread$PackageInfo");
      }
      Field mApplication = loadedApkClass.getDeclaredField("mApplication");
      mApplication.setAccessible(true);
      Field mResDir = loadedApkClass.getDeclaredField("mResDir");
      mResDir.setAccessible(true);
      
      Field mLoadedApk = null;
      try
      {
        mLoadedApk = Application.class.getDeclaredField("mLoadedApk");
      }
      catch (NoSuchFieldException localNoSuchFieldException) {}
      for (String fieldName : new String[] { "mPackages", "mResourcePackages" })
      {
        Field field = activityThread.getDeclaredField(fieldName);
        field.setAccessible(true);
        Object value = field.get(currentActivityThread);
        for (Map.Entry<String, WeakReference<?>> entry : ((Map)value).entrySet())
        {
          Object loadedApk = ((WeakReference)entry.getValue()).get();
          if (loadedApk != null) {
            if (mApplication.get(loadedApk) == bootstrap)
            {
              if (realApplication != null) {
                mApplication.set(loadedApk, realApplication);
              }
              if (externalResourceFile != null) {
                mResDir.set(loadedApk, externalResourceFile);
              }
              /**
              由于BootstrapApplication在attachBaseContext方法中就将其已经替换为了IncrementalClassLoader，所以代码4处反射将BootstrapApplication的mLoadedApk赋值给了MyApplication，那么接下来MyApplication的所有类的加载都将由IncrementalClassLoader来负责。
              **/
              if ((realApplication != null) && (mLoadedApk != null)) {
                mLoadedApk.set(realApplication, loadedApk);
              }
            }
          }
        }
      }
    }
    catch (Throwable e)
    {
      throw new IllegalStateException(e);
    }
  }
  ```



![](/img/2017-05-02-multidex/14935174812314.jpg)

首先直接获取`ActivityThread.currentActivityThread`,如果获取不到从`mLoadedApk.mActivityThread`中获取


>Instant run的重启更新机制更像保守方案即单ClassLoader方案，首先，该种方案只有一个ClassLoader，只不过是通过替换Application达到的替换mLoadedApk进而替换ClassLoader的目的，并没有涉及到缓存mPackage然后dexList也是它自己维护的。

##### MonkeyPatchExistingResources todo

* 如果`resource.ap_`文件有改变，那么新建一个`AssetManager`对象`newAssetManager`，然后用`newAssetManager`对象替换所有当前`Resource、Resource.Theme`的`mAssets`成员变量。 
* 如果当前的已经有`Activity`启动了，还需要替换所有`Activity`中`mAssets`成员变量

```java
public static void monkeyPatchExistingResources(Context context, String externalResourceFile, Collection<Activity> activities)
  {
    if (externalResourceFile == null) {
      return;
    }
    try
    {
      newAssetManager = (AssetManager)AssetManager.class.getConstructor(new Class[0]).newInstance(new Object[0]);
      //设置AssetManager新的assetPath为externalResourceFile
      Method mAddAssetPath = AssetManager.class.getDeclaredMethod("addAssetPath", new Class[] { String.class });
      mAddAssetPath.setAccessible(true);
      if (((Integer)mAddAssetPath.invoke(newAssetManager, new Object[] { externalResourceFile })).intValue() == 0) {
        throw new IllegalStateException("Could not create new AssetManager");
      }
      Method mEnsureStringBlocks = AssetManager.class.getDeclaredMethod("ensureStringBlocks", new Class[0]);
      mEnsureStringBlocks.setAccessible(true);
      mEnsureStringBlocks.invoke(newAssetManager, new Object[0]);
      if (activities != null) {
        for (Activity activity : activities)
        {
          Resources resources = activity.getResources();
          try
          {
            Field mAssets = Resources.class.getDeclaredField("mAssets");
            mAssets.setAccessible(true);
            mAssets.set(resources, newAssetManager);
          }
          catch (Throwable ignore)
          {
            Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
            mResourcesImpl.setAccessible(true);
            Object resourceImpl = mResourcesImpl.get(resources);
            Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
            implAssets.setAccessible(true);
            implAssets.set(resourceImpl, newAssetManager);
          }
          Resources.Theme theme = activity.getTheme();
          try
          {
            try
            {
              Field ma = Resources.Theme.class.getDeclaredField("mAssets");
              ma.setAccessible(true);
              ma.set(theme, newAssetManager);
            }
            catch (NoSuchFieldException ignore)
            {
              Field themeField = Resources.Theme.class.getDeclaredField("mThemeImpl");
              themeField.setAccessible(true);
              Object impl = themeField.get(theme);
              Field ma = impl.getClass().getDeclaredField("mAssets");
              ma.setAccessible(true);
              ma.set(impl, newAssetManager);
            }
            Field mt = ContextThemeWrapper.class.getDeclaredField("mTheme");
            mt.setAccessible(true);
            mt.set(activity, null);
            Method mtm = ContextThemeWrapper.class.getDeclaredMethod("initializeTheme", new Class[0]);
            mtm.setAccessible(true);
            mtm.invoke(activity, new Object[0]);
            if (Build.VERSION.SDK_INT < 24)
            {
              Method mCreateTheme = AssetManager.class.getDeclaredMethod("createTheme", new Class[0]);
              mCreateTheme.setAccessible(true);
              Object internalTheme = mCreateTheme.invoke(newAssetManager, new Object[0]);
              Field mTheme = Resources.Theme.class.getDeclaredField("mTheme");
              mTheme.setAccessible(true);
              mTheme.set(theme, internalTheme);
            }
          }
          catch (Throwable e)
          {
            Log.e("InstantRun", "Failed to update existing theme for activity " + activity, e);
          }
          pruneResourceCaches(resources);
        }
      }
      Object references;
      Object references;
      if (Build.VERSION.SDK_INT >= 19)
      {
        Class<?> resourcesManagerClass = Class.forName("android.app.ResourcesManager");
        Method mGetInstance = resourcesManagerClass.getDeclaredMethod("getInstance", new Class[0]);
        mGetInstance.setAccessible(true);
        Object resourcesManager = mGetInstance.invoke(null, new Object[0]);
        try
        {
          Field fMActiveResources = resourcesManagerClass.getDeclaredField("mActiveResources");
          fMActiveResources.setAccessible(true);
          
          ArrayMap<?, WeakReference<Resources>> arrayMap = (ArrayMap)fMActiveResources.get(resourcesManager);
          references = arrayMap.values();
        }
        catch (NoSuchFieldException ignore)
        {
          Object references;
          Field mResourceReferences = resourcesManagerClass.getDeclaredField("mResourceReferences");
          mResourceReferences.setAccessible(true);
          
          references = (Collection)mResourceReferences.get(resourcesManager);
        }
      }
      else
      {
        activityThread = Class.forName("android.app.ActivityThread");
        Field fMActiveResources = activityThread.getDeclaredField("mActiveResources");
        fMActiveResources.setAccessible(true);
        Object thread = getActivityThread(context, activityThread);
        
        HashMap<?, WeakReference<Resources>> map = (HashMap)fMActiveResources.get(thread);
        references = map.values();
      }
      for (WeakReference<Resources> wr : (Collection)references)
      {
        Resources resources = (Resources)wr.get();
        if (resources != null)
        {
          try
          {
            Field mAssets = Resources.class.getDeclaredField("mAssets");
            mAssets.setAccessible(true);
            mAssets.set(resources, newAssetManager);
          }
          catch (Throwable ignore)
          {
            Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
            mResourcesImpl.setAccessible(true);
            Object resourceImpl = mResourcesImpl.get(resources);
            Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
            implAssets.setAccessible(true);
            implAssets.set(resourceImpl, newAssetManager);
          }
          resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
        }
      }
    }
    catch (Throwable e)
    {
      AssetManager newAssetManager;
      Class<?> activityThread;
      throw new IllegalStateException(e);
    }
  }
  ```
  
  太长了,先随便看看,等分析完资源再回头看吧


#### onCreate

继续看onCreate:启动server与AS交互

```java
    super.onCreate();
    if (AppInfo.applicationId != null) {
      try
      {
        boolean foundPackage = false;
        int pid = Process.myPid();
        ActivityManager manager = (ActivityManager)getSystemService("activity");
        
        List<ActivityManager.RunningAppProcessInfo> processes = manager.getRunningAppProcesses();
        boolean startServer;
        if ((processes != null) && (processes.size() > 1))
        {
          boolean startServer = false;
          for (ActivityManager.RunningAppProcessInfo processInfo : processes) {
            if (AppInfo.applicationId.equals(processInfo.processName))
            {
              foundPackage = true;
              if (processInfo.pid == pid)
              {
                startServer = true;
                break;
              }
            }
          }
          if ((!startServer) && (!foundPackage))
          {
            startServer = true;
            if (Log.isLoggable("InstantRun", 2)) {
              Log.v("InstantRun", "Multiprocess but didn't find process with package: starting server anyway");
            }
          }
        }
        else
        {
          startServer = true;
        }
        if (startServer) {
          Server.create(AppInfo.applicationId, this);
        }
      }
      catch (Throwable t)
      {
        if (Log.isLoggable("InstantRun", 2)) {
          Log.v("InstantRun", "Failed during multi process check", t);
        }
        Server.create(AppInfo.applicationId, this);
      }
    }
    //调用OnCreate
    if (this.realApplication != null) {
      this.realApplication.onCreate();
    }
```

#### Server

```java
this.mServerSocket = new LocalServerSocket(packageName);
startServer();
```


```java
Thread socketServerThread = new Thread(new SocketServerThread(null));
socketServerThread.start();
```

##### socketServerThread

```java
public void run()
    {
      try
      {
        for (;;)
        {
          LocalServerSocket serverSocket = Server.this.mServerSocket;
          if (serverSocket == null) {
            break;
          }
          //接受来自AS的连接
          LocalSocket socket = serverSocket.accept();
          if (Log.isLoggable("InstantRun", 2)) {
            Log.v("InstantRun", "Received connection from IDE: spawning connection thread");
          }
          Server.SocketServerReplyThread socketServerReplyThread = new Server.SocketServerReplyThread(Server.this, socket);
          
          socketServerReplyThread.run();
          if (Server.sWrongTokenCount > 50)
          {
            if (Log.isLoggable("InstantRun", 2)) {
              Log.v("InstantRun", "Stopping server: too many wrong token connections");
            }
            Server.this.mServerSocket.close();
            break;
          }
        }
      }
      catch (Throwable e)
      {
        if (Log.isLoggable("InstantRun", 2)) {
          Log.v("InstantRun", "Fatal error accepting connection on local socket", e);
        }
      }
    }
```

在`reply`中进行回复:`socket`开启后，开始读取数据，当读到1时，获取代码变化的`ApplicationPatch`列表，然后调用`handlePatches`来处理代码的变化。

```java
private void handle(DataInputStream input, DataOutputStream output)
      throws IOException
    {
      long magic = input.readLong();
      if (magic != 890269988L)
      {
        Log.w("InstantRun", "Unrecognized header format " + 
          Long.toHexString(magic));
        return;
      }
      int version = input.readInt();
      
      output.writeInt(4);
      if (version != 4)
      {
        Log.w("InstantRun", "Mismatched protocol versions; app is using version 4 and tool is using version " + version);
      }
      else
      {
        int message;
        for (;;)
        {
          message = input.readInt();
          switch (message)
          {
          case 7: 
            if (Log.isLoggable("InstantRun", 2)) {
              Log.v("InstantRun", "Received EOF from the IDE");
            }
            return;
          case 2: 
            boolean active = Restarter.getForegroundActivity(Server.this.mApplication) != null;
            output.writeBoolean(active);
            if (Log.isLoggable("InstantRun", 2)) {
              Log.v("InstantRun", "Received Ping message from the IDE; returned active = " + active);
            }
            break;
          case 3: 
            String path = input.readUTF();
            long size = FileManager.getFileSize(path);
            output.writeLong(size);
            if (Log.isLoggable("InstantRun", 2)) {
              Log.v("InstantRun", "Received path-exists(" + path + ") from the " + "IDE; returned size=" + size);
            }
            break;
          case 4: 
            long begin = System.currentTimeMillis();
            String path = input.readUTF();
            byte[] checksum = FileManager.getCheckSum(path);
            if (checksum != null)
            {
              output.writeInt(checksum.length);
              output.write(checksum);
              if (Log.isLoggable("InstantRun", 2))
              {
                long end = System.currentTimeMillis();
                String hash = new BigInteger(1, checksum).toString(16);
                Log.v("InstantRun", "Received checksum(" + path + ") from the " + "IDE: took " + (end - begin) + "ms to compute " + hash);
              }
            }
            else
            {
              output.writeInt(0);
              if (Log.isLoggable("InstantRun", 2)) {
                Log.v("InstantRun", "Received checksum(" + path + ") from the " + "IDE: returning <null>");
              }
            }
            break;
          case 5: 
            if (!authenticate(input)) {
              return;
            }
            Activity activity = Restarter.getForegroundActivity(Server.this.mApplication);
            if (activity != null)
            {
              if (Log.isLoggable("InstantRun", 2)) {
                Log.v("InstantRun", "Restarting activity per user request");
              }
              Restarter.restartActivityOnUiThread(activity);
            }
            break;
          case 1: 
            if (!authenticate(input)) {
              return;
            }
            List<ApplicationPatch> changes = ApplicationPatch.read(input);
            if (changes != null)
            {
              boolean hasResources = Server.hasResources(changes);
              int updateMode = input.readInt();
              updateMode = Server.this.handlePatches(changes, hasResources, updateMode);
              
              boolean showToast = input.readBoolean();
              
              output.writeBoolean(true);
              
              Server.this.restart(updateMode, hasResources, showToast);
            }
            break;
          case 6: 
            String text = input.readUTF();
            Activity foreground = Restarter.getForegroundActivity(Server.this.mApplication);
            if (foreground != null) {
              Restarter.showToast(foreground, text);
            } else if (Log.isLoggable("InstantRun", 2)) {
              Log.v("InstantRun", "Couldn't show toast (no activity) : " + text);
            }
            break;
          }
        }
        if (Log.isLoggable("InstantRun", 6)) {
          Log.e("InstantRun", "Unexpected message type: " + message);
        }
      }
    }
```


##### handlePatches

```java
  private int handlePatches(List<ApplicationPatch> changes, boolean hasResources, int updateMode)
  {
    if (hasResources) {
      FileManager.startUpdate();
    }
    //根据文件后缀名进行patch的选择
    for (ApplicationPatch change : changes)
    {
      String path = change.getPath();
      if (path.endsWith(".dex"))
      {
        handleColdSwapPatch(change);
        
        boolean canHotSwap = false;
        for (ApplicationPatch c : changes) {
          if (c.getPath().equals("classes.dex.3"))
          {
            canHotSwap = true;
            break;
          }
        }
        if (!canHotSwap) {
          updateMode = 3;
        }
      }
      else if (path.equals("classes.dex.3"))
      {
        updateMode = handleHotSwapPatch(updateMode, change);
      }
      else if (isResourcePath(path))
      {
        updateMode = handleResourcePatch(updateMode, change, path);
      }
    }
    if (hasResources) {
      FileManager.finishUpdate(true);
    }
    return updateMode;
  }
  ```

##### hotPatches


是通过dexPath创建一个ClassLoader，并且通过它去创建一个AppPatchesLoaderImpl，然后执行AppPatchesLoaderImpl的load方法。AppPatchesLoaderImpl这个类大家还记得吧，就是之前的那个［记录类］。

```java
 try
    {
      String dexFile = FileManager.writeTempDexFile(patch.getBytes());
      if (dexFile == null)
      {
        Log.e("InstantRun", "No file to write the code to");
        return updateMode;
      }
      if (Log.isLoggable("InstantRun", 2)) {
        Log.v("InstantRun", "Reading live code from " + dexFile);
      }
      String nativeLibraryPath = FileManager.getNativeLibraryFolder().getPath();
      //这里的dexFile是patch的dex..AS给应用的,包含$override文件
      DexClassLoader dexClassLoader = new DexClassLoader(dexFile, this.mApplication.getCacheDir().getPath(), nativeLibraryPath, getClass().getClassLoader());
      ////加载AppPatchesLoaderImpl类，初始化，执行load方法,注意这里新创建了一个classLoader区加载这个类
      Class<?> aClass = Class.forName("com.android.tools.fd.runtime.AppPatchesLoaderImpl", true, dexClassLoader);
      try
      {
        if (Log.isLoggable("InstantRun", 2)) {
          Log.v("InstantRun", "Got the patcher class " + aClass);
        }
        PatchesLoader loader = (PatchesLoader)aClass.newInstance();
        if (Log.isLoggable("InstantRun", 2)) {
          Log.v("InstantRun", "Got the patcher instance " + loader);
        }
        String[] getPatchedClasses = (String[])aClass.getDeclaredMethod("getPatchedClasses", new Class[0]).invoke(loader, new Object[0]);
        if (Log.isLoggable("InstantRun", 2))
        {
          Log.v("InstantRun", "Got the list of classes ");
          for (String getPatchedClass : getPatchedClasses) {
            Log.v("InstantRun", "class " + getPatchedClass);
          }
        }
        if (!loader.load()) {
          updateMode = 3;
        }
      }
      catch (Exception e)
      {
        Log.e("InstantRun", "Couldn't apply code changes", e);
        e.printStackTrace();
        updateMode = 3;
      }
    }
```
    
>Instant run热更新是多ClassLoader加载方案，每个插件dex都有一个ClassLoader，如果插件需要升级，直接重新创建一个自定的ClassLoader加载新的插件。但是目前来看，Instant run修改java代码大部分情况下都是重启更新机制，可能热更新机制还有bug。资源更新是热更新，重启对应Activity就可以。



![](/img/2017-05-02-multidex/14935318889556.jpg)

##### ColdPatches

```java
private static void handleColdSwapPatch(ApplicationPatch patch){
    if (patch.path.startsWith("slice-")){
      File file = FileManager.writeDexShard(patch.getBytes(), patch.path);
      if (Log.isLoggable("InstantRun", 2)) {
        Log.v("InstantRun", "Received dex shard " + file);
      }
    }
  }
  public static File writeDexShard(byte[] bytes, String name)
  {
    //可以看到就是写到之前解压instant-run.zip的文件夹中去
    File dexFolder = getDexFileFolder(getDataFolder(), true);
    if (dexFolder == null) {
      return null;
    }
    File file = new File(dexFolder, name);
    writeRawBytes(file, bytes);
    return file;
  }
```

把dex文件写到私有目录，等待整个app重启，重启之后，使用前面提到的IncrementalClassLoader加载dex即可。

##### 资源patch

直接写资源文件吧？？

```java
  private static int handleResourcePatch(int updateMode, ApplicationPatch patch, String path)
  {
    if (Log.isLoggable("InstantRun", 2)) {
      Log.v("InstantRun", "Received resource changes (" + path + ")");
    }
    FileManager.writeAaptResources(path, patch.getBytes());
    
    updateMode = Math.max(updateMode, 2);
    return updateMode;
  }
  
```

#### AbstractPatchesLoaderImpl

##### load

dexFile中会包含修改过得patch,即加了$override的类,这里使用了多classLoader体系因为每个插件都创建了一个classLoader

```java
public boolean load(){
    try{
    //getPatchedClasses前面提到过,列出了改变的class
      for (String className : getPatchedClasses())
      {
        //初始化带override的类
        ClassLoader cl = getClass().getClassLoader();
        Class<?> aClass = cl.loadClass(className + "$override");
        Object o = aClass.newInstance();
        Class<?> originalClass = cl.loadClass(className);
        Field changeField = originalClass.getDeclaredField("$change");
        
        changeField.setAccessible(true);
        
        Object previous = changeField.get(null);
        if (previous != null)
        {
          Field isObsolete = previous.getClass().getDeclaredField("$obsolete");
          if (isObsolete != null) {
            isObsolete.set(null, Boolean.valueOf(true));
          }
        }
        //反射调用,使得 $change = ..$override
        changeField.set(null, o);
        if ((Log.logging != null) && (Log.logging.isLoggable(Level.FINE))) {
          Log.logging.log(Level.FINE, String.format("patched %s", new Object[] { className }));
        }
      }
    }
    catch (Exception e)
    {
      if (Log.logging != null) {
        Log.logging.log(Level.SEVERE, String.format("Exception while patching %s", new Object[] { "foo.bar" }), e);
      }
      return false;
    }
    return true;
  }
```

#### 编译

[总结的很好](http://blog.csdn.net/sgwhp/article/details/52423313)

##### 首次编译 

* 将`instant-run.jar`编译成`dex`，打入`apk`。
* 修改`manifest`文件，将a`pplication`替换为`instant-run.jar`中的`BootstrapApplication`。
* app的所有类都添加类型为`IncrementalChange`的change字段，所有方法前面都添加一段代码，如果change字段不为空则调用change的`accessdispatch`方法并返回。编译出来的dex都将打包到`instant-run.zip`并打入apk。

##### 非首次编译 

* `hot swap` 
根据修改的类生成一个新的实现`IncrementalChange`接口的新类，类名称为原名称加上`$override`，里面的方法大部分来自修改过后的类。以及生成对应的`AppPatchesLoaderImpl`，记录所有修改的类原名称。
* `warm swap` 
生成新的`resources.arsc`
* `cold swap` 
重新生成修改类对应的dex

##### someThing

由于首次编译时`application`已被替换成`BootstrapApplication`，启动的`application`自然也是它，重点:

>调用setupClassLoaders将PathClassLoader的parent设置为IcreamentalClassLoader，而IcreamentalClassLoader的parent则设置为PathClassLoader的原parent即BootClassLoader，并且通过DelegateClassLoader代理完成findClass工作。相当于在PathClassLoader和BootClassLoader之间插入了一个DelegateClassLoader。此时instant-run.zip里面所有的dex以及后续修改代码生成的dex都已经在data/data的cache目录下，DelegateClassLoader会去这个目录进行class的加载。 
 


![](/img/2017-05-02-multidex/14935282972784.jpg)



### todo

* dx :-multi-dex原理
* instant-run之资源补丁


