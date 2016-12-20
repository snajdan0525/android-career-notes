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


