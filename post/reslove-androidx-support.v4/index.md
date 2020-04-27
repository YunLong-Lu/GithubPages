>最近两天接手了一个同事上传的Android工程，需要接入一个第三方的SDK，接入完成开始打包报了错,检查后发现是同事升级了AndroidX,而我接入的第三方SDK引用了android.support.v4。瞬间有点蒙，还查了半天资料如何解决，突然想起大概半年前碰到过这个问题，本着再一再二不再三的原则，这次要记录在案。
<!--more-->
接入SDK后打包发生问题如下：
```
Process: packagename, PID: 22181
    java.lang.NoClassDefFoundError: Failed resolution of: Landroid/support/v4/app/FragmentManager$FragmentLifecycleCallbacks;
        at android.arch.lifecycle.LifecycleDispatcher.init(LifecycleDispatcher.java:58)
        at android.arch.lifecycle.ProcessLifecycleOwnerInitializer.onCreate(ProcessLifecycleOwnerInitializer.java:35)
        at android.content.ContentProvider.attachInfo(ContentProvider.java:2101)
        at android.content.ContentProvider.attachInfo(ContentProvider.java:2075)
        at android.app.ActivityThread.installProvider(ActivityThread.java:7147)
        at android.app.ActivityThread.installContentProviders(ActivityThread.java:6630)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6525)
        at android.app.ActivityThread.access$1400(ActivityThread.java:220)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1883)
        at android.os.Handler.dispatchMessage(Handler.java:107)
        at android.os.Looper.loop(Looper.java:224)
        at android.app.ActivityThread.main(ActivityThread.java:7520)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:539)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)
     Caused by: java.lang.ClassNotFoundException: Didn't find class "android.support.v4.app.FragmentManager$FragmentLifecycleCallbacks" on path: DexPathList[[zip file "/data/app/packagename-ir6wH0r_cvF-StUhV3NDeg==/base.apk"],nativeLibraryDirectories=[/data/app/packagename-ir6wH0r_cvF-StUhV3NDeg==/lib/arm64, /data/app/packagename-ir6wH0r_cvF-StUhV3NDeg==/base.apk!/lib/arm64-v8a, /system/lib64, /system/product/lib64]]
        at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:230)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:379)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
        at android.arch.lifecycle.LifecycleDispatcher.init(LifecycleDispatcher.java:58) 
        at android.arch.lifecycle.ProcessLifecycleOwnerInitializer.onCreate(ProcessLifecycleOwnerInitializer.java:35) 
        at android.content.ContentProvider.attachInfo(ContentProvider.java:2101) 
        at android.content.ContentProvider.attachInfo(ContentProvider.java:2075) 
        at android.app.ActivityThread.installProvider(ActivityThread.java:7147) 
        at android.app.ActivityThread.installContentProviders(ActivityThread.java:6630) 
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6525) 
        at android.app.ActivityThread.access$1400(ActivityThread.java:220) 
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1883) 
        at android.os.Handler.dispatchMessage(Handler.java:107) 
        at android.os.Looper.loop(Looper.java:224) 
        at android.app.ActivityThread.main(ActivityThread.java:7520) 
        at java.lang.reflect.Method.invoke(Native Method) 
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:539) 
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950) 
2020-04-26 21:53:02.095 22181-22181/packagename E/MQSEventManagerDelegate: failed to get MQSService.
```

从上面的报错信息可以看出包里面是缺少了support.v4的依赖，所以此时的解决办法就是添加依赖包，在App下的build里面引入

`
  dependencies {
      implementation 'com.android.support:support-v4:27.1.1'
  }
`

这次引入之后还没来得及打包就直接“红色”报错，在看一下代码发现上面引入了androidx，冲突了，这也是我心慌的点，我只记得之前的一次报错我是直接将v4包删掉了，但这次不一样，第三方库必须要使用v4。我去查了一下之前的修改方法，结果发现确实是自己忘记了上次的修改方法。

当时的修改记录里面还有：修改当前项目的 gradle.properties 添加下面两句代码


`
android.useAndroidX=true
`

`
android.enableJetifier=true
`


 > android.useAndroidX=true 表示当前项目启用 AndroidX
  android.enableJetifier=true 表示将依赖包也迁移到AndroidX 。
  如果取值为 false ,表示不迁移依赖包到AndroidX，但在使用依赖包中的内容时可能会出现问题，当然了，如果你的项目中没有使用任何三方依赖，那么，此项可以设置为 false。
  (ps：这里是当时修改的时候的注释内容，但是非常惭愧的是我想不起是哪为大神的文章给出的指导，还是应该给他说声谢谢，半年之后再次为我排忧解难)

这样修改之后打包，然后又冲突了


`
Program type already present: androidx.lifecycle.ClassesInfoCache$CallbackInfo
`

这种问题就不慌了，直接排除掉这个引用就可以了，同样是App内的build，我是两个冲突，全部排除，问题解决


`
configurations.all {
    exclude group: 'androidx.lifecycle'
    exclude group: 'androidx.arch.core'
}
`

总结一下就是问题及时记录，这次的修改算不上是什么大的问题，也算不上是一次像样的技术分享，更多的是提醒自己吧。想想之前解决这次问题的方法也许更值得分享，最初的缺少v4依赖，到最终解决，碰到的每个问题代表什么含义，为什么会出现这个问题，然后找到解决方法（what、why、how）三步，是我这个刚刚接触到Android开发的菜鸟最应该记住的东西。