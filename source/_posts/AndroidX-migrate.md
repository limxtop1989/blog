---
title: 穿山甲集成报错
date: 2023-08-06 23:10:48
tags: 穿山甲, 集成报错，AndroidX migrate，FileProvider not found
categories:
- [Android]
---
# 异常描述
引入穿山甲SDK后，编译运行时异常，堆栈如下
```java 
    2023-08-06 23:08:51.659 32243-32243/com.maxim.wordcard.debug E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.maxim.wordcard.debug, PID: 32243
    java.lang.RuntimeException: Unable to get provider com.bytedance.sdk.openadsdk.TTFileProvider: java.lang.ClassNotFoundException: Didn't find class "com.bytedance.sdk.openadsdk.TTFileProvider" on path: DexPathList[[zip file "/data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/base.apk"],nativeLibraryDirectories=[/data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/lib/arm64, /data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/base.apk!/lib/arm64-v8a, /system/lib64, /system_ext/lib64]]
        at android.app.ActivityThread.installProvider(ActivityThread.java:7479)
        at android.app.ActivityThread.installContentProviders(ActivityThread.java:6985)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6756)
        at android.app.ActivityThread.-$$Nest$mhandleBindApplication(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2129)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loopOnce(Looper.java:201)
        at android.os.Looper.loop(Looper.java:288)
        at android.app.ActivityThread.main(ActivityThread.java:7884)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:936)
     Caused by: java.lang.ClassNotFoundException: Didn't find class "com.bytedance.sdk.openadsdk.TTFileProvider" on path: DexPathList[[zip file "/data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/base.apk"],nativeLibraryDirectories=[/data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/lib/arm64, /data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/base.apk!/lib/arm64-v8a, /system/lib64, /system_ext/lib64]]
        at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:259)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:379)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
        at android.app.AppComponentFactory.instantiateProvider(AppComponentFactory.java:147)
        at androidx.core.app.CoreComponentFactory.instantiateProvider(CoreComponentFactory.java:67)
        at android.app.ActivityThread.installProvider(ActivityThread.java:7463)
        at android.app.ActivityThread.installContentProviders(ActivityThread.java:6985) 
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6756) 
        at android.app.ActivityThread.-$$Nest$mhandleBindApplication(Unknown Source:0) 
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2129) 
        at android.os.Handler.dispatchMessage(Handler.java:106) 
        at android.os.Looper.loopOnce(Looper.java:201) 
        at android.os.Looper.loop(Looper.java:288) 
        at android.app.ActivityThread.main(ActivityThread.java:7884) 
        at java.lang.reflect.Method.invoke(Native Method) 
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548) 
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:936) 
    	Suppressed: java.lang.NoClassDefFoundError: Failed resolution of: Landroid/support/v4/content/FileProvider;
        at java.lang.VMClassLoader.findLoadedClass(Native Method)
        at java.lang.ClassLoader.findLoadedClass(ClassLoader.java:738)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:363)
        		... 15 more
     Caused by: java.lang.ClassNotFoundException: Didn't find class "android.support.v4.content.FileProvider" on path: DexPathList[[zip file "/data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/base.apk"],nativeLibraryDirectories=[/data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/lib/arm64, /data/app/~~8aOWO0BH2NR-PT5SN6_9Sw==/com.maxim.wordcard.debug-z4MN-3uI1JjNQYgIxbhNYQ==/base.apk!/lib/arm64-v8a, /system/lib64, /system_ext/lib64]]
        at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:259)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:379)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
```        

app目录下的 build.gradle 配置如下：
```gradle
dependencies {

    implementation fileTree(dir: 'libs', include: ['*.jar'])
    // Ad begin
    implementation 'com.pangle.cn:ads-sdk-pro:5.4.1.6'
    // Ad end
    implementation 'com.github.PhilJay:MPAndroidChart:v3.1.0'

    implementation  'com.umeng.umsdk:common:9.4.7'// required
    implementation  'com.umeng.umsdk:asms:1.4.0'// required
    implementation 'com.umeng.umsdk:apm:1.7.0' // crash and performance
    implementation  'com.umeng.umsdk:abtest:1.0.0'// optional

    implementation 'com.github.deano2390:MaterialShowcaseView:1.3.7'
    // Navigation start
    def nav_version = "2.5.3"
    // Kotlin
    implementation ("androidx.navigation:navigation-fragment-ktx:$nav_version")
    implementation ("androidx.navigation:navigation-ui-ktx:$nav_version")
    // Feature module Support
    implementation "androidx.navigation:navigation-dynamic-features-fragment:$nav_version"
    // Navigation end

    implementation 'androidx.core:core-ktx:1.7.0'
    implementation 'androidx.appcompat:appcompat:1.6.0'
    implementation 'com.google.android.material:material:1.8.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.5.1'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.5.1'
    implementation "androidx.activity:activity-ktx:1.7.1"
    implementation "androidx.fragment:fragment-ktx:1.5.7"

    api "io.reactivex.rxjava2:rxandroid:2.1.0"
    implementation 'androidx.lifecycle:lifecycle-livedata-core-ktx:2.5.1'
    implementation 'androidx.legacy:legacy-support-v4:1.0.0'
    // room start
    def room_version = "2.5.1"
    implementation("androidx.room:room-runtime:$room_version")
    annotationProcessor("androidx.room:room-compiler:$room_version")
    // To use Kotlin annotation processing tool (kapt)
    kapt("androidx.room:room-compiler:$room_version")
    // To use Kotlin Symbol Processing (KSP)
//    ksp("androidx.room:room-compiler:$room_version")
    // optional - Kotlin Extensions and Coroutines support for Room
    implementation("androidx.room:room-ktx:$room_version")
    // optional - RxJava2 support for Room
    implementation("androidx.room:room-rxjava2:$room_version")
    // optional - RxJava3 support for Room
    implementation("androidx.room:room-rxjava3:$room_version")
    // optional - Guava support for Room, including Optional and ListenableFuture
    implementation("androidx.room:room-guava:$room_version")
    // optional - Test helpers
    testImplementation("androidx.room:room-testing:$room_version")
    // optional - Paging 3 Integration
    implementation("androidx.room:room-paging:$room_version")
    // room end

    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
}
```    

# 异常分析
打开 `TTFileProvider`， 它继承的是 `android.support.v4.content.FileProvider`
```java 
package com.bytedance.sdk.openadsdk;

import android.support.v4.content.FileProvider;

public class TTFileProvider extends FileProvider {
    public TTFileProvider() {
    }
}
```

如果添加包含 `android.support.v4.content.FileProvider` 的依赖，`implementation 'com.android.support:support-v4:26.1.0'` 则编译报出好几个类重复定义的异常。

```java
Duplicate class android.support.v4.os.ResultReceiver found in modules core-1.9.0-runtime (androidx.core:core:1.9.0) and support-compat-26.1.0-runtime (com.android.support:support-compat:26.1.0)
Duplicate class android.support.v4.os.ResultReceiver$1 found in modules core-1.9.0-runtime (androidx.core:core:1.9.0) and support-compat-26.1.0-runtime (com.android.support:support-compat:26.1.0)
Duplicate class android.support.v4.os.ResultReceiver$MyResultReceiver found in modules core-1.9.0-runtime (androidx.core:core:1.9.0) and support-compat-26.1.0-runtime (com.android.support:support-compat:26.1.0)
Duplicate class android.support.v4.os.ResultReceiver$MyRunnable found in modules core-1.9.0-runtime (androidx.core:core:1.9.0) and support-compat-26.1.0-runtime (com.android.support:support-compat:26.1.0)
```

异常日志里有一段警告
>Your project has set `android.useAndroidX=true`, but configuration `:app:googleDebugRuntimeClasspath` still contains legacy support libraries, which may cause runtime issues.
This behavior will not be allowed in Android Gradle plugin 8.0.
Please use only AndroidX dependencies or set `android.enableJetifier=true` in the `gradle.properties` file to migrate your project to AndroidX (see https://developer.android.com/jetpack/androidx/migrate for more info).

我原来的  `gradle.properties` 只有 `android.useAndroidX=true` 的配置，按照警告的建议，修改后如下：
```
android.useAndroidX=true
android.enableJetifier=true
```

# [Migrate an existing project using Android Studio](https://developer.android.google.cn/jetpack/androidx/migrate#migrate_an_existing_project_using_android_studio)
With Android Studio 3.2 and higher, you can migrate an existing project to AndroidX by selecting Refactor > Migrate to AndroidX from the menu bar.

The refactor command makes use of two flags. By default, both of them are set to `true` in your `gradle.properties` file:

`android.useAndroidX=true`
    The Android plugin uses the appropriate AndroidX library instead of a Support Library.
`android.enableJetifier=true`
    The Android plugin automatically migrates existing third-party libraries to use AndroidX by rewriting their binaries.

看这描述，开启 `android.enableJetifier=true` 后，编译时将重写二进制文件，将 `android.support.v4.content.FileProvider` 改为 AndroidX 对应的类 `androidx.core.content.FileProvider` 相当于将穿山甲库里引用的support的类，重写为AndroiX的，如下

```java 
package com.bytedance.sdk.openadsdk;

import androidx.core.content.FileProvider;

public class TTFileProvider extends FileProvider {
    public TTFileProvider() {
    }
}
```

至此问题原因找到并解决，移植 AndroidX 不彻底，所致，把问题暴露出来。