# 6. Mybatis源码调试
----------

----------
### 1. 版本与源码
在引入jar之后，点开代码，首先是反编译的代码，没有相关的注释，在IDEA里面右上角会出现Download Sources和Choose Sources。IDEA里面直接下载官方的源代码包。

如果我们有自己下载的源码目录，包括一些我们自己加了注解的源代码，也可以用Choose Sources的方法选择源代码目录。切换源码版本，只要切换pom.xml里面的jar包版本就行了。

如果下载了其他版本的源码，例如带中文注释版，可以install，产生jar包以后，在pom.xml中修改依赖。
https://github.com/tuguangquan/mybatis（3.3.0，snapshot，非最新版本）

jar包里面的源代码是不可以修改的，想要自己给下载的源码加注释怎么办？可以添加source到下载的源码目录。