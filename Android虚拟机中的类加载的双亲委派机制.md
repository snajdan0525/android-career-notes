依次来看一下
1) 系统类的加载器
[java] view plain copy 在CODE上查看代码片派生到我的代码片
Log.i("DEMO", "Context的类加载加载器:"+Context.class.getClassLoader());  
Log.i("DEMO", "ListView的类加载器:"+ListView.class.getClassLoader());  
从结果看到他们的加载器是：BootClassLoader，关于他源码我没有找到，只找到了class文件(用jd-gui查看)：

看到他也是继承了ClassLoader类。

2) 应用程序的默认加载器
[java] view plain copy
Log.i("DEMO", "应用程序默认加载器:"+getClassLoader());  
运行结果：

默认类加载器是PathClassLoader，同时可以看到加载的apk路径，libPath(一般包括/vendor/lib和/system/lib)

3) 系统类加载器
[java] view plain copy 在CODE上查看代码片派生到我的代码片
Log.i("DEMO", "系统类加载器:"+ClassLoader.getSystemClassLoader());  
运行结果：

系统类加载器其实还是PathClassLoader，只是加载的apk路径不是/data/app/xxx.apk了，而是系统apk的路径：/system/app/xxx.apk

4) 默认加载器的委派机制关系
[java] view plain copy 在CODE上查看代码片派生到我的代码片
Log.i("DEMO","打印应用程序默认加载器的委派机制:");  
ClassLoader classLoader = getClassLoader();  
while(classLoader != null){  
    Log.i("DEMO", "类加载器:"+classLoader);  
    classLoader = classLoader.getParent();  
}  
打印结果：

默认加载器PathClassLoader的父亲是BootClassLoader

5) 系统加载器的委派机制关系
[java] view plain copy
Log.i("DEMO","打印系统加载器的委派机制:");  
classLoader = ClassLoader.getSystemClassLoader();  
while(classLoader != null){  
    Log.i("DEMO", "类加载器:"+classLoader);  
    classLoader = classLoader.getParent();  
}  
运行结果：

可以看到系统加载器的父亲也是BootClassLoader