[TOC]
#### 1. 修改父项目和子项目的version
```
mvn versions:set -DnewVersion=1.4-RELEASE
>  有的说需要versions-maven-plugin 插件,经过证实,不需要

            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>versions-maven-plugin</artifactId>

                <version>2.3</version>
                <configuration>
                    <generateBackupPoms>false</generateBackupPoms>
                </configuration>
            </plugin>
```
#### 2. 打包
```
mvn  clean isntall -Dmaven.test.skip=true
```
>bat脚本
```
@echo off
echo git pull ..  && git pull && mvn clean install -U  -Dmaven.test.skip=true
```
#### 3. 上传私服
```
 mvn deploy -X -Dmaven.test.skip=true
```

>bat 脚本
```
@echo off
echo git pull ..  && git pull && mvn clean install -U  -Dmaven.test.skip=true && mvn deploy -X -Dmaven.test.skip=true


```
#### 4. 创建Maven的普通Java项目：
> mvn archetype:create -DgroupId=packageName -DartifactId=projectName
#### 5. 创建Maven的Web项目：
> mvn archetype:create -DgroupId=packageName -DartifactId=webappName-DarchetypeArtifactId=maven-archetype-webapp
#### 6. 编译源代码
>  mvn compile
#### 7. 编译测试代码
> mvn test-compile
#### 8. 运行测试
> mvn test
#### 9. 产生site
> mvn site
#### 10 打包
> mvn package 
#### 11. 在本地Repository 中安装jar
> mvn install 
#### 12. 清除产生的项目
> mvn clean 
#### 13. 生成eclipse 项目
> mvn eclipse:eclipse
#### 14. 生成idea 项目
> mvn idea:idea
#### 15. 发布第三方包到本地库
> mvn install:install-file -DgroupId=com -DartifactId=client -Dversion=0.1.0 -Dpackaging=jar -Dfile=d:\client-0.1.0.jar
### 16.maven archetype 模版化
#### 16.1 生成步骤
```
1. 新建项目
2. 执行 mvn archetype:create-from-project 命令
3. cd target/generated-sources/archetype
4. mvn install -U -Dmaven.test.skip=true
```
#### 16.2 使用步骤
```
 1. 在需要创建项目的目录执行  mvn archetype:generate -DarchetypeCatalog=local
 2. 输入要使用的archtype的序号
 3. 输入groupId
 4. 输入 artifactId 和version和package
 5. 确定生成 
```