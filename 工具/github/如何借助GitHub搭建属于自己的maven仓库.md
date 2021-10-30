# 借助GitHub搭建属于自己的maven仓库

## I. 背景

在Github上也写了不少的项目了，然后经常遇到的一个问题就是，很多自己写的项目，希望在另外一个项目中使用时，只能把这个项目下载下来，相当之不方便

因为大多数的java后端项目都是基于maven管理依赖的，所以就希望能有一个公共的maven仓库，可以把自己的项目扔进去，然后再应用就方便很多了

基于此，就有了本文这个教程了

## II. 实现步骤

### 1. github仓库建立

新建一个repository的前提是有github帐号，默认看到本文的是有帐号的

首先是在github上新建一个仓库，命令随意，如我新建项目为

- [github.com/liuyueyi/ma…](https://github.com/liuyueyi/maven-repository)

### 2. 配置本地仓库

本地指定一个目录，新建文件夹 `maven-repository`, 如我的本地配置如下

```
## 进入目录
cd /Users/yihui/GitHub

## 新建目录
mkdir maven-repository; cd maven-repository

## 新建repository目录
# 这个目录下面就是存放我们deploy的项目相关信息
# 也就是说我们项目deploy指定的目录，就是这里
mkdir repository

## 新增一个readme文档
# 保持良好的习惯，每个项目都有一个说明文档
touch README.md
复制代码
```

**这个目录结构为什么是这样的？**

我们直接看maven配置中默认的目录结构，同样拷贝一份出来而已

### 3. 仓库关联

将本地的仓库和远程的github仓库关联起来，执行的命令也比较简单了

```
git add .
git commit -m 'first comit'
git remote add origin https://github.com/liuyueyi/maven-repository.git
git push -u origin master
复制代码
```

接着就是进行分支管理了

- 约定将项目中的snapshot版，deploy到仓库的 snapshot分支上
- 约定将项目中的release版，deploy到仓库的 release分支上
- master分支管理所有的版本

所以需要新创建两个分支

```
## 创建snapshot分支
git checkout -b snapshot 
git push origin snapshot
# 也可以使用 git branch snapshot , 我通常用上面哪个，创建并切换分支

## 创建release分支
git checkout -b release
git push origin release
复制代码
```

### 4. 项目deploy

项目的deploy，就需要主动的指定一下deploy的地址了，所以我们的deploy命令如下

```
## deploy项目到本地仓库
mvn clean deploy -Dmaven.test.skip  -DaltDeploymentRepository=self-mvn-repo::default::file:/Users/yihui/GitHub/maven-repository/repository
复制代码
```

上面的命令就比较常见了，主要需要注意的是file后面的参数，根据自己前面设置的本地仓库目录来进行替换

### 5. deploy脚本

每次进行上面一大串的命令，不太好记，特别是不同的版本deploy到不同的分支上，主动去切换分支并上传，也挺麻烦，所以就有必要写一个deploy的脚本了

由于shell实在是不太会写，所以下面的脚本只能以凑合能用来说了

```
#!/bin/bash

if [ $# != 1 ];then
  echo 'deploy argument [snapshot(s for short) | release(r for short) ] needed!'
  exit 0
fi

## deploy参数，snapshot 表示快照包，简写为s， release表示正式包，简写为r
arg=$1

DEPLOY_PATH=/Users/yihui/GitHub/maven-repository/
CURRENT_PATH=`pwd`

deployFunc(){
  br=$1
  ## 快照包发布
  cd $DEPLOY_PATH
  ## 切换对应分支
  git checkout $br
  cd $CURRENT_PATH
  # 开始deploy
  mvn clean deploy -Dmaven.test.skip  -DaltDeploymentRepository=self-mvn-repo::default::file:/Users/yihui/GitHub/maven-repository/repository

  # deploy 完成,提交
  cd $DEPLOY_PATH
  git add -am 'deploy'
  git push origin $br

  # 合并master分支
  git checkout master
  git merge $br
  git commit -am 'merge'
  git push origin master
  cd $CURRENT_PATH
}

if [ $arg = 'snapshot' ] || [ $arg = 's' ];then
  ## 快照包发布
  deployFunc snapshot
elif [ $arg = 'release' ] || [ $arg = 'r' ];then
  ## 正式包发布
  deployFunc release
else
  echo 'argument should be snapshot(s for short) or release(r for short). like: `sh deploy.sh snapshot` or `sh deploy.sh s`'
fi
复制代码
```

将上面的脚本，考本到项目的根目录下，然后执行

```
chmod +x deploy.sh

## 发布快照包
./deploy.sh s
# sh deploy.sh snapshot 也可以

## 发布正式包
./deploy.sh r
复制代码
```

基于此，整个步骤完成

### III. 使用

上面仓库的基本搭建算是ok了，然后就是使用了，maven的pom文件应该怎么配置呢？

首先是添加仓库地址

**添加仓库**

如果要区分snapshot和release的话，如下配置

```
<repositories>
    <repository>
        <id>yihui-maven-repo-snap</id>
        <url>https://raw.githubusercontent.com/liuyueyi/maven-repository/snapshot/repository</url>
    </repository>
    <repository>
        <id>yihui-maven-repo-release</id>
        <url>https://raw.githubusercontent.com/liuyueyi/maven-repository/release/repository</url>
    </repository>
</repositories>
复制代码
```

如果不care的话，直接添加下面的即可

```
<repositories>
    <repository>
        <id>yihui-maven-repo</id>
        <url>https://raw.githubusercontent.com/liuyueyi/maven-repository/master/repository</url>
    </repository>
</repositories>
复制代码
```

仓库配置完毕之后，直接引入依赖即可，如依赖我的Quick-Alarm包，就可以添加下面的依赖配置

```
<dependency>
  <groupId>com.hust.hui.alarm</groupId>
  <artifactId>core</artifactId>
  <version>0.1</version>
</dependency>
复制代码
```

## IV. 其他


