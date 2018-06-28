Android Maven私有仓库搭建笔记

### 前言
###### &emsp;&emsp;什么是maven、gradle？

&emsp;&emsp;Maven是一个项目管理和自动构建工具。

&emsp;&emsp;Gradle是一个基于JVM的构建工具，是一款通用灵活的构建工具，支持maven， Ivy仓库，支持传递性依赖管理，而不需要远程仓库或者是pom.xml和ivy.xml配置文件，基于Groovy，build脚本使用Groovy编写。

&emsp;&emsp;Android支持的Maven仓库：

&emsp;&emsp;**mavenCentral** 是最早的 maven 中央仓库

&emsp;&emsp;**jcenter** 是 Android Studio 0.8 版本起的默认 maven **中央仓库**

&emsp;&emsp;**本机的仓库**

&emsp;&emsp;**部署在内网服务器的私有仓库**

### 一、为什么需要搭建maven私有仓库？

&emsp;&emsp;做java开发的童鞋对Maven一定不陌生；做android开发的童鞋，用得最多的是gradle。其实gradle的第三方库，也是放在maven仓库上。

&emsp;&emsp;对于第三方库，大家基本都配置maven、gradle从远程获取，估计很少直接下载jar放在工程里（对于没有放在maven repository上的库，只能这么干）。这么做方便管理依赖。

app开发中遇到问题:
&emsp;&emsp;做app开发，特别是只有几万行代码量的小项目，开发团队也就几个人，通常只用一个工程玩耍。随着业务扩展，工程变得越来越大，代码量大大增加，开发人数也多了，问题开始暴漏：改动一个地方往往影响到其他人的代码，功能模块耦合严重，构建速度慢....

&emsp;&emsp;业界一些解决方法：
&emsp;&emsp;1.组件化，按功能拆分出各种组件，数据存储、网络层、日志 等；
&emsp;&emsp;2.拆分业务，一个业务一个module；
&emsp;&emsp;3.业务插件化，一个业务一个工程，每个业务独立编译并运行.....

&emsp;&emsp;因此，引入依赖管理是必不可少的。把各个模块单独编译，部署上maven仓库，主工程or业务工程通过maven、gradle引用这些依赖。这么做还有好处，就是持续集成！某个模块修改了，跑单元测试，通过后才放上仓库。业务工程同步一下maven，万一有问题，还可以在服务端回滚到上一个版本。

&emsp;&emsp;所以我们希望通过搭建一个私有maven仓库，来提高我们的开发效率。

### 二、 使用Nexus搭建 maven 私服
###### &emsp;&emsp;Nexus是什么？

&emsp;&emsp;Nexus是一个基于maven的仓库管理的社区项目.主要的使用场景就是可以在局域网搭建一个maven私服,用来部署第三方公共构件或者作为远程仓库在该局域网的一个代理.简单举几个例子就是:

&emsp;&emsp;第三方Jar包可以放在nexus上,项目可以直接通过Url和路径配置直接引用.方便进行统一管理.

&emsp;&emsp;同时有多个项目在开发的时候,一些共用基础模块可以单独抽取到nexus上,需要用的项目直接从nexus上拉取就行(基础模块的实现,维护和部署可以交给专门的人员,其他项目不用关心代码实现,这样也可以达到保证核心代码不泄露).

&emsp;&emsp;封闭开发的过程中开发机是不能上公网的,所以连接central repository和下载jar就比较麻烦,这时就可以用nexus搭建起来一个介于公网和局域网之间的桥梁

### 三、所需工具

*   [JDK](http://www.oracle.com/technetwork/java/javase/downloads/index.html)

*   [Maven](https://maven.apache.org/)

*   [Nexus OSS](https://www.sonatype.com/download-oss-sonatype)

### 四、使用Nexus搭建 maven 私库

###### **1、Nexus下载**

&emsp;&emsp;官网下载地址：[https://www.sonatype.com/download-oss-sonatype](https://www.sonatype.com/download-oss-sonatype)，我的开发环境是Windows，我下载的是Nexus Repository Manager OSS 2.xx下面的 All platforms nexus-2.14.8-01-bundle.zip压缩文件。
![Nexus下载](https://upload-images.jianshu.io/upload_images/2783386-276489be260b216f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### **2、Nexus启动**

&emsp;&emsp;下载完成之后，解压后进入D:\xpkit\other\nexus-2.14.8-01-bundle\nexus-2.14.8-01\bin\jsw\windows-x86-64，根据操作系统类型选择文件夹，我选的是windows-x86-64文件夹，进入后可看到如下所示bat文件。

![Nexus解压后文件](https://upload-images.jianshu.io/upload_images/2783386-56c4bd1fd09e61be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


*  把zip包解压到指定路径如“D:\xpkit\other”
*  运行cmd然后进入“D:\xpkit\other\nexus-2.14.8-01-bundle\nexus-2.14.8-01\bin\jsw\windows-x86-64”路径
*  运行nexus.bat install命令安装nexus
*  运行nexus.bat start命令启动nexus
*  nexus.bat stop停止 nexus.bat restart重启 nexus.bat uninstall卸载

&emsp;&emsp;双击console-nexus.bat运行。再浏览器中输入[http://127.0.0.1:8081/nexus/](http://127.0.0.1:8081/nexus/),出现如下图所示就代表nexus已经启动成功。

  ![Neuxs运行成功](https://upload-images.jianshu.io/upload_images/2783386-0403157364ce2be9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### **3、登录Nexus**

&emsp;&emsp;管理nexus要以管理员身份登录，点击首页右上角的login输入默认登录名、密码admin/admin123即可登录。(如果是公司的局域网服务器换成局域网ip地址就可以了)。登录成功就可以看到如下界面了:

![nexus登录成功](https://upload-images.jianshu.io/upload_images/2783386-a79f3ec592f5bf2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;&emsp;这里的仓库分了四种类型：
&emsp;&emsp;**hosted(宿主仓库):**用来部署自己,第三方或者公共仓库的构件
&emsp;&emsp;**proxy(代理仓库):**代理远程仓库
&emsp;&emsp;**virtual(虚拟仓库):**默认提供了一个 Central M1虚拟仓库 用来将maven 2适配为maven 1
&emsp;&emsp;**group(仓库组):**统一管理多个仓库

&emsp;&emsp;名词解释：
&emsp;&emsp;**Public Repositories:** 仓库组
&emsp;&emsp;**3rd party: **无法从公共仓库获得的第三方发布版本的构件仓库
&emsp;&emsp;**Apache Snapshots: **用了代理ApacheMaven仓库快照版本的构件仓库
&emsp;&emsp;**Central:** 用来代理maven中央仓库中发布版本构件的仓库
&emsp;&emsp;**Central M1 shadow:** 用于提供中央仓库中M1格式的发布版本的构件镜像仓库
&emsp;&emsp;**Codehaus Snapshots: **用来代理
&emsp;&emsp;**CodehausMaven **仓库的快照版本构件的仓库
&emsp;&emsp;**Releases:** 用来部署管理内部的发布版本构件的宿主类型仓库
&emsp;&emsp;**Snapshots:**用来部署管理内部的快照版本构件的宿主类型仓库

###### **4、创建仓库**

&emsp;&emsp;这里以建立hosted仓库为例简单介绍Nexus在Android开发中的实际使用情况.点击Repositories &ndash;> Add &ndash;> Hosted Repository，键入ID(部署项目的标识) Name等属性,这里需要注意的是,如果该仓库有多次部署的情况的话,将policy设置为allow redeploy,不然后续在部署的时候会出现403错误。这里我点击添加宿主类型的仓库，在仓库列表的下方会出现新增仓库的配置，如下所示：

![新增仓库配置](https://upload-images.jianshu.io/upload_images/2783386-7fdf25349304e560.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;&emsp;点击save按钮后就会在仓库列表中看到刚才新增的仓库。

![新增仓库](https://upload-images.jianshu.io/upload_images/2783386-83b7d7dbcd5a6872.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 五、上传库到Maven仓库
**1.首先新建一个module，选择Android Library，类似下面这种结构**

![Android Library项目](https://upload-images.jianshu.io/upload_images/2783386-e7b18517f2d31d88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2.项目的根目录的gradle.properties配置一些相关信息，主要是一些全局的配置信息**

![gradle.properties](https://upload-images.jianshu.io/upload_images/2783386-6810a6b29d60dcc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**3.修改module对应的build.gradle文件，添加以下配置**

![build.gradle](https://upload-images.jianshu.io/upload_images/2783386-8093325d4c8f6552.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&emsp;&emsp;**注意：记得在module对应的build.gradle文件上面添加maven依赖apply plugin: 'maven'**

**4.点击uploadArchives进行编译上传**

![uploadArchives编译上传](https://upload-images.jianshu.io/upload_images/2783386-317a9aa073316a26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**5.去仓库查看到刚刚上传的库文件**

![查看库文件](https://upload-images.jianshu.io/upload_images/2783386-ec48820681cb94fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 六、在Android项目中应用Maven库文件

**1.新建一个项目，在项目的根目录build.gradle配置如下：**

![项目的根目录build.gradle配置](https://upload-images.jianshu.io/upload_images/2783386-1d388bfcf10f89e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2.在app目录下的build.gradle配置如下：**

![app目录下的build.gradle配置](https://upload-images.jianshu.io/upload_images/2783386-09be69e4ac4d196c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


&emsp;&emsp;至此配置就算结束了，就可以在Android项目中应用刚刚上传的Maven库文件了。

参考：

&emsp;&emsp;[1,拥抱 Android Studio 之四：Maven 仓库使用与私有仓库搭建](http://kvh.io/cn/embrace-android-studio-maven-deploy.html)

&emsp;&emsp;[2,使用Gradle和Nexus 搭建私有maven仓库](https://m.2cto.com/kf/201608/543685.html)

&emsp;&emsp;[3,Android的Nexus搭建Maven私有仓库与使用](https://blog.csdn.net/a565102223/article/details/62891676)

&emsp;&emsp;[4,Android业务组件化之Gradle和Sonatype Nexus搭建私有maven仓库](https://www.cnblogs.com/whoislcj/p/6490120.html)
