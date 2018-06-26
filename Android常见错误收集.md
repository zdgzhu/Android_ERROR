

#               Android常见操作和错误收集

### 一、背景 

在Android开发过程，会遇到各种各样的错误，再次记录下来。

### 二、常见错误

#### 错误1

1.1  android studio 3.0.1新建项目时，出现以下错误；（2018.6.15）

```
Error:Execution failed for task ':app:preDebugAndroidTestBuild'.
```

   ![](D:\AndroidFile\Photo\Android常见错误收集\error01_01.png) 

1.2  解决方法：在app下的build.gradle文件中的dependences {}中添加如下代码： 

```
androidTestImplementation('com.android.support:support-annotations:26.1.0') {
    force = true
}
```

添加后dependences中结构类似 

```
dependencies {
    androidTestImplementation('com.android.support:support-annotations:26.1.0') {
        force = true
    }
    ...
    }
```

#### 错误2  一直停留在build.gradle(下载状态)，添加阿里云镜像

2.1  **com.android.support:appcompat-v7:26.0.0以上无法下载的问题**

2.2 解决方法：在项目的根目录的build.gradle文件中，添加阿里云镜像地址(2018.6.15)

```

allprojects {
    repositories {
        maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
        google()
        jcenter()
    }
}


allprojects {
    repositories {
        maven{ //如果下载还是比较慢，就考虑在该目录中也添加阿里云镜像
            url 'http://maven.aliyun.com/nexus/content/groups/public/'
        }
        google()
//        jcenter()
    }
}
```



###错误3 AndroidManifest.xml 覆盖(合并)问题

   3.1  出现问题时显示的代码(2018.6.15)

```
Error:java.lang.RuntimeException: Manifest merger failed : Attribute meta-data#android.support.VERSION@value value=(26.0.0) from [com.android.support:design:26.0.0] AndroidManifest.xml:28:13-35
```

​       ![](D:\AndroidFile\Photo\Android常见错误收集\error03_01.JPG)

   或者下面的代码

```
Error:Execution failed for task ':app:processDebugManifest'.
> Manifest merger failed : Attribute meta-data#android.support.VERSION@value value=(26.0.0) from [com.android.support:design:26.0.0] AndroidManifest.xml:28:13-35
  	is also present at [com.android.support:appcompat-v7:26.1.0] AndroidManifest.xml:28:13-35 value=(26.1.0).
  	Suggestion: add 'tools:replace="android:value"' to <meta-data> element at AndroidManifest.xml:26:9-28:38 to override.
```

![](D:\AndroidFile\Photo\Android常见错误收集\error03_02.JPG)

3.2  解决方法：在Manifest.xml的根节点，添加下面的代码

```
xmlns:tools="http://schemas.android.com/tools"
//大概是这样子
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.winanainc"
    android:versionCode="3"  //这行可以不要
    android:versionName="1.2" //这行可以不要
    xmlns:tools="http://schemas.android.com/tools">
```

还需要添加下面的内容

```
<application>
   ...
   ..
    <meta-data
        tools:replace="android:value"
        android:name="android.support.VERSION"
        android:value="26.0.0" />
</application>
```

### 错误4 AndroidManifest.xml合并冲突(sdk)

4.1 出现问题时的代码 (2018.6.16)

```

Manifest merger failed : uses-sdk:minSdkVersion 15 cannot be smaller than version 16 declared in library [cn.sharesdk:ShareSDK:3.2.0] C:\Users\Administrator\.gradle\caches\transforms-1\files-1.1\ShareSDK-3.2.0.aar\43ec3adb895d42769f73b236e54ee4be\AndroidManifest.xml as the library might be using APIs not available in 15
	Suggestion: use a compatible library with a minSdk of at most 15,
		or increase this project's minSdk version to at least 16,
		or use tools:overrideLibrary="cn.sharesdk" to force usage (may lead to runtime failures)
Manifest merger failed : uses-sdk:minSdkVersion 15 cannot be smaller than version 16 declared in library [cn.sharesdk:ShareSDK:3.2.0] C:\Users\Administrator\.gradle\caches\transforms-1\files-1.1\ShareSDK-3.2.0.aar\43ec3adb895d42769f73b236e54ee4be\AndroidManifest.xml as the library might be using APIs not available in 15
	Suggestion: use a compatible library with a minSdk of at most 15,
		or increase this project's minSdk version to at least 16,
		or use tools:overrideLibrary="cn.sharesdk" to force usage (may lead to runtime failures)
```

![](D:\AndroidFile\Photo\Android常见错误收集\error04_01.PNG)

4.2 解决方法 在Manifest.xml的根节点，添加下面的代码

```
 <uses-sdk tools:overrideLibrary="cn.sharesdk,cn.sharesdk.onekeyshare" />
 //大概是这样子
 <manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.demotest">
    //添加的代码
    <uses-sdk tools:overrideLibrary="cn.sharesdk,cn.sharesdk.onekeyshare" />
    <application
        tools:replace="android:name"
        ...
        />
</manifest>
```

4.3 出错原因

​       出现这个错误的原因是我引入的第三方库最低支持版本高于我的项目的最低支持版本，异常中的信息显示：我的项目的最低支持版本为15（Android 4.0.3），而第三方库的最低支持版本为16（Android 4.1），所以抛出了这个异常 .

​       在AndroidManifest.xml文件中 标签中添加<uses-sdktools:overrideLibrary="xxx.xxx.xxx"/>，其中的xxx.xxx.xxx为第三方库包名，如果存在多个库有此异常，则用逗号分割它们，例如：<uses-sdk tools:overrideLibrary="xxx.xxx.aaa, xxx.xxx.bbb"/>，这样做是为了项目中的AndroidManifest.xml和第三方库的AndroidManifest.xml合并时可以忽略最低版本限制。 



### 三 、常用操作

#### 1、查看gradle的依赖树的命令

​     -1.1  在Terminal 中输入 以下命令(2018.6.15)

```
// 查看 compile 时的依赖关系 （compile 或者是 implementation）
gradlew :app:dependencies --configuration compile   
```

  ![](D:\AndroidFile\Photo\Android常见错误收集\operat01_01.JPG)

![](D:\AndroidFile\Photo\Android常见错误收集\operat01_02.JPG)

![](D:\AndroidFile\Photo\Android常见错误收集\operat01_03.JPG)





​    -1.2  [Gradle解决依赖冲突](https://www.jianshu.com/p/8d02da77c83d)

####2、