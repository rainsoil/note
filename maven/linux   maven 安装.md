#### 1. 安装jdk

#### 2. 下载maven
> 去http://maven.apache.org/download.cgi 下载maven 包

![image](9B167A8DDA7D41E3934139CE6D9298F5)
##### 2.1 下载到服务器
>  wget http://mirrors.shu.edu.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz
##### 2.2 解压
>   tar -zxvf apache-maven-3.5.4-bin.tar.gz
>
>   ##### 2.3  配置环境变量
```
  vim /etc/profile
  // 在最底层加入
export M2_HOME=你的maven安装路径
export PATH=${JAVA_HOME}/bin:${M2_HOME}/bin:$PATH
```
##### 2.4 刷新系统配置
> source /etc/profile
> ##### 2.5  验证
> 执行 mvn -v