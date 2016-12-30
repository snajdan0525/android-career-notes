# 提高Android应用程序的安全性的策略 #
## 1.Android中使用NDK安全的保存敏感数据 ##
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

## 2.网络通信中的安全策略 ##
　　Android应用程序经常通过网络通信与Server交换数据或者token，因此防止数据被窃取至关重要。
　　为了实现安全的网络通信,我们必须要采用HTTPS,但是这是一个必要条件，即使使用HTTPS是远远不够的，它并不能保证数据通信的安全.
> 这篇文章就介绍了一种情况：
> [https://drakeet.me/android-security-guide](https://drakeet.me/android-security-guide)


　　在网络攻击中，最常见的就是MITM(Man-In-The-Middle),它包括主动攻击和被动攻击两种方式。为了防止你的android应用程序遭受被动MITM攻击，可以使用Diffie-Hellman密钥交换算法。面对主动攻击时，我们就需要SSL了，在Android中Retrofit和OKHttp支持HTTPS和SSL.
## 3.使用Cipher加密算法类 ##
　　java7给Android提供了很多数据加密算法，其中Cipher就是最常用的一个，Cipher类中提供了AES和RSA算法，再例如安全随机数生成器SecureRandom给KeyGenerator提供了更加可靠的初始化参数，避免离线攻击等等。我们可以在存储数据时，用过提供的Cipher对数据进行加密。下面是一个使用Cipher的例子：
```java
private static byte[] encrypt(byte[] key, byte[] input) throws Exception {
    SecretKeySpec skeySpec = new SecretKeySpec(key, "AES");
    Cipher cipher = Cipher.getInstance("AES");
    cipher.init(Cipher.ENCRYPT_MODE, skeySpec);
    byte[] encrypted = cipher.doFinal(input);
    return encrypted;
}

private static byte[] decrypt(byte[] key, byte[] encrypted) throws Exception {
    SecretKeySpec skeySpec = new SecretKeySpec(key, "AES");
    Cipher cipher = Cipher.getInstance("AES");
    cipher.init(Cipher.DECRYPT_MODE, skeySpec);
    byte[] decrypted = cipher.doFinal(encrypted);
    return decrypted;
}
```




## 4.提高SharedPerference的安全性 ##
　　SharedPerference是我们经常使用的一个存储数据的方式，乱七八糟废话就不说了，什么MODE_PRIVATE可以防止其他应用访问你的数据的就不多说了，非ROOT不能访问也不说了。这个东西绝对不能用明文存放，而且是一个很不安全的东西。下面介绍一个[github](https://github.com/scottyab/secure-preferences)，这个作者还搞了一大堆其他的加密相关的开源库，SecureSharedPreferences这个是以SharedPerference为基础，在此之上添加了许多加密算法，它和SharedPerference并没有什么不一样：
```java
SharedPreferences prefs = new SecurePreferences(context, "userpassword", "my_user_prefs.xml");
```
　　my_user_prefs.xml里的数据是经过加密的存放的：
```xml
<map>
    <string name="TuwbBU0IrAyL9znGBJ87uEi7pW0FwYwX8SZiiKnD2VZ7">
   pD2UhS2K2MNjWm8KzpFrag==:MWm7NgaEhvaxAvA9wASUl0HUHCVBWkn3c2T1WoSAE/g=rroijgeWEGRDFSS/hg
    </string>
    <string name="8lqCQqn73Uo84Rj">k73tlfVNYsPshll19ztma7U>
</map>
```

## 5.Key的存放 ##
　　如1中所说，预埋的Key要用NDK并加密存放，使用so库存储预设 key/secret，使用 Android KeyStore存储运行时动态获取到的私密内容.
　　这里主要贴几个链接，非常值得看：
　　https://developer.android.com/training/articles/security-tips.html
　　https://source.android.com/security/
　　https://source.android.com/security/keystore/index.html#top
　　https://developer.android.com/training/articles/keystore.html
　　https://drakeet.me/android-security-guide
未完待续....
