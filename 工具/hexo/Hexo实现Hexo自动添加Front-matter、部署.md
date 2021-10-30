# 实现hexo的自动添加Front-matter和部署

## 背景

我的笔记是存在gitee上的, 分类为目录名字.会有多层目录. 但是要把这些笔记转到hexo上的话,  有几个问题:

- 首先, hexo 是不支持多层目录结构的
- 第二,hexo 需要在文章头上添加 Front-matter的
- 第三, 每次我提交了笔记代码,还需要去服务器上执行一下 hexo的重新编译部署,比较麻烦

基于上面的这几个问题,我基于脚本和python 实现了自动添加Front-matter 和部署

## 自动添加Front-matter

hexoFile.py

```python
#!/usr/bin/python3
import os;
import shutil;
import time;
import datetime;



#  tag 列表
def alltagLists():
    tagList = [];
    tagList.append("Hexo");
    tagList.append("mybatis");
    tagList.append("前端");
    tagList.append("后端");
    tagList.append("javaScript");
    tagList.append("Github");
    tagList.append("架构");
    tagList.append("代码规范");
    tagList.append("面试");
    tagList.append("算法");
    tagList.append("Android");
    tagList.append("CSS");
    tagList.append("程序员");
    tagList.append("Vue.js");
    tagList.append("java");
    tagList.append("Node.js");
    tagList.append("数据库");
    tagList.append("设计");
    tagList.append("设计模式");
    tagList.append("前端框架");
    tagList.append("HTML");
    tagList.append("开源");
    tagList.append("产品");
    tagList.append("Linux");
    tagList.append("React.js");
    tagList.append("Git");
    tagList.append("Python");
    tagList.append("IOS");
    tagList.append("人工智能");
    tagList.append("Webpack");
    tagList.append("全栈");
    tagList.append("微信小程序");
    tagList.append("微信");
    tagList.append("Mysql");
    tagList.append("Google");
    tagList.append("HTTP");
    tagList.append("正则表达式");
    tagList.append("机器学习");
    tagList.append("黑客");
    tagList.append("JQuery");
    tagList.append("响应式设计");
    tagList.append("APP");
    tagList.append("创业");
    tagList.append("Chrome");
    tagList.append("Nginx");
    tagList.append("编程语言");
    tagList.append("命令行");
    tagList.append("产品经理");
    tagList.append("Docker");
    tagList.append("Redis");
    tagList.append("Mac");
    tagList.append("Angular.js");
    tagList.append("React Native");
    tagList.append("Bootstrap");
    tagList.append("Apple");
    tagList.append("图片资源");
    tagList.append("Photoshop");
    tagList.append("PHP");
    tagList.append("API");
    tagList.append("设计师");
    tagList.append("数据挖掘");
    tagList.append("Sublime Text");
    tagList.append("操作系统");
    tagList.append("gradle");
    tagList.append("阿里巴巴");
    tagList.append("Mongo DB");
    tagList.append("数据可视化");
    tagList.append("安全");
    tagList.append("招聘");
    tagList.append("Swift");
    tagList.append("Go");
    tagList.append("MVVC");
    tagList.append("Vuex");
    tagList.append("ReJava");
    tagList.append("Xcode");
    tagList.append("敏捷开发");
    tagList.append("Markdown");
    tagList.append("动效");
    tagList.append("运维");
    tagList.append("linux");
    tagList.append("字体");
    tagList.append("运营");
    tagList.append("云计算");
    tagList.append("物联网");
    tagList.append("Canvas");
    tagList.append("Icon");
    tagList.append("Spring");
    tagList.append("深度学习");
    tagList.append("爬虫");
    tagList.append("Objective-C");
    tagList.append("C++");
    tagList.append("虚拟现实");
    tagList.append("HTTPS");
    tagList.append("Eclipse");
    tagList.append("Debug");
    tagList.append("电子书");
    tagList.append("Ubuntu");
    tagList.append("NPM");
    tagList.append("测试");
    tagList.append("JSON");
    tagList.append("微服务");
    tagList.append("Ajax");
    tagList.append("DOM");
    tagList.append("Facebook");
    tagList.append("源码");
    tagList.append("VIM");
    tagList.append("Apache");
    tagList.append("TypeScript");
    tagList.append("游戏");
    tagList.append("Maven");
    tagList.append("SVG");
    tagList.append("Kotlin");
    tagList.append("Window");
    tagList.append("SEO");
    tagList.append("负载均衡");
    tagList.append("区块链");
    tagList.append("函数式编程");
    tagList.append("Gulp");
    tagList.append("SqlLite");
    tagList.append("浏览器");
    tagList.append("SQL");
    tagList.append("APK");
    tagList.append("Firefox");
    tagList.append("Flutter");
    tagList.append("Atom");
    tagList.append("Promise");
    tagList.append("Hadoop");
    tagList.append("嵌入式");
    tagList.append("机器人");
    tagList.append("IDEA");
    tagList.append("IntelliJ IDEA");
    tagList.append("Spring Boot");
    tagList.append("JVM");
    return tagList;


def matchTags(fileName, fileCategories, content):
    matchTagList = [];
    tagLists = alltagLists();
    for tag in tagLists:
        if tag.lower() in fileName.lower():
            matchTagList.append(tag);
        if tag.lower() in fileCategories.lower():
            matchTagList.append(tag);
        if tag.lower() in content.lower():
            matchTagList.append(tag);
    matchTagList = list({}.fromkeys(matchTagList).keys())
    return matchTagList;


def copyFile(sourceDir, targetDir):
    if os.path.exists(targetDir):
        shutil.rmtree(targetDir);
    shutil.copytree(sourceDir, targetDir);


##  获取文件的创建时间
def getFileCreateTime(file):
    createTime = os.path.getctime(file);
    timeStruct = time.localtime(createTime);
    return time.strftime('%Y-%m-%d %H:%M:%S', timeStruct);


# 遍历文件夹下的所有文件,并把结果存到一个列表中
def listFile(dir, fileList):
    newDir = dir;
    if os.path.isfile(dir):
        if os.path.basename(dir).endswith("md"):
            fileList.append(dir);
        ##  若只是想返回文件名,使用这个
        # fileList.append(os.path.basename(dir));

    elif os.path.isdir(dir):

        for s in os.listdir(dir):
            ##  如果需要忽略某些文件夹,使用一下代码
            if s == '.git':
                continue;
            newDir = os.path.join(dir, s);
            listFile(newDir, fileList);

    return fileList;


#########################开始干活#############################
#  列出所有的文件,将所有的组装成hexo支持的文件
sourceDir: str = "/web/hexo/note";
hexoDir: str = "/web/hexo/hexo_blog/blog/source/_posts";

## 清空hexo下的所有文件
for hexiFile in  os.listdir(hexoDir):
    print(hexiFile)
    os.remove(os.path.join(hexoDir,hexiFile));
# copyFile(sourceDir, targetDir);
fileList = [];
listFile(sourceDir, fileList);
for file in fileList:
    frontMatter = "---" + "\n";
    ## 分类
    categories = [];
    fileName = os.path.basename(file);
    fileCategorieStr = os.path.abspath(file).replace("\\", "/").replace(sourceDir, "").replace(fileName,
                                                                                               "");
    fileCategories = fileCategorieStr.split("/");
    if len(fileCategories) > 0:
        for fileCategorie in fileCategories:
            if len(fileCategorie) > 0:
                categories.append(fileCategorie);

    # 添加标题
    frontMatter = frontMatter + "title: " + fileName.replace(".md", "") + "\n";
    # 添加分类
    frontMatter = frontMatter + "categories:" + "\n";
    for ca in categories:
        frontMatter = frontMatter + "- " + ca + "\n";
    # # ##  进行文件拼接,并且移动文件到hexo文件夹中
    sourcefile = open(os.path.abspath(file), 'r+', encoding='utf-8');
    ## 添加标签
    content  =  sourcefile.read();
    tags = matchTags(fileName, fileCategorieStr,content)
    frontMatter = frontMatter + "tags: " + "\n";
    for t in tags:
        frontMatter = frontMatter + "- " + t + "\n";
    ## 文件创建时间
    createTime = getFileCreateTime(file);
    frontMatter = frontMatter + "date: "+createTime + "\n";
    frontMatter = frontMatter + "---" + "\n";
    print(time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()) + "---------fileName: " + file + "\n" + "-----------------Front-matter----------------------" +
          "\n" + frontMatter + "\n" + "------------------end---------------------")
    targetFile = open(hexoDir + "/" + os.path.basename(file).replace(' ',""), 'w', encoding='utf-8');
    targetFile.write(frontMatter + content);
    sourcefile.close();
    targetFile.close();


```

分类: 取得是文件夹名称

标签: 这里准备了一堆标签, 用文件名、分类名、和文章内容进行匹配

文章时间:  取得是文件的创建时间

然后最后将所有的文件都写入到 hexo的source/_posts 目录中

##  自动部署

我这里先拉去我的笔记的git代码.然后执行上面的 hexoFile.py 脚本实现 笔记的自动分类到hexo目录下,然后实现hexo的自动编译

hexorun.sh

```sh

##  先拉取笔记的代码
cd  ../note/  
git pull && cd /web/hexo/hexo_blog
pwd
python3 hexoFile.py  && cd  blog/  &&  hexo clean &&  hexo g

```

 最后使用python的定时任务.定时部署

schedule.py

```python
import time
import os
import sched

# 初始化sched模块的scheduler类
# 第一个参数是一个可以返回时间戳的函数，第二个参数可以在定时未到达之前阻塞。
schedule = sched.scheduler(time.time, time.sleep)


# 被周期性调度触发的函数
def execute_command(cmd, inc):
    print('执行主程序')

    ''''' 
    终端上显示当前计算机的连接情况 
    '''
    os.system(cmd)
    schedule.enter(inc, 0, execute_command, (cmd, inc))


def main(cmd, inc=60):
    # enter四个参数分别为：间隔事件、优先级（用于同时间到达的两个事件同时执行时定序）、被调用触发的函数，
    # 给该触发函数的参数（tuple形式）
    schedule.enter(0, 0, execute_command, (cmd, inc))
    schedule.run()


# 每60秒查看下网络连接情况
if __name__ == '__main__':
    main("sh hexorun.sh", 14400)

```

使用 nohup python3 schedule.py  &  就可以实现自动部署了

