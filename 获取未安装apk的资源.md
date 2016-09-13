---
title: 获取未安装的APK的资源
---


　　当我们要获取应用组件的资源时，通过调用getResource()获取到Resource对象，用过这个资源去访问图片，布局文集等等资源，同理要想获得插件的资源文件必须得到一个插件的Resource对象。
　　getResource()函数是Context的成员函数，且是abstract，看到这个，则根据以往的分析经验可以知道这个函数的实现就在ContextImpl类中。接下来就结合源码分析下如何获取插件的Resource对象.不过在这里我先给出一个函数过程调用的流程图,然后再去分析代码.
	![](http://i.imgur.com/LryF4gP.jpg)

```java
@Override
public Resources getResources() {
    return mResources;
}
```
　　接下来我们看下mResource是从哪里的得到值的。通过查看源码发现是在ContextImplt.init函数中得到值的。

```java
final void init(LoadedApk packageInfo,
            IBinder activityToken, ActivityThread mainThread,
            Resources container, String basePackageName) {
    mPackageInfo = packageInfo;
    mBasePackageName = basePackageName != null ? basePackageName : packageInfo.mPackageName;
    mResources = mPackageInfo.getResources(mainThread);
    if (mResources != null && container != null
            && container.getCompatibilityInfo().applicationScale !=
                    mResources.getCompatibilityInfo().applicationScale) {
        if (DEBUG) {
            Log.d(TAG, "loaded context has different scaling. Using container's" +
                    " compatiblity info:" + container.getDisplayMetrics());
        }
        mResources = mainThread.getTopLevelResources(
                mPackageInfo.getResDir(), container.getCompatibilityInfo());
    }
    mMainThread = mainThread;
    mContentResolver = new ApplicationContentResolver(this, mainThread);

    setActivityToken(activityToken);
}　　
```

　　继续跟入到 mResources = mPackageInfo.getResources(mainThread);这个代码后发现调用了getTopLevelResources来获取的Resource.继续跟入，代码如下：

```java
/**
 * Creates the top level Resources for applications with the given compatibility info.
 *
 * @param resDir the resource directory.
 * @param compInfo the compability info. It will use the default compatibility info when it's
 * null.
 */
Resources getTopLevelResources(String resDir, CompatibilityInfo compInfo) {
    ResourcesKey key = new ResourcesKey(resDir, compInfo.applicationScale);
    Resources r;
    synchronized (mPackages) {
        // Resources is app scale dependent.
        if (false) {
            Slog.w(TAG, "getTopLevelResources: " + resDir + " / "
                    + compInfo.applicationScale);
        }
        WeakReference<Resources> wr = mActiveResources.get(key);
        r = wr != null ? wr.get() : null;
        //if (r != null) Slog.i(TAG, "isUpToDate " + resDir + ": " + r.getAssets().isUpToDate());
        if (r != null && r.getAssets().isUpToDate()) {
            if (false) {
                Slog.w(TAG, "Returning cached resources " + r + " " + resDir
                        + ": appScale=" + r.getCompatibilityInfo().applicationScale);
            }
            return r;
        }
    }

    //if (r != null) {
    //    Slog.w(TAG, "Throwing away out-of-date resources!!!! "
    //            + r + " " + resDir);
    //}

    AssetManager assets = new AssetManager();
    if (assets.addAssetPath(resDir) == 0) {
        return null;
    }

    //Slog.i(TAG, "Resource: key=" + key + ", display metrics=" + metrics);
    DisplayMetrics metrics = getDisplayMetricsLocked(null, false);
    r = new Resources(assets, metrics, getConfiguration(), compInfo);
    if (false) {
        Slog.i(TAG, "Created app resources " + resDir + " " + r + ": "
                + r.getConfiguration() + " appScale="
                + r.getCompatibilityInfo().applicationScale);
    }
    
    synchronized (mPackages) {
        WeakReference<Resources> wr = mActiveResources.get(key);
        Resources existing = wr != null ? wr.get() : null;
        if (existing != null && existing.getAssets().isUpToDate()) {
            // Someone else already created the resources while we were
            // unlocked; go ahead and use theirs.
            r.getAssets().close();
            return existing;
        }
        
        // XXX need to remove entries when weak references go away
        mActiveResources.put(key, new WeakReference<Resources>(r));
        return r;
    }
}
```
上面这段代码中最重要的几行代码就是：
```java
	.....
    AssetManager assets = new AssetManager();
    if (assets.addAssetPath(resDir) == 0) {
        return null;
    }

    //Slog.i(TAG, "Resource: key=" + key + ", display metrics=" + metrics);
    DisplayMetrics metrics = getDisplayMetricsLocked(null, false);
    r = new Resources(assets, metrics, getConfiguration(), compInfo);
    if (false) {
        Slog.i(TAG, "Created app resources " + resDir + " " + r + ": "
                + r.getConfiguration() + " appScale="
                + r.getCompatibilityInfo().applicationScale);
    }
```
　　通过这段代码以及Android.content.res.AssetManager.java的源码我们发现了一个私有方法addAssetPath。只需要将apk的路径作为参数传入，就可以获得对应的AssetsManager对象，从而创建一个Resources对象，然后就可以从Resource对象中访问apk中的资源了。


　　**获取非安装APK即插件的的AssetManager对象**
```java
private void createApkAssetManager(String path) {
	try {
		AssetManager assetManager = AssetManager.class.newInstance();
		assetManager.getClass().getMethod("addAssetPath", String.class)
				.invoke(am, path);
		return assetManager;
	} catch (Exception o) {
	}
	return null;
}
```
　　**获取非安装APK即插件的Resource对象**

```java
private Resource getApkResource(AssetManager){
	return  new Resources(assetManager, super.getResources().getDisplayMetrics(),
	super.getResources().getConfiguration());
}
```


　　以下代码段是我们通过插件中的Resource来加载一个插件R.string.xx字符串的资源和图片资源的方法
```java
    imageView = (ImageView)findViewById(R.id.image_view_iv);
    if(resources != null){
        String str = resources.getString(resources.getIdentifier("test_str", "string", "net.mobctrl.normal.apk"));
        String strById = resources.getString(0x7f050001);
		//注意，id参照Bundle apk中的R文件
        System.out.println("debug:"+str);
        Toast.makeText(getApplicationContext(),strById, Toast.LENGTH_SHORT).show();
        Drawable drawable = resources.getDrawable(0x7f020000);//注意，id参照
        imageView.setImageDrawable(drawable);
    }
```
　　利用插件的Resource对象获取layout布局资源并转换成view对象

```java
DexClassLoader cl = new DexClassLoader(f.getAbsolutePath(),
				this.getApplicationInfo().dataDir, null, getClass()
						.getClassLoader());
		Class<?> layout = null;
		try {
			layout = cl.loadClass("com.example.dampingscrollview.R$layout");
		} catch (ClassNotFoundException e) {
			Log.i("MainActivity", "ClassNotFoundException");
		}
Field field = layout.getField("activity_host");
Integer obj = (Integer) field.get(null);
// 使用包含插件APK的Resources对象来获取这个布局才能正确获取插件中定义的界面效果
View view = LayoutInflater.from(this).inflate(
	res.getLayout(obj), null);
// 或者这样， 但一定要重写getResources方法， 才能这样写
// View view =
// LayoutInflater.from(MainActivity.this).inflate(obj,
// null);
```

　　至此，Over