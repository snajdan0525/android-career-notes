

## Gradle依赖管理 ##
　支持多方式依赖管理：包括从 maven 远程仓库、 nexus 私服、 ivy 仓库以及本地文件系统的 jars 或者 dirs 。

**新项目**
----------
![](http://i.imgur.com/1Abthrg.png)
　一个新的项目中就包含这些文件,build 是两个,一个是项目本身的,一个是 APP Model 的.另外在 APP 中可以看见有一个 manifest 文件夹，这意味着着可以有多 AndroidManifest 文件。另外值得一说的是 gradle.properties 文件也是含有两个，但是却是一个是全局，一个是项目的；这与上面的 Build 文件有何区别？区别在于全局文件存在于 C:Users用户名.gradle文件夹中，该文件有可能没有，需要自己创建，创建后所有项目都将具有访问权限，在该文件中一般保存的是项目的一些变量等，如果是无关紧要的变量可以保存在项目文件中，如果是用户名密码等变量则需要保存在全局文件中。

**settings.gradle**
----------
> include ':app'

　在你的项目中如果有多个 Model 存在的时候，就可以选择包含哪些进行编译。

**项目本身的build.gradle**
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
　这个也就是所谓的依赖了，在这里不光可以进行远程依赖（上面所说的方法）,也可以本地依赖： 
> compile fileTree(dir:'libs',include:['*.jar'])

这句话也就是说编译时依赖libs文件夹下的所有jar文件。

> 　compile project(':library')

这也是依赖，不过依赖的是一个model，前面说了在一个项目中可以有多个model，这句话的意思也就是依赖一个本项目中。名称为library的model库。

>compile "com.android.support:appcompat-v7:25.0.0"

　至于这句话也就是依赖一个远程的库了，这个库的作用是在低版本中使用一定的 Material Design 的东西。

**gradle文件中的android{...}部分**

    compileSdkVersion 25
    buildToolsVersion "25.0.0"
　这两个就是指定的编译SDK以及编辑工具版本，具体可以打开SDK Manager看看。
>     defaultConfig {
>         applicationId "io.github.snalopainen"
>         minSdkVersion 15
>         targetSdkVersion 25
>         versionCode 2
>         versionName "1.0.1"
>     }


**自定义Task**
----------

　下面是一个自定义的Task：
```groovy
class HelloWorldTask extends DefaultTask {
    @Optional
    String message = 'I am davenkin'

    @TaskAction
    def hello(){
        println "hello world $message"
    }
}

task hello(type:HelloWorldTask)


task hello1(type:HelloWorldTask){
   message ="I am a programmer"
}

```
　在上例中,我们定义了一个名为HelloWorldTask的Task,它需要继承自DefaultTask,它的作用是向命令行输出一个字符串。@TaskAction表示该Task要执行的动作,即在调用该Task时，hello()方法将被执行。另外，message被标记为@Optional，表示在配置该Task时，message是可选的。在定义好HelloWorldTask后，我们创建了两个Task实例，第一个hello使用了默认的message值，而第二个hello1在创建时重新设置了message的值。

**单独的项目中自定义Task类型**

----------
　创建另外一个项目，将buildSrc目录下的内容考到新建项目中，由于该项目定义Task的文件是用groovy写的，因此我们需要在该项目的build.gradle文件中引入groovy Plugin。另外，由于该项目的输出需要被其他项目所使用，因此我们还需要将其上传到repository中，在本例中，我们将该项目生成的包含了Task定义的jar文件上传到了本地的文件系统中。最终的build.gradle文件如下：
```groovy
apply plugin: 'groovy'
apply plugin: 'maven'
version = '1.0'
group = 'davenkin'
archivesBaseName = 'hellotask'

repositories.mavenCentral()

dependencies {
    compile gradleApi()
    groovy localGroovy()
}
uploadArchives {
    repositories.mavenDeployer {
        repository(url: 'file:../lib')
    }
}
```

　执行“gradle uploadArchives”，所生成的jar文件将被上传到上级目录的lib(../lib)文件夹中。

　在使用该HelloWorldTask时，客户端的build.gradle文件可以做以下配置：
```groovy
buildscript {
    repositories {
        maven {
            url 'file:../lib'
        }

    }

    dependencies {
        classpath group: 'davenkin', name: 'hellotask', version: '1.0'
    }
}
task hello(type: davenkin.HelloWorldTask)
```
　首先，我们需要告诉Gradle到何处去取得依赖，即配置repository。另外，我们需要声明对HelloWorldTask的依赖，该依赖用于当前build文件。之后，对hello的创建与（2）中一样。