

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

![](http://i.imgur.com/g4jrC8s.png)

>apply plugin: 'com.android.application'

　表示的是添加插件，其是可以理解为该 model 为一个 com.android.application 程序，也就是应用程序，如果你的 Model 是一个库，那么自然也就是
> apply plugin: 'com.android.library'

**dependencies：**
　这个也就是所谓的依赖了，在这里不光可以进行远程依赖（上面所说的方法）,也可以本地依赖： compile fileTree(dir:'libs',include:['*.jar'])这句话也就是说编译时依赖libs文件夹下的所有jar文件。
　compile project(':library')这也是依赖，不过依赖的是一个model，前面说了在一个项目中可以有多个model，这句话的意思也就是依赖一个本项目中。名称为library的model库。

>compile "com.android.support:appcompat-v7:25.0.0"

　至于这句话也就是依赖一个远程的库了，这个库的作用是在低版本中使用一定的 Material Design 的东西。