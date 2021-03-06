

# Http  通信协议

## Http通信协议的基本原理

http协议在远程通信场景中的应用还是挺广泛的,包括现在主流的微服务架构的通信都是基于http协议,. 由于经常使用的关系,所以大家对于http协议的理解还是比较深刻的,我这里就直接帮大家梳理一下http协议的基本原理

### 一次Http协议的基本请求

我们先来思考一个问题,我们在浏览器输入一个网址后,浏览器是如何展示目标网址中的内容的? 内容是从哪里来的?

来通过图形来把这个过程画一下

![](http://files.luyanan.com//img/20190829204654.png)

DNS:（Domain Name System） 服务是和http协议一样处于应用层的协议. 它提供域名到IP地址的解析服务,用户通常使用主机名或者域名来访问对方的计算机，而不是直接通过IP地址访问. 因为与IP地址的一组数字相比,用字母配合数字的表示形式来指定计算机名更符合人类的习惯,

但是要让计算机去理解名称，相对而言就比较困难了. 因为计算机擅长处理一长串数字,为了解决上述的问题,DNS服务应运而生.DNS协议提供通过域名查找IP服务,或逆向从ip查找域名的行为.

## Http 通信协议的组成.

刚刚我们已经得知Http协议的工作过程,同时我们也应该知道HTTP协议是基于应用层的协议,并且在传输层使用的TCP的k可靠性通信协议, 既然是协议,那么就已应该符合协议的定义: 协议是两个需要网络通讯的程序达成的一种约定,它规定了报文的交换方式和包含的意义. 所以,接下来我们去深入的剖析HTTP协议的组成和原理.

### 请求URL 定位资源

我们在浏览器输入一个地址,浏览器是如何根据地址去找到服务器对应的资源并返回的? 以及这个地址包含了哪些有价值的i信息?

这就需要我们了解URL(Uniform Resource Locator) , 统一资源定位符,用于描述一个网络上的资源,具体格式如下:

URI用字符串标识某一互联网资源,而URL标识资源的地点(互联网所处的位置),可见URL是URI的子集。

> https://www.baidu.com/s?ie=utf-8&f=8#head
>
> schema://host[:port#]/path/.../?[url-params]#[query-string]
>
> schema  指定应用层使用的协议(例如http,https/ftp)
>
> host       HTTP服务器的ip地址或者域名
>
> port#       HTTP服务器的默认的端口号是80,这种情况下端口号可以省略,如果使用了别的端口号,必须指定端口号，例如 http://csdn:8080/
>
> path    访问资源的路径
>
> url-params      查询字符串
>
> query-string    片段标识符(使用片段标识符通常可标记已获取资源中的子资源(文档中的某个位置))

通过这个url地址,我们就可以读到,当前用户要使用http协议访问指定服务器上对应进程中的资源, 并且携带了请求参数。

### MIME TYPE

服务器根据用户请求的资源找到对应的文件以后,会返回一个资源给到客户端浏览器,浏览器会对这个资源解析并且渲染. 但是服务器上的资源类型有很多,比如图片类型,视频类型,JS、CSS、文本等. 浏览器是如何识别当前类型做不同的渲染的呢?MIME Type: 是描述消息内容类型的因特网标准,常见的几种类型:

- 文本文件: text/html,text/plain,text/css,application/xhtml+xml,application/xml
- 图片文件: image/jpeg,image/gif,image/png.
- 视频文件: video/mpeg,video/quicktime

我们可以通过两种方式来设置文件的渲染类型,第一种是Accept,  第二种是Content-type 

**Accept:** 表示客户端希望接受的数据类型,即告诉服务器我需要什么媒体类型的数据,此时服务器应该根据Accept 请求头生产指定媒体类型的数据.

**Content-type:** 表示发送端发送的实体数据类型,比如我们应该写过类似的: resposne.setContentType(“application/json;charset=utf-8”)的代码.表示服务器返回的数据格式是json

如果Accept 和 Content-type 不一致, 假如说 Accept 要接受的类型是 image/gif, dan是服务器端返回的数据是text/html,那么浏览器将会无法解析。

###  如果用户访问一个不存在的地址呢?

如果用户访问的地址没问题, 或者服务器也能正常解析以及处理当前用户的请求, 那就能返回正确的信息给到客户端. 但是如果用户访问的地址有问题, 或者服务端在解析用户请求以及处理请求逻辑时出现问题,怎么办呢? 浏览器应该怎么告诉用户当前是怎么处理失败的呢? 因为这里就设计到了一个状态码的概念

|      | 类别                             | 原因短语                   |
| ---- | -------------------------------- | -------------------------- |
| 1XX  | informational(信息性状态码)      | 接受的请求正在处理         |
| 2XX  | Success(成功状态码)              | 请求正常处理完毕           |
| 3XX  | Redirection(重定向状态吗)        | 需要进行附加操作以完成请求 |
| 4XX  | Client Error(客户端错误状态码)   | 服务器无法处理请求         |
| 5XX  | Server Error(服务器端错误状态码) | 服务器处理请求出错         |



大家见的比较多的错误码:

200: 一切正常

301: 永久重定向

404: 请求资源不存在

500: 服务器内部错误

有了状态码,在用户访问某个网站出现非正常状态时, 浏览器就可以友好的提示用户.

###  告诉服务器端当前请求的意图

有了url、mimetype、状态码 能够基本满足用户的需求. 但是, 很多时候一个网站不单纯只是不断的从服务端获取资源并做渲染, 可能还需要做一个数据的提交、删除等功能. 所以浏览器定义了8中方法来表示对于不同请求的操作方式,当然最常用的还是GET 和POST

##### GET

GET 一般是用户客户端发送一个URL地址去获取服务器资源(一般用于查询操作),Get不支持的传输数据有限制,具体限制由浏览器决定

##### POST

一般用于客户端传输一个实体给服务器端,让服务带你去保存(一般用户创建操作),

##### PUT

向服务器发送数据,一般用户更新数据操作

#####  DELETE

客户端发起一个Delete请求 要求服务端把某个数据删除(一般用于删除操作)

#####  HEAD

获取报文首部

#####  OPTIONS

询问支持 的方法

##### TRACE

追踪路径

#####  CONNECT

用隧道协议连接代理

在REST 架构风格中,由严格规定对于不同的请求类型要设置合适的请求方式, 也是避免出现乱用导致的混乱问题.  这里说一下为什么要定义REST这个架构风格

我个人认为是这样的:

1. 随着服务化的普及,http协议的使用频次越来越高.
2. 很多人在错误的使用 http协议定义接口, 比如 各种各样的命名,  什么getUserInfoById\deleteById  之类的, 有状态和无状态请求混用.
3. 对于http协议本身提供的规则并没有很好的利用,

所以, 为了更好的解决这些问题, 干脆就定义了一套规则, 这套规则并没有引入新的东西, 无非就是对http协议本身的使用做了一些约束,比如说:

1. REST 是面向资源的, 每一个URI 代表一个资源
2. 强调无状态化, 服务器端不能存储来自某个客户的某个请求中的信息, 并在该客户的其他请求中使用.
3. 强调URL 暴露资源时,不要在URI中出现动词
4. 合理的使用http状态码,请求方法.

因此大家在参照这种标准去使用REST 风格的时候,要明白你遵循的是什么以及要解决什么问题.

###  http协议的完整组成

ok,推演到这里,基本明白了一个http协议的基本组成,接下来简单总结一下, http协议包含两个报文, 一个是请求报文, 一个是响应报文.

####  请求报文

请求报文格式包含三个部分(起始行,首部字段,主体)

![](http://files.luyanan.com//img/20190902134831.png)

#### 响应报文

响应的报文格式也是一样的, 分为三部分

![](http://files.luyanan.com//img/20190902134908.png)

### HTTP协议中的扩展

http协议除了这两种组成以外, 还由很多大家比较常见的属性或者配置, 我也简单的罗列一下

####  如果上传的文件过大怎么办?

服务器返回的资源文件比较大,比如有些js文件大小可能就几兆. 文件过大可能就会影响传输的效率.同时也会带来带宽的消耗,怎么办呢?

1. 常见的手段是 对文件进行压缩,减少文件大小. 那压缩或者解压缩的流程怎么实现呢?

    首先服务器端需要能支持文件的压缩功能,其次浏览器能够针对被压缩的文件及逆行解压缩. 浏览器可以指定 Accept-Encoding 来告诉服务器我当前支持的编码类型.Accept-Encoding:gzip,deflate . 那服务器端会根据支持的编码类型,选择合适的类型进行压缩,  常见的编码方式由: gzip/deflate

2. 分割传输 

    在传输大容量数据的时候,通过把数据分割多块,能够让浏览器逐步显示页面, 这种把实体主体分块的功能成为 分块传输编码(Chunked Transfer Coding)

####  每次请求都要建立连接吗?

在最早的http协议中,每进行一次http通信,就需要做一次http通信. 而一次连接需要进行三次握手,这种通信方式会增加通信量的开销.

![](http://files.luyanan.com//img/20190902135804.png)

所以在http /1.1中改用了持久连接, 就是在一个连接建立之后,只要客户端或者服务端没有明确断开连接,那么这个tcp连接会一直保持连接状态.

持久连接的一个最大的好处是: 大大减少了连接的建立以及关闭时延.

HTTP 1.1 中有一个Transport 段 会携带一个 Connection: Keep-Alive ,表示希望将此条连接作为持久连接.

HTTP 1.1 持久连接在默认情况下是激活的,除非特别指明,否则HTTP 1.1 假定所有的连接都是持久的,要在事务处理结束后将连接关闭,HTTP 1.1 应用程序必须向报文中显示的添加一个Connection: close 首部.

HTTP 1.1 客户端 在收到响应后, 除非响应中包含了Conection: close 首部, 不然HTTP 1.1 连接就仍然维持在打开的状态,但是,客户端和服务器仍然可以随时关闭空闲的连接. 不发送Connection:close 并不意味着服务器承诺永远将连接保持在打开的状态.

管道化连接: http1.1 允许在持久连接上使用请求管道,以前发送请求后需要等待并收到响应, 才能发送下一个请求. 管线化技术出现后, 不用等待响应也可以直接发送下一个请求,这样就能够做到同时并行发送多个请求, 而不需要一个接一个的等待响应了.

![](http://files.luyanan.com//img/20190902141205.png)

## HTTP协议的特点

### http 无状态协议

HTTP 协议是无状态的, 什么是无状态呢? 就是说HTTP协议本身不会对请求和响应之间的通信状态做保存.

但是现在的应用都是由状态的,如果是无状态M那么这些应用基本没人用,你想想, 访问一个电商网站,先登陆,然后去选购商品, 当点击一个商品加入购物车以后又提示你登陆,这种用户体验根本不会有人去使用. 那么我们是怎么实现带状态的协议的呢?

### 客户端支持的Cookie

http协议中引入了Cookie技术,用来解决http协议无状态的问题, 通过在请求和响应报文中写入 Cookie 信息来控制客户端的状态，Cookie 会根据从服务器发送的响应报文中 一个叫Set-Cookie的首部字段信息, 通知客户端保存Cookie. 当下次客户端再往该服务求发送请求时，客户端会自动在请求报文中加入Cookie值后发送出去.

### 服务端支持的Session

服务端是通过什么方式来保存状态的呢? 在基于tomcat 这类的jsp/servlet 容器中,会提供Session 这样的机制来保存服务端的对象状态, 服务器会使用一种类似于散列表的结构来保存信息,当程序需要为某个客户端的请求创建一个session的时候,服务端首先先检查这个客户端的请求中是否包含了一个session标识 - session id

如果已经包含了一个session id 则说明以前已经为客户端创建过session, 服务器就按照session id 把session检索出来使用(如果检索不到,就新建一个)。

如果客户端请求不包含session id, 则为此客户端创建一个session 并且生成一个与此session 相关联的session id, session id的值是一个既不会重复,又不容易被找出规律的仿造字符串,这个session id 将会返回给客户端保存.

![](http://files.luyanan.com//img/20190902143122.png)

###  Tomcat 实现session的代码逻辑分析

我们以 HttpServletRequest# getSession() 作为切入点, 对Session的创建过程进行分析, 我们的应用程序拿到的HttpServletRequest 是org.apache.catalina.connector.RequestFacade(除非某些Filter 进行的特殊处理),它是 org.apache.catalina.connector.Request的门面模式. 首先,会判断Request 对象中是否存在Session, 如果存在并且未失效则直接返回.

如果不存在Session, 则尝试根据 requestdSessionId 查找Session,如果存在Session的话则直接返回, 如果不存在的话,则创建新的Session,并且把Sessionid 添加到Cookie 中, 后续的请求会携带该Cookid,这样便可以根据Cookie中的sessionId  找到原来创建的Session了

```java
  if (this.session != null && context.getServletContext().getEffectiveSessionTrackingModes().contains(SessionTrackingMode.COOKIE)) {
                            Cookie cookie = ApplicationSessionCookieConfig.createSessionCookie(context, this.session.getIdInternal(), this.isSecure());
                            this.response.addSessionCookieInternal(cookie);
                        }
```

## HTTP 协议自动分析

由于HTTP协议在通信过程中,是基于明文传输, 并且底层是基于TCP/IP 协议进行通信, 那么按照TCP/IP协议的工作机制,通信内容在所有的通信线路上都有可能遭到拦截和窃取. 窃取这个过程其实非常简单, 通过抓包工具 Wireshark 就可以截获请求和响应的内容.

### http安全传输

由于HTTP协议通信的不安全性,所以人们为了防止信息在传输过程中操作泄露或者篡改, 就像出来对传输通道进行加密的方式 https

https 是一种加密的超文本传输协议, 它与http 在协议差异在对数据传输的过程中,https 对数据做了完全加密. 由于 http协议或者https 协议都是处于tcp传输层之上,同时网络协议又是一个分层的结构,所以在tcp 协议层之上增加了 一层ssl(Secure Socket Layer ,安全层) 或者TLS(Transport Layer Security) 安全层传输协议组合使用 用于构造加密通道.

> SSL 是netscape 公司设计的(Secure socket layer ),后来互联网标准化组织ISOC接替了NetScapt 公司,发布了SSL的升级版TLS. 接着TLS的版本 又进行了多次升级, 实际上我们现在的HTTPS都是用的TLS协议, 但是由于SSL出现的时间较早,并且依旧被现在的浏览器支持,因此SSL依然是HTTPS的代名词

![](http://files.luyanan.com//img/20190902152042.png)

## 逆向推导https的设计过程

我们先不去探究ssl的实现原理, 我们先从设计者的角度去思考如何去建立一个安全的传输通道.

###  从第一个消息开始

客户端A 向服务端B 发送一条消息,这个消息可能会被拦截以及篡改, 我们如何做到A发送给B的数据包,即使被拦截了，也没办法得知消息内容并且也不能查看呢？

![1567409181236](C:\Users\luyanan\AppData\Roaming\Typora\typora-user-images\1567409181236.png)

###  利用对称加密

要做到消息不能被第三方查看以及篡改, 那么第一想法就是对内容进行加密,同时, 该消息还需要能被服务端进行解密,. 所以我们可以使用对称加密算法来实现, 密钥S 扮演者加密和解密的角色.在密钥S 不公开的情况下,就可以保证安全性?

![](http://files.luyanan.com//img/20190902153005.png)

###  没那么简单

在互联网世界,通信不会这么简单, 也许是这样

![](http://files.luyanan.com//img/20190902153051.png)

会存在多个客户端和服务端产生连接, 而这个客户端也许是一个潜伏着, 如果它也有对称密钥S,那相当于上面的方案是不可行的, 如果服务端和每个客户端通信的时候使用不同的加密算法呢?

![1567409622231](C:\Users\luyanan\AppData\Roaming\Typora\typora-user-images\1567409622231.png)

似乎能够完美的解决问题, 然后? 密钥如何分配呢? 那就是服务端告诉客户端该使用哪种对称加密算法呢? 解决方法似乎只能通过建立会话以后进行协商了.

###  协商过程又是不安全的

协商过程意味着又是基于一个网络传输的情况下去动态分配密钥,可是这个协商过程又是不安全的? 怎么破?

###  非对称加密出马

非对称加密算法的特点是: 私钥加密后的密文, 只要有公钥, 都能解密, 但是公钥加密后的密文只有私钥可以解密, 私钥只有一个人有, 而公钥可以发给所有人.

![](http://files.luyanan.com//img/20190902154239.png)

这样就可以保证A/B 向服务器方向发送的消息是安全的, 似乎我们可以通过非对称加密算法解决了密钥的协商的问题,但是

###  公钥怎么拿?

使用非对称加密,那么如何让A、B 客户端安全的持有公钥呢?

那么我们逐步思考, 有两种我们能想到的方案:

1. 服务器端将公钥发送给每一个客户端
2. 服务端将公钥放到一个远程服务器上,客户端可以请求到(多了一次请求,还得解决公钥的放置问题)

方案一似乎是不可行的, 因为传输过程又是不安全的, 公钥可能会被掉包

![](http://files.luyanan.com//img/20190902154659.png)



###  引入第三方机构

到上面这一步,最关键的问题是客户端如何直到给我 公钥的是黄蓉还是小龙女? 只能找本人去证实, 或者有一个第三者来帮你证实,并且第三者是绝对公平的.

所以,引入一个可信任的第三者是最好的方案

服务端把需要传递给客户端的公钥,通过第三方机构提供的私钥对公钥内容进行加密后,再传递给客户端,.通过第三方机构私钥对服务器端公钥加密以后的内容, 就是一个简陋版本的  "数字证书", 这个证书中包含了[服务器公钥]

![](http://files.luyanan.com//img/20190902155155.png)

客户端拿到这个证书后,因为证书是第三方机构使用私钥加密的, 客户端必须要有第三方机构提供的公钥才能解密证书.这块又涉及到第三方机构的公钥怎么传输? (假设是先内置在系统中)以及还有一个问题, 第三方机构颁发的证书是面向所有用户, 不会只针对一家发放,  如果不法分子也去申请一个证书呢?

###  如果不法分子也拿到证书?

如果不法分子也申请了证书,那它就可以对证书进行调包. 客户端在这种情况下是无法分辨出收到的是你的证书还是中间人的, 因为不论是中间人的还是你的证书，都能使用第三方机构的公钥进行解密.

![](http://files.luyanan.com//img/20190902160332.png)

![](http://files.luyanan.com//img/20190902160345.png)

###  验证证书的有效性

事情发展到现在,问题演变成了,客户端如何识别证书的真伪? 在现实生活中,要验证一个东西的真伪,绝大部分都是基于编号去验证的, 所以在这里 ,解决方案也是一样的,如果给这个证书添加一个证书编号,是不是就能达到目的呢?

 证书上写了如何根据证书的内容生成证书编号. 客户端拿到证书后根据证书上的方法自己生成一个证书编号,如果生成的证书编号与证书上的证书编号相同,那么说明这个证书是真实的. 这块有点类似于md5的验证, 我们下载一个软件包,都会提供一个md5的值,我们可以拿到这个软件包以后通过一个第三方软件去生成一个md5值去做比较,是不是一样, 如果一样就表示这个软件包没有被篡改过.

![1567411819291](C:\Users\luyanan\AppData\Roaming\Typora\typora-user-images\1567411819291.png)

对服务器端的数据进行MD5算法得到一个MD5的值,生成证书编号,使用第三方机构的私钥对这个证书编号进行加密,并且会在证书中添加证书编号的生成算法.

浏览器内置的CA公钥可以解密服务端CA私钥加密的证书,通过浏览器内置的CA证书的证书编号算法对服务端返回的证书编号进行验签.

![](http://files.luyanan.com//img/20190902161323.png)

###  第三方机构的公钥证书存在哪里?

浏览器和操作系统都会维护一个权威的第三方机构列表(包括他们的公钥)

因为客户端接受到的证书中会有颁发机构,客户端会根据这个颁发机构的值在本地找到相应的公钥.

说到这里,我想大家一定知道了,证书就是HTTPS中的数字证书, 证书编号就是数字签名,而第三方机构就是数字证书的签发机构(CA)

## HTTPS  原理分析

### HTTPS证书的申请过程

1. 服务器上生成的CSR文件(证书申请文件,内容包括证书公钥、使用的HASH签名算法、申请的域名、公司名称、职位等)

   ![](http://files.luyanan.com//img/20190902161904.png)

   

![](http://files.luyanan.com//img/20190902161927.png)

2. 把CSR文件和其他可能的证书上传到CA认证机构, CA机构收到证书申请后使用申请中的HASH算法,对部分内容进行摘要,然后使用CA 机构自己的私钥对这段摘要进行签名(相当于证书的唯一编号)
3. 然后CA机构把签名过的证书通过邮件形式发送到申请者手中
4. 申请者收到证书后部署到自己的web服务器上

###  客户端请求交互流程

1. 客户端发起i请求(Client Hello包)

   1. 三次握手,建立TCP连接
   2. 支持的协议版本(TLS/SSL)
   3. 客户端生成的随机数 client.random,后续用于生成 "对话密钥"
   4. 客户端支持的加密算法
   5. sessionId, 用于保持同一个会话(如果客户端与服务器端费尽周折建立了一个HTTPS连接,刚建完就断了,太可惜了)

2. 服务端收到请求,然后响应(Server Hello)

   1. 确认加密通道协议版本
   2. 服务端生成的随机数 server.random ,后续用于生成"对话密钥"
   3. 确认使用的加密算法(用于后续的握手消息进行签名防止篡改)
   4. 服务器证书(CA机构颁发给服务器端的证书)

3. 客户端收到证书进行验证

   1. 验证证书是否是上级CA 签发的，在验证证书的时候,浏览器会调用系统的证书管理器接口对证书路径中的所有证书进行一级一级的验证, 只有路径i中的所有证书是受信任的,这个验证的结果才是可信的.
   2. 服务端返回的证书中会包含证书的有效期,可以通过失效日期来验证证书是否过期
   3. 验证证书是否被吊销了
   4. 前面我们知道CA机构在签发证书的时候,都会使用自己的私钥对证书进行前面, 证书里面的签名算法字段 sha256RSA  表示CA机构使用sha256 对证书进行摘要,然后使用RSA算法对进行私钥签名, 而我们也知道RSA算法中, 使用私钥签名后,只有公钥才能进行验签.
   5. 浏览器内置在操作系统上的CA机构的公钥对服务器的证书进行验签, 确定这个证书是不是由正规机构签发的, 验证之后得到CA 机构使用sha256 进行证书摘要, 然后客户端再使用sha256 对证书内容进行一次摘要, 如果得到的值和服务区端返回的证书验签之后的摘要相同， 表示证书没有被修改过
   6. 验证通过后, 就会显示绿色的安全字样
   7. 客户端生成随机数, 验证通过后,客户端会生成一个随机数 pre-master secret ,客户端根据之前的 Client.random + server.random + pre-master 生成对称密钥然后使用证书中的公钥进行加密,同时利用前面协商好的HASH算法,把握手消息取HASH值, 然后用随机数加密"握手消息+握手消息、Hash值(签名)",并一起发送给服务端(在这里之所以要取握手消息的HASH值,主要是把握手消息的HASH值做一个签名,用于验证握手消息在传输过程中没有被篡改过)

4. 服务端接受随机数 

   1. 服务端收到客户端的加密数据以后, 用自己的私钥对密文进行解密,然后得到 client.random/server.random/pre-master secret,Hash值,并与传过来的HASH值做对比确认是否一致.
   2. 然后用随机密码加密一段握手消息(握手消息+握手消息的HASH值)给客户端

5. 客户端接受消息

   1. 客户端用随机数解析并计算握手消息的HASH值,如果与服务端发来的HASH一致.此时握手过程结束.
   2. 之后所有的通信数据将由之前交互过程中产生的 pre-master secret/client.random/server.random 通过算法得到session key,作为后续交互过程中的对称密钥

   
##  https 应用实战

接下来,为来让大家更好的理解https的原理, 我们基于Nginx 配置一个https的证书.,

在生产环境中的SSL证书都是需要通过第三方认证机构购买的, 分为专业版OV证书(浏览器地址栏上不显示企业名称)和高级版EV(可以显示企业名称)证书, 证书所保护的域名数不同也会影响价格(比如只对www认证和通配*认证,价格是不一样的),且不支持三级域名. 代表证书过期或者无效.,如果是黄色的话代表网站有部分连接使用的仍然是http协议.如果大家自己买了域名的话,可以在阿里云上申请一个免费的证书来使用

我们为了演示证书的申请过程,直接使用openssl, 自己作为证书颁发机构来制作证书， 但是这个证书是不受信任的.,所以到时候演示的结果浏览器上会提示证书不受信任的错误.

### 证书的申请过程

####  生成服务器证书的申请文件和私钥文件

在nginx的 conf 目录下创建 cert 文件夹, 在该文件夹下生成公私钥

```
openssl req -nodes -newkey rsa:2048 -out myreq.csr -keyout privatekey.key
```

req: 表示发出一个申请数字证书的请求

rsa:2048  表示加密算法以及长度

out : 输出一个请求文件

keyout:  生成私钥

myreq.csr:  证书签名请求, 这个并不是一个证书, 而是向权威机构获取签名证书的申请，它是主要内容是一个公钥.

privatekey.key :  与公钥相匹配的私钥.

CSR(证书请求文件),用来向CA机构申请的文件,一般以CSR结尾， 包含申请证书所需要的相关信息,其中最重要的是域名, 添加的域名必须是你要https 方式访问的那个域名.

一个是KEY文件,这个文件一定要保存好,这个文件就是对应server端的私钥, 如果key文件没有保存好, 是无法找回的, 因为key生成的过程是不可逆的, 即使添加的过程都是一样的, 生成的KEY 是不同的, 具有随机性.

####  模拟CA机构制作CA 机构证书

CA机构由自己的公私钥, CA会使用自己的公私钥对证书申请者提交的公钥进行加密, 所以为了模拟CA机构的工作流程,首先要创建一个CA的证书

openssl的配置文件  /etc/pki/tls/openssl.conf

以下是 openssl CA的默认配置, 我们需要配置CA的证书,就需要在指定的目录下创建相应的文件

![](http://files.luyanan.com//img/20190902173137.png)

1. 创建所需要的文件

   > touch /etc/pki/CA/index.txt 生成证书索引数据库文件
   >
   >  echo 01 > /etc/pki/CA/serial 指定第一个颁发证书的序列号, 必须是两位十六进制数, 99之后是9A

   ​        

2. CA 自签证书- 生成私钥

   >  cd /etc/pki/CA
   >
   > openssl genrsa -out /etc/pki/CA/private/cakey.pem 20148

3. 生成自签名证书

   >  openssl req -new -x509 -key /etc/pki/CA/private/cakey.pem -days 365 -out /etc/pki/CA/cacert.pem

4. 颁发证书

   >  openssl ca -policy policy_anything -in myreq.csr -out mycert.crt -days 3654
   >
   > policy policy_anything  policy  参数允许签名的CA 和网站证书可以有不同的国家、地名等信息
   >
   > out : ca颁发的证书文件
   >
   > days: 证书有效期

   

   ​        

#### nginx 配置https

在nginx.conf 中配置 server段， 将证书 mycert.pem 和私钥 pem 添加到指定文件中

```
server {
 listen 443 ssl;
 ssl on;
 ssl_certificate cert/mycert.crt;
 ssl_certificate_key cert/privatekey.key;
 ssl_session_cache shared:SSL:1m;
 ssl_session_timeout 5m;
 ssl_ciphers HIGH:!aNULL:!MD5;
 ssl_prefer_server_ciphers on;
 location / {
 root html;
 index index.html index.htm;
 }
 }
```



####  小tips

为什么证书一下子是以pem 结尾,一下又是以crt 结尾,一下又是key结尾  到底是什么意思？

其实,x509标准的证书,有两种编码格式, 一种的PEM，一种是 DER

但是实际上我们呢在创建证书和私钥的时候,并不一定要以PEM或者DER 作为扩展名, 

比如证书的表示形式有: PEM、DER、CRT、CER

私钥或者公钥的表示形式: PEM、DER、KEY

只是对应的编码格式不同而已.