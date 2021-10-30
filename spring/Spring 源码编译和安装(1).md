# 1. Spring 源码编译和安装



-----------
### Spring5源码下载注意事项
首先你的JDK需要升级到1.8以上。Spring3.0开始,Spring源码采用github托管，不再提供官网下载
链接。这里不做过多赘述，大家可自行去github网站下载，我们使用的版本下载链接为：
https://github.com/spring-projects/spring-framework/archive/v5.0.2.RELEASE.zip，下载完成后，
解压源码包会看到以下文件目录：

### 基于Gradle的源码构建技巧
由于Spirng5以后都是采用Gradle来编译，所以构建源码前需要先安装Gradle环境。Gradle下载地
址：https://gradle.org/releases，我使用的是Spring5官方推荐的版本Gradle4.0,下载链接为：
https://gradle.org/next-steps/?version=4.0&format=bin，下载完成后按以下步骤操作，以
Windows操作系统为例：
第一步：配置环境变量
![image](http://files.luyanan.com/07e95fdf-be39-4352-8153-be595c186989.jpg)
第二步：添加环境变量：Path：%GRADLE_HOME%\bin
![image](http://files.luyanan.com/4b4beb1a-a240-49a8-ac84-b0dd540e36ff.jpg)
第三步：检测环境，输入gradle -v命令，得到以下结果：
![image](http://files.luyanan.com/c0dd1283-3be6-44cf-bf33-aa3841e05ad2.jpg)
第四步：编译源码,cmd 切到 spring-framework-5.0.2.RELEASE 目录，运行gradlew.bat
![image](http://files.luyanan.com/a7c0d97a-1eca-49ee-9505-fc272c2c1369.jpg)
第五步：转换为eclipse项目，执行import-into-eclipse.bat命令，构建前，请确保网络状态良好，按
任意键继续。
![image](http://files.luyanan.com/5b8338d7-dd35-4606-98be-82b7a72fcacb.jpg)
第六步：等待构建成功（若中途出现错误，大部分情况是由于网络中断造成的，重试之后一般都能解决
问题），构建成功后，会出现如下界面：
![image](http://files.luyanan.com/0bf4c7d1-e8a0-45c3-995d-658c2b390d30.jpg)
到这一步为止，还在使用Eclipse的小伙伴已经可以将项目导入到Eclipse中了。而我们推荐使用的IDEA
也比较智能，可以直接兼容Eclipse项目。接下来看下面的步骤：
第七步：导入IDEA。打开IntelliJ IDEA，点击Import Project，弹出如下界面，选择
spring-framework-5.0.2.RELEASE文件夹:

第八步：等待构建完成，在网络良好的情况下大约需要10分钟便可自动构建完成，你会看到如下界面：

第九步：在IDEA中，如果Project下的子项目文件夹变成粗体字之后，说明已经构建成功。还有一种
验证方式是：找到ApplicationContext类，按Ctrl + Shift + Alt + U，出现类图界面说明构建成功。


### Gradle构建过程中的坑
如果项目环境一直无法构建，项目文件夹没有变粗体字，类图无法自动生成。那么你一定是踩到了这样一个坑。
第一步：首先打开 View->Tool Windows -> Gradle
![image](http://files.luyanan.com/76f322dc-7cce-4af8-9015-bc7667894516.jpg)
然后，点击右侧Gradle 视图中的 Refresh，会出现如下的错误：
![image](http://files.luyanan.com/2a7b3474-6ff7-427c-ac33-70146491a651.jpg)

![image](http://files.luyanan.com/86344312-7f70-4378-a844-f93d52d96f6a.jpg)
第二步：看错误，显然跟Gradle 没有任何关系，解决办法：
1.关闭 IDEA，打开任务管理器，结束跟 java有关的所有进程。
2.找到 JAVA_HOME -> jre -> lib目录，将 tools.jar 重命名 tools.jar.bak。
3.重启 IDEA，再次点击refresh，等待构建完成。