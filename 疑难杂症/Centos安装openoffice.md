# Centos 安装openoffice

## 下载

```bash
wget https://jaist.dl.sourceforge.net/project/openofficeorg.mirror/4.1.9/binaries/zh-CN/Apache_OpenOffice_4.1.9_Linux_x86-64_install-rpm_zh-CN.tar.gz
--2021-04-28 10:33:26--  https://jaist.dl.sourceforge.net/project/openofficeorg.mirror/4.1.9/binaries/zh-CN/Apache_OpenOffice_4.1.9_Linux_x86-64_install-rpm_zh-CN.tar.gz
```

## 解压，得到zh-CN目录。

```bash
tar -zxvf Apache_OpenOffice_4.1.9_Linux_x86-64_install-rpm_zh-CN.tar.gz 
```



##  进入zh-CN文件夹下的RPMS目录下，执行yum localinstall *.rpm安装必要的包。（或者执行：rpm -Uvih *rpm）

```bash
rpm -Uvih *rpm

cd desktop-integration/
rpm -ivh  openoffice4.1.9-redhat-menus-4.1.9-9805.noarch.rpm
cd /opt/openoffice4/program/
yum install libXext.x86_64
yum groupinstall "X Window System"


## 启动
nohup soffice -headless -accept="socket,host=127.0.0.1,port=8100;urp;" -nofirststartwizard &
```



