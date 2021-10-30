#  CENTOS 7 和 JDK 添加中文字体

【1】在我们的 Windows 的 **C:\Windows\Fonts** 下面选择一个中文字体，如宋体，先拷贝到桌面，然后字体就变成了英文的：***\*SIMSUN.TTC\****

 ![img](https://img2018.cnblogs.com/blog/979767/201809/979767-20180920164849487-905318821.png)

备注：我这里只是写了 Windows 的，没有用过 Mac 系列的 ...

 

 【2】在服务器上面建立相关目录，为了便于区分，我们把目录名字叫做 ***\*zh_CN\****：

```
mkdir /usr/share/fonts/zh_CN
```

 

 【3】上传我们的字体到该目录下并改名为 **simsun.ttf**，上传可以在 CENTOS 上面 **yum 安装 lrzsz**，之后我们就能直接敲 ***\*rz\**** 命令或者拖拽进行交互式上传文件了：

```
cd /usr/share/fonts/zh_CN
mv SIMSUN.TTC simsun.ttf
```

 

 【4】收集系统的字体，保存到相关文件，此时会在当前目录生成 **fonts.scale** 文件：

```
yum -y install ttmkfdir
ttmkfdir -e /usr/share/X11/fonts/encodings/encodings.dir
```

 

【5】为了不重启机器，我们手动添加配置，强迫症顺便还帮他调整了一下格式：

```
vi /etc/fonts/fonts.conf

# 内容如下
<dir>/usr/share/fonts/zh_CN</dir>
```


# 给JDK添加中文字体

 

 由于 JDK 添加中文字体比较简单，这里就直接给出方法：

```
# 进入 JDK 的目录
cd /usr/local/jdk1.7.0_79/jre/lib/fonts
yum install mkfontscale
yum install fontconfig
# 创建目录
mkdir fallback
cd fallback

# 将公共系统那个中文字体拷贝过来
cp /usr/share/fonts/zh_CN/simsun.ttf .

# 生效
mkfontscale
mkfontdi
```