#               Android常见错误收集

### 一、背景 

在Android开发过程，会遇到各种各样的错误，再次记录下来。

### 二、常见错误

#### 错误1

1.1  android studio 3.0.1新建项目时，出现以下错误；（2018.6.15）

```
Error:Execution failed for task ':app:preDebugAndroidTestBuild'.
```

![](C:\Users\dell\Desktop\a.png)

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

#### 错误2







