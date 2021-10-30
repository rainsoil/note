#  **CentOS7卸载openJDK安装JDK1.8**

CentOS7自带了一个openjdk，使用的时候用诸多问题，例如明明配置了Java环境变量但是不能使用，这个时候需要卸载重新安装。

### 查看已有openjdk版本

```
rpm -qa|grep jdk
```

### 卸载openjdk

remove后面的参数是上面得到的结果.noarch结尾的包

```
yum -y remove copy-jdk-configs-3.3-10.el7_5.noarch
```

### 下载jdk1.8

下载jdk-8u40-linux-x64.tar.gz，上传到/usr/local/soft/java

### 解压

```
tar -zxvf jdk-8u40-linux-x64.tar.gz -C /usr/local/soft/java
```

### 配置环境变量

在/etc/profile文件的末尾加上

```
export JAVA_HOME=/usr/local/soft/java/jdk1.8.0_40
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
```

编译使之生效

```
source /etc/profile
```

验证

```
java -version
```

