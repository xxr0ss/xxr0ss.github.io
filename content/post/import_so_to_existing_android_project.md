---
title: "Import_so_to_existing_android_project"
date: 2022-03-12T15:56:44+08:00
ShowToc: true
---

---
title: "引入so文件到现有Android项目"
date: 2022-03-12T15:44:50+08:00
ShowToc: true
---

这里介绍一种如下情况下，将so导入到Android项目的方法：

- 仅提供了JNI接口的so
- 一个常规的Android项目，在Android Studio上，使用Gradle进行构建。



为了方便演示，我们直接在Android Studio新建一个Native工程叫testso，这个工程只是用来方便我们创建so的，编译出来的apk我们不需要，我们的目的是将apk中的`libtestso.so`提取出来，并假设我们只有so，将其导入到一个Android项目中。

## 提供so

### Native代码

编写了一个究极简单的函数。

```C++
#include <jni.h>
#include <string>

extern "C"
JNIEXPORT jint JNICALL
Java_com_xxr0ss_TestSo_addJNI(JNIEnv *env, jobject thiz, jint a, jint b) {
    return a + b;
}
```




### 对应的kotlin代码

其实我是先创建的这个，然后Android Studio能提示addJNI不存在，方便地自动在`cpp/native-lib.cpp`里生成函数，避免我们写错JNI函数声明。

```Kotlin
package com.xxr0ss;

class TestSo {

    external fun addJNI(a: Int, b: Int): Int

    companion object {
        init {
            System.loadLibrary("testso")
        }
    }
}
```




### 准备完毕


```text
lib
 ├── arm64-v8a
 │   └── libtestso.so
 ├── armeabi-v7a
 │   └── libtestso.so
 ├── x86
 │   └── libtestso.so
 └── x86_64
     └── libtestso.so
```



## 将so导入Android项目

### 为外部so创建对应的类

现在假设，我们拿到的只有几个不同架构下的so，没有其他任何文件，现在需要导入到我们的项目里。任挑一个架构的so，看一下有哪些导出函数：

```Bash
$ nm -D ./arm64-v8a/libtestso.so
00000000000005e8 T Java_com_xxr0ss_TestSo_addJNI
0000000000002000 A __bss_end__
0000000000002000 A __bss_start
0000000000002000 A __bss_start__
                 U __cxa_atexit
                 U __cxa_finalize
0000000000002000 A __end__
                 U __register_atfork
0000000000002000 A _bss_end__
0000000000002000 A _edata
0000000000002000 A _end
```




根据名称`Java_com_xxr0ss_TestSo_addJNI`推断出我们需要在自己的项目中新建包`com.xxr0ss`，并且编写`TestSo.kt`（java也一样）



```Kotlin
// TestSo.kt
package com.xxr0ss

class TestSo {
    external fun addJNI(a: Int, b: Int): Int
    
    companion object {
        init {
            System.loadLibrary("testso") // 注意名称
        }
    }
}
```


值得**注意**的是，`System.loadLibrary("testso")`里面这个library名字是不带前面lib的




```Kotlin
// MainActivity.kt
package com.example.use_3rd_party_so

import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.widget.TextView
import com.xxr0ss.TestSo

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        findViewById<TextView>(R.id.text).text = "1 + 2 = ${TestSo().addJNI(1, 2)}"
    }
}
```




### 将so放入项目中

创建目录`app/src/main/jniLibs`，放入so，现在的目录结构（有省略）是：

```text
main
 ├── AndroidManifest.xml
 ├── java
 │   └── com
 │       ├── example
 │       │   └── use_3rd_party_so
 │       └── xxr0ss
 │           └── TestSo.kt
 ├── jniLibs
 │   ├── arm64-v8a
 │   │   └── libtestso.so
 │   ├── armeabi-v7a
 │   │   └── libtestso.so
 │   ├── x86
 │   │   └── libtestso.so
 │   └── x86_64
 │       └── libtestso.so
 └── res
```




### 修改app的build.gradle
在`android`闭包下添加`sourceSets`

```gradle
android {

    // ...

    sourceSets {
        main {
            jniLibs.srcDirs = ['src/main/jniLibs']
        }
    }
}
```


这样使得build apk时，会把jniLibs目录里的so，放到apk根目录的`lib/`下


## 补充说明

假如我们只有一个单独的so文件，那我们在引入当前项目时，也要注意**放在对应架构名的目录下**。可以通过readelf等方法进行判断。