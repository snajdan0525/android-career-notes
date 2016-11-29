

## 依赖管理 ##
　支持多方式依赖管理：包括从 maven 远程仓库、 nexus 私服、 ivy 仓库以及本地文件系统的 jars 或者 dirs 。

**settings.gradle**
----------
> include ':app'

　在你的项目中如果有多个 Model 存在的时候，就可以选择包含哪些进行编译。

**build.gradle**
----------
![](http://i.imgur.com/lN7blzA.png)
　两个大的包围一看就明了,一个是为编译准备的,一个是为所有项目准备的。其中,Repositories 配置的是依赖管理的服务器。默认是 jcenter() 你可以添加其他，多个之间不干扰。
　dependencies这个也是依赖管理的东西，上面是指定依赖管理的服务器，这个就是具体依赖什么库。
　联合起来也就是,依赖jcenter()服务中的 gradle 库,其包名是：“com.android.tools.build”,版本是：2.2.2 版本

**app modle的build.gradle**
----------
