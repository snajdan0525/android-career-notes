# Storing your secure information in the NDK #
　　在安卓应用中,可以非常容易的实现逆向工程做到查看篡改隐私数据。我们有很多办法去组织"黑客"去查看篡改你的应用程序,但是对于有毅力的黑客来说,他总能有办法成功。
　　下面是一个*apk包中的一段代码：
```java
public class MainActivity extends AppCompatActivity {
private static final String TAG = "MainActivity";
private static final String AUTH_TOKEN = "V293ISBob3cgY3VyaW91cyBlaD8=";

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    // Some super API call using that key
    Log.i(TAG, "key: " + AUTH_TOKEN);
}
}
```
　　很多次程序员都是通过这种方式去存放authentication keys的。但是这种方法会带来巨大的安全隐患，对于一个恶意用户来说，这个authentication keys不到10秒就能破解拿到。
![](http://i.imgur.com/xbSs43I.gif)
　　java源代码很容易就能被反编译,然而C++是不能被反编译的但可以被反汇编,而且是个不轻松的活。我们可以利用这个特性,并结合ndk把authentication keys等隐私数据写入c++ code实现隐私数据的保护。

**编码开始:)**
----------
　　为了在Android Studio上方便工作.我们首选把编辑界面从Android切换到Project.
![](http://i.imgur.com/qeENyJF.png)
　　在app/src/main目录下新建一个cpp的目录,在cpp目录下新建一个名为Native-libcpp的c/c++ source file.现在我们的这个project需要用我们编写的c++代码使用和构建我们的库,在app/src目录下新建一个CMakeLists.text的CMake 构建脚本文件,这个文件描述了我们将构建的什么以及被构建的代码在哪里。

```CMake
# Sets the minimum version of CMake required to build your native library.
# This ensures that a certain set of CMake features is available to
# your build.

cmake_minimum_required(VERSION 3.4.1)

# Specifies a library name, specifies whether the library is STATIC or
# SHARED, and provides relative paths to the source code. You can
# define multiple libraries by adding multiple add.library() commands,
# and CMake builds them for you. When you build your app, Gradle
# automatically packages shared libraries with your APK.

add_library( # Specifies the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )
```
　　在app/src/main/cpp/native_lib.cpp文件上右击选择Link C++ Project with Gradle.Gradle将会自动同步,同步结束后打开MainActivity或者任何你准备使用这个库的地方，在类中添加如下代码：
```java
static {
    System.loadLibrary("native-lib");
}
private native String invokeNativeFunction();
```
　　此时还不能直接使用invokeNativeFunction JNI函数，需要首先创建这个JNI函数

```c
#include <jni.h>

extern "C" {
    JNIEXPORT jstring JNICALL
    Java_info_androidsecurity_helloworld_MainActivity_invokeNativeFunction(JNIEnv *env, jobject instance) {
        return env->NewStringUTF("V293ISBob3cgY3VyaW91cyBlaD8=");
    }
}
```
　　添加extern关键字是为了让这个JNI函数对JVM虚拟机可见,此时可以通过这个JNI函数返回需要被保护的AUTH_TOKEN了,在Java代码中调用JNI函数非常的简单,如下：
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Context mContext = getApplicationContext();

    // Some super API call using that key
    Log.i(TAG, "key: " + invokeNativeFunction());
}
```
　　现在即使恶意用户反编译了apk文件,它也看不到AUTH_TOKEN,唯一的办法就是反汇编这个library并且在16进制编辑器内打开它，但是依然是非常难以找到这个字符串的。
![](http://i.imgur.com/goaq2Gc.png)

## 总结 ##
　　上述方法并不是一个完全安全的方法，采用这个方法，恶意用户依然是可以通过一定手段获取到AUTH_TOKEN的。通过使用NDK存放隐私数据，只是让获取因数信息的难度加大了。
　　这个方法一般会结合一定的加密编码方式，这样会使得破解的难度大大增加，通常情况下我们也会把AUTH_TOKEN这种需要被保护的隐私数据转换成16进制，这样即使在16进制编辑器中打开了，也分辨不清。