```java
private void useDexClassLoader() {
	// 创建一个意图，用来找到指定的apk
	Intent intent = new Intent("com.suchangli.android.plugin", null);
	// 获得包管理器
	PackageManager pm = getPackageManager();
	List<ResolveInfo> resolveinfoes = pm.queryIntentActivities(intent, 0);
	// 获得指定的activity的信息
	ActivityInfo actInfo = resolveinfoes.get(0).activityInfo;

	// 获得包名
	String pacageName = actInfo.packageName;
	// 获得apk的目录或者jar的目录
	String apkPath = actInfo.applicationInfo.sourceDir;
	// dex解压后的目录,注意，这个用宿主程序的目录，android中只允许程序读取写自己
	// 目录下的文件
	String dexOutputDir = getApplicationInfo().dataDir;

	// native代码的目录
	String libPath = actInfo.applicationInfo.nativeLibraryDir;
	// 创建类加载器，把dex加载到虚拟机中
	DexClassLoader calssLoader = new DexClassLoader(apkPath, dexOutputDir,
			libPath, this.getClass().getClassLoader());

	// 利用反射调用插件包内的类的方法

	try {
		Class<?> clazz = calssLoader.loadClass(pacageName + ".Plugin1");

		Object obj = clazz.newInstance();
		Class[] param = new Class[2];
		param[0] = Integer.TYPE;
		param[1] = Integer.TYPE;

		Method method = clazz.getMethod("function1", param);

		Integer ret = (Integer) method.invoke(obj, 1, 12);

		Log.i("Host", "return result is " + ret);

	} catch (ClassNotFoundException e) {
		e.printStackTrace();
	} catch (InstantiationException e) {
		e.printStackTrace();
	} catch (IllegalAccessException e) {
		e.printStackTrace();
	} catch (NoSuchMethodException e) {
		e.printStackTrace();
	} catch (IllegalArgumentException e) {
		e.printStackTrace();
	} catch (InvocationTargetException e) {
		e.printStackTrace();
	}
}
```