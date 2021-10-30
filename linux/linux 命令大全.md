[TOC]
#### 1. 查看  HTTP  链接情况
```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```
#### 2. 查看当前IP个数和每IP连接数
```
netstat -an | grep 80 | awk '{print $5}' | awk 'BEGIN {FS=":"} NF==2 {print $1} NF==5 {print $4}' | sort | uniq -c | sort -n
```
#### 3. iptables 开放端口

3.1. 关闭firewall：
> systemctl stop firewalld.service

3.2. 停止firewall
> systemctl disable firewalld.service  

3.3. 安装安装iptables防火墙
> yum install iptables-services

3.4.升级
  ```
  yum update iptables 
--允许关联的状态包通过
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
  ```
3.5. 开放特定的端口，以80为例
> iptables -A INPUT -p tcp --dport 80 -j ACCEP

3.6. 重启
> systemctl restart iptables.service

3.7. 保存配置
> service iptables save

3.8. 设置开机自启动
> systemctl enable iptables.service 

#### 4. 列出相关目录下的所有目录和文件
> ls [选项名] [目录名] 
```
 -a 列出包括.a开头的隐藏文件的所有文件
 -A 通 -a,但是不列出 "."和".."
 -l 列出文件的详细信息
 -c 根据ctime 排序显示
 -t 根据文件修改时间排序
 -r  反序排列
 -S 以文件大小排序
 -h  以异度大小显示
 ---color[=WHEN] 用色彩辨别文件类型, WHEN 可以是'never','always'或'auto' 其中之一
   白色:表示普通文件
   蓝色:表示目录
   绿色:表示可执行文件
   红色:表示压缩文件
   浅蓝色:链接文件
   红色闪烁:表示链接的文件有问题
   黄色:表示设备文件
   灰色:表示其他文件
 
```
>>  例子
```
#  按易读的方式按时间排序,并显示文件详细信息
ls -lhrt
# 按照大小 反序显示文件的详细信息
ls -lrS
# 列出文件绝对路径（不包含隐藏文件）
ls | sed "s:^:`pwd`/:"
# 列出文件绝对路径（包含隐藏文件）
find $pwd -maxdepth 1 | xargs ls -ld
```
#### 5. 移动或重命名文件
>  mv [选项] 源文件或目录 目录或多个源文件 
```
-b 覆盖前做备份
-f 如存在不询问而强制覆盖
-i 如存在则询问是否覆盖
-u 较新才覆盖
-t 将多个源文件移动到统一目录下,目录参数在前,文件参数在后
 eg:
  mv a /tmp/ 将文件a 移动到 /tmp 目录下
  mv a b 将a 命令为b
  mv /home/zenghao test1.txt test2.txt test3.txt
```
#### 6.  将源文件复制至目标文件,或将多个源文件复制至目标目录
> cp [选项] 源文件或目录 目录或多个源文件   
```
-r -R 递归复制该目录以及子目录内容
-p 连同档案属性一起复制过去
-f 不询问而强制复制
-s 生成快捷方式
-a 将档案的所有特性都一起复制
```
#### 7. 在linux 服务器之间复制文件和目录
> scp [参数] [原路径] [目标路径]
```
-v 详细显示输入的具体情况
-r 递归复制整个目录
(1) 复制文件：  
命令格式：  
scp local_file remote_username@remote_ip:remote_folder  
或者  
scp local_file remote_username@remote_ip:remote_file  
或者  
scp local_file remote_ip:remote_folder  
或者  
scp local_file remote_ip:remote_file  
第1,2个指定了用户名，命令执行后需要输入用户密码，第1个仅指定了远程的目录，文件名字不变，第2个指定了文件名  
第3,4个没有指定用户名，命令执行后需要输入用户名和密码，第3个仅指定了远程的目录，文件名字不变，第4个指定了文件名   
(2) 复制目录：  
命令格式：  
scp -r local_folder remote_username@remote_ip:remote_folder  
或者  
scp -r local_folder remote_ip:remote_folder  
第1个指定了用户名，命令执行后需要输入用户密码；  
第2个没有指定用户名，命令执行后需要输入用户名和密码；
eg:
   从 本地 复制到 远程
   scp /home/daisy/full.tar.gz root@172.19.2.75:/home/root 
   从 远程 复制到 本地
   scp root@/172.19.2.75:/home/root/full.tar.gz /home/daisy/full.tar.gz
```
#### 8. 删除文件
> rm [选项] 文件

```
-r 删除文件夹
-f 删除不提示
-i 删除提示
-v 详细显示进行步骤
```
#### 9. 创建空的文件或更新文件时间
> touch [选项] 文件
```
-a 只修改存取时间
-m 只修改变动时间
-r  
  eg:touch -r a b ,使b的时间和a相同
-t 指定特定的时间 
  eg:touch -t 201211142234.50 log.log 
   -t time [[CC]YY]MMDDhhmm[.SS],C:年前两位
```
#### 10.查看当前所在的路径
> pwd
>
> >  例子
```
# 查看软链接的实际路径
pwd -p
```
#### 11. 改变当前目录
> cd 
```
- ：返回上次目录
.. :返回上层目录
回车  ：返回主目录
/   :根目录
```
#### 12. 创建新目录
> mkdir [选项] 目录
```
-p 递归创建目录,若父目录不存在则以此创建
-m 自定义创建目录的权限 
  eg: mkdir -m 777 hehe
-v 显示创建目录的详细信息
```
#### 13. 删除空的目录
> rmdir
```
-v 显示执行过程
-p 自父目录删除,若父目录为空,则一并删除
```
#### 14. 删除一个或多个文件或目录
> rm [选项] 文件
```
-f 忽略不存在的文件,不给出提示
-i 交互式删除
-r 将列出的目录以及子目录递归删除
-v 列出详细的信息
```
####  15 . 显示内容
> echo
```
-n 输出后不换行
-e 遇到转义字符特殊处理
```
#### 16. 一次显示整个文件或从键盘创建一个文件或将几个文件合并成一个文件
> cat [选项 文件
```
-n 编号文件内容再输出
-E 在结束行提示$
cat >  fileanme   #创建一个文件
cat file1 file2 > file  # 将几个文件合并为一个文件
```
#### 17. 反向显示
> tac
#### 18. 按页查看文章内容,从前向后读取文件,因此在启动的时候就加载整个文件
> more
```
+n 从第n行开始显示
-n 每次查看n行数据
+/string  搜寻string字符串位置,从其前两行开始查看
-c 清屏再显示
-d       提示“Press space to continue，’q’ to quit（按空格键继续，按q键退出）”，禁用响铃功能
-l        忽略Ctrl+l（换页）字符
-p       通过清除窗口而不是滚屏来对文件进行换页，与-c选项相似
-s       把连续的多个空行显示为一行
-u       把文件内容中的下画线去掉
```
> 常用操作命令
```
Enter    向下 n 行，需要定义。默认为 1 行
Ctrl+F   向下滚动一屏
空格键  向下滚动一屏
Ctrl+B  返回上一屏
=       输出当前行的行号
:f     输出文件名和当前行的行号
V      调用vi编辑器
!命令   调用Shell，并执行命令
q       退出more
```
#### 19. 可前后移动的逐屏查看文章内容,在查看钱不会加载整个文件
> less
```
-m 显示类似于more命令的百分比
-N 显示行号
/  字符串:向下搜索"字符串"的功能
?  字符串:向上搜索"字符串"的功能
n  重复前一个搜索(与/或?有关)
b  向后翻一页
d  向后翻半夜
```
> 常用命令
```
-i  忽略搜索时的大小写
-N  显示每行的行号
-o  <文件名> 将less 输出的内容在指定文件中保存起来
-s  显示连续空行为一行
/字符串：向下搜索“字符串”的功能
?字符串：向上搜索“字符串”的功能
n：重复前一个搜索（与 / 或 ? 有关）
N：反向重复前一个搜索（与 / 或 ? 有关）
-x <数字> 将“tab”键显示为规定的数字空格
b  向后翻一页
d  向后翻半页
h  显示帮助界面
Q  退出less 命令
u  向前滚动半页
y  向前滚动一行
空格键 滚动一行
回车键 滚动一页
[pagedown]： 向下翻动一页
[pageup]：   向上翻动一页
```
#### 20. 将输出内容自动加上行号
> nl [选项] ...[文件]...
```
-b 
-b a 不论是否有空格,都列出行号 (类似 cat -n)
-b t 空格则不列行号(默认)
-n 有ln,rn,rz 三个参数,分别为在最左方显示,最右方显示不加0,最右方显示加0
```
#### 21. 显示档案开头,默认显示10行
> head [参数]...[文件]...
```
-v  显示文件名
-c number 显示前number个字符,若number 为负数,则显示除最后number个字符的所有内容
-number/n (+)number 显示前number 行内容
-n number 若number为负数,则显示除最后number行数据的所有内容
```
#### 22. 显示文件结尾内容
> tail [必要参数] [选择参数]
```
-v 显示详细的处理信息
-q  不显示处理信息
-num/-n  (-)num  显示最后num 行内容
-c  显示最后c个字符
-f  循环读取
```
#### 23. 编辑文件
> vi
```
:w filename 将文章以指定的文件名保存起来  
:wq 保存并退出
:q! 不保存而强制退出
:set number 或者 :set nu 显示行号
:set  nonumber 或者 :set nonu 隐藏行号
命令行模式功能键
1）插入模式
   按「i」切换进入插入模式「insert mode」，按"i"进入插入模式后是从光标当前位置开始输入文件；
   按「a」进入插入模式后，是从目前光标所在位置的下一个位置开始输入文字；
   按「o」进入插入模式后，是插入新的一行，从行首开始输入文字。

2）从插入模式切换为命令行模式
 按「ESC」键。
3）移动光标
　　vi可以直接用键盘上的光标来上下左右移动，但正规的vi是用小写英文字母「h」、「j」、「k」、「l」，分别控制光标左、下、上、右移一格。
　　按「ctrl」+「b」：屏幕往"后"移动一页。
　　按「ctrl」+「f」：屏幕往"前"移动一页。
　　按「ctrl」+「u」：屏幕往"后"移动半页。
　　按「ctrl」+「d」：屏幕往"前"移动半页。
　　按数字「0」：移到文章的开头。
　　按「G」：移动到文章的最后。
　　按「$」：移动到光标所在行的"行尾"。
　　按「^」：移动到光标所在行的"行首"
　　按「w」：光标跳到下个字的开头
　　按「e」：光标跳到下个字的字尾
　　按「b」：光标回到上个字的开头
　　按「#l」：光标移到该行的第#个位置，如：5l,56l。

4）删除文字
　　「x」：每按一次，删除光标所在位置的"后面"一个字符。
　　「#x」：例如，「6x」表示删除光标所在位置的"后面"6个字符。
　　「X」：大写的X，每按一次，删除光标所在位置的"前面"一个字符。
　　「#X」：例如，「20X」表示删除光标所在位置的"前面"20个字符。
　　「dd」：删除光标所在行。
　　「#dd」：从光标所在行开始删除#行

5）复制
　　「yw」：将光标所在之处到字尾的字符复制到缓冲区中。
　　「#yw」：复制#个字到缓冲区
　　「yy」：复制光标所在行到缓冲区。
　　「#yy」：例如，「6yy」表示拷贝从光标所在的该行"往下数"6行文字。
　　「p」：将缓冲区内的字符贴到光标所在位置。注意：所有与"y"有关的复制命令都必须与"p"配合才能完成复制与粘贴功能。

6）替换
　　「r」：替换光标所在处的字符。
　　「R」：替换光标所到之处的字符，直到按下「ESC」键为止。

7）回复上一次操作
　　「u」：如果您误执行一个命令，可以马上按下「u」，回到上一个操作。按多次"u"可以执行多次回复。

8）更改
　　「cw」：更改光标所在处的字到字尾处
　　「c#w」：例如，「c3w」表示更改3个字

9）跳至指定的行
　　「ctrl」+「g」列出光标所在行的行号。
　　「#G」：例如，「15G」，表示移动光标至文章的第15行行首。
```
#### 24. 查看可执行文件的位置,在PATH变量指定的路径中查看系统命令是否存在及其位置
> which
#### 25. 定位可执行文件,源代码文件,帮助文件在文件系统中的位置
> whereis [-bmsu] [BMS 目录名 -f] 文件名
```
-b 定位可执行文件
-m 定位帮助文件
-s  定位源代码文件
-u  搜索默认路径下除可执行文件,源代码文件,帮助文件以外的其他文件
-B 指定搜索可执行文件的路径
-M 指定搜索帮助文件文件的路径
-S 指定搜索源文件的路径
```
#### 26. 通过搜寻数据库快速搜寻档案
> locate 
```
-r  使用正则运算式做寻找的条件
```
#### 27. 在文件树中查找文件,并作出相应的处理
> find [PATH] [option] [action]
```
选项与参数：
1. 与时间有关的选项：共有 -atime, -ctime 与 -mtime 和-amin,-cmin与-mmin，以 -mtime 说明
   -mtime n ：n 为数字，意义为在 n 天之前的『一天之内』被更动过内容的档案；
   -mtime +n ：列出在 n 天之前(不含 n 天本身)被更动过内容的档案档名；
   -mtime -n ：列出在 n 天之内(含 n 天本身)被更动过内容的档案档名。
   -newer file ：file 为一个存在的档案，列出比 file 还要新的档案档名

2. 与使用者或组名有关的参数：
   -uid n ：n 为数字，这个数字是用户的账号 ID，亦即 UID
   -gid n ：n 为数字，这个数字是组名的 ID，亦即 GID
   -user name ：name 为使用者账号名称！例如 dmtsai
   -group name：name 为组名，例如 users ；
   -nouser ：寻找档案的拥有者不存在 /etc/passwd 的人！
   -nogroup ：寻找档案的拥有群组不存在于 /etc/group 的档案！

3. 与档案权限及名称有关的参数：
   -name filename：搜寻文件名为 filename 的档案（可使用通配符）
   -size [+-]SIZE：搜寻比 SIZE 还要大(+)或小(-)的档案。这个 SIZE 的规格有：
       c: 代表 byte
       k: 代表 1024bytes。所以，要找比 50KB还要大的档案，就是『 -size +50k 』
   -type TYPE ：搜寻档案的类型为 TYPE 的，类型主要有：
       一般正规档案 (f)
       装置档案 (b, c)
       目录 (d)
       连结档 (l)
       socket (s)
       FIFO (p)
   -perm mode ：搜寻档案权限『刚好等于』 mode的档案，这个mode为类似chmod的属性值，举例来说，-rwsr-xr-x 的属性为4755！
   -perm -mode ：搜寻档案权限『必须要全部囊括 mode 的权限』的档案，举例来说，
       我们要搜寻-rwxr--r-- 亦即 0744 的档案，使用-perm -0744，当一个档案的权限为 -rwsr-xr-x ，亦即 4755 时，也会被列出来，因为 -rwsr-xr-x 的属性已经囊括了 -rwxr--r-- 的属性了。
   -perm +mode ：搜寻档案权限『包含任一 mode 的权限』的档案，举例来
       说，我们搜寻-rwxr-xr-x ，亦即 -perm +755 时，但一个文件属性为 -rw-------也会被列出来，因为他有 -rw.... 的属性存在！
4. 额外可进行的动作：
   -exec command ：command 为其他指令，-exec 后面可再接额外的指令来处理搜寻到的结果。
   -print ：将结果打印到屏幕上，这个动作是预设动作！
   eg:
       find / -perm +7000 -exec ls -l {} ; ,额外指令以-exec开头，以;结尾{}代替前面找到的内容
   | xargs 
       -i  默认的前面输出用{}代替 
       eg:
           find . -name "*.log" | xargs -i mv {} test4
```
#### 28.用正则表达式搜索文本,并把匹配的行打印出来
> grep "正则表达式" 文件名
```
-c  只输出匹配行的计数
-I 不区分大小写
-l 只显示文件名
-v 显示不包含匹配文本的所有行
-n 显示匹配数据以及行数
```
#### 29.判断文件类型
> file 
#### 30. 压缩,解压缩
> gzip [选项] 文件/文件夹
```
-a：使用ASCII文字模式；
-d：解开压缩文件；
-f：强行压缩文件。不理会文件名称或硬连接是否存在以及该文件是否为符号连接；
-h：在线帮助；
-l：列出压缩文件的相关信息；
-L：显示版本与版权信息；
-n：压缩文件时，不保存原来的文件名称及时间戳记；
-N：压缩文件时，保存原来的文件名称及时间戳记；
-q：不显示警告信息；
-r：递归处理，将指定目录下的所有文件及子目录一并处理；
-S或<压缩字尾字符串>或----suffix<压缩字尾字符串>：更改压缩字尾字符串；
-t：测试压缩文件是否正确无误；
-v：显示指令执行过程；
-V：显示版本信息；
-<压缩效率>：压缩效率是一个介于1~9的数值，预设值为“6”，指定愈大的数值，压缩效率就会愈高；
--best：此参数的效果和指定“-9”参数相同；
--fast：此参数的效果和指定“-1”参数相同
```

####  31. 多个目录或文件打包,压缩成一个大文件
> tar [主选项+辅选项] 文件或目录
```
主选项：
   -c  建立打包档案，可搭配 -v 来察看过程中被打包的档名(filename)
   -t  察看打包档案的内容含有哪些档名，重点在察看『档名』就是了；
   -x  解打包或解压缩的功能，可以搭配 -C (大写) 在特定目录解开
辅选项：
   -j  透过 bzip2 的支持进行压缩/解压缩：此时档名最好为 *.tar.bz2
   -z  透过 gzip 的支持进行压缩/解压缩：此时档名最好为 *.tar.gz
   -v  在压缩/解压缩的过程中，将正在处理的文件名显示出来！
   -f filename -f 后面要立刻接要被处理的档名！
   -C 目录   这个选项用在解压缩，若要在特定目录解压缩，可以使用这个选项。
   --exclude FILE：在压缩打包过程中忽略某文件 eg: tar --exclude /home/zenghao -zcvf myfile.tar.gz /home/* /etc
   -p  保留备份数据的原本权限与属性，常用于备份(-c)重要的配置文件
   -P(大写）  保留绝对路径，亦即允许备份数据中含有根目录存在之意；
eg:
   压 缩：tar -jcvf filename.tar.bz2 要被压缩的档案或目录名称
   查 询：tar -jtvf filename.tar.bz2
   解压缩：tar -jxvf filename.tar.bz2 -C 欲解压缩的目录
```
#### 32. 关机
> shutdown -n now
#### 33. 显示当前登录的用户
> users
#### 34. 登录在本机的用户与来源
> who 
```
-H 或 --heading  显示和栏位的标题信息列
```
#### 35.给当前联机的用户发消息
> wirite

#### 36. 查看用户的登陆日志
> last
> ####  37. 查看每个用户的最后登录时间
> lastlog
#### 38.查看用户信息
> finger [选项][使用者] [用户@主机]
```
-a 显示用户的注册名,实际姓名,终端名称,写状态,停滞时间,登录时间等信息
-l 除了用-s 选项显示的信息外,还显示用户主目录,登录shell,邮件状态等信息,以及用户主目录下的.plan,.project和.forward 文件的内容
-p 除了不显示.plan文件和.project 文件之外,与-l 选项相同
```
#### 39. 查看主机名
> hostname
#### 40. 别名
> alias ii = “ls -l”  添加别名

> unalias ii         清除别名
> #### 41.  新增用户
> .useradd [-u UID] [-g 初始群组] [-G 次要群组] [-c 说明栏] [-d 家目录绝对路径] [-s shell] 使用者账号名
```
-M 不建立用户目录,
-m  建立用户目录
-r 建立一个系统的账号,这个账号的UID会有限制
-e 账号失效日期,格式为[YYYY-MM=DD]
-D  查看useradd 的各项默认值
```
#### 42. 密码
> passwd
```
-l  密码失效
-u  与-l 相对,用户解锁
-S 列出登录用户passwd文件的相关参数
-n  后面接天数，shadow 的第 4 字段，多久不可修改密码天数
-x  后面接天数，shadow 的第 5 字段，多久内必须要更动密码
-w  后面接天数，shadow 的第 6 字段，密码过期前的警告天数
-i  后面接『日期』，shadow 的第 7 字段，密码失效日期
使用管道刘设置密码：echo "zeng" | passwd --stdin zenghao
```
#### 43. 删除用户
> userde 
```
-r 用户文件一并删除
```
#### 44. 修改用户密码的相关属性
> chage [-ldEImMW] 账号名
```
-l  列出该账号的详细密码参数；
-d  后面接日期，修改 shadow 第三字段(最近一次更改密码的日期)，格式YYYY-MM-DD
-E  后面接日期，修改 shadow 第八字段(账号失效日)，格式 YYYY-MM-DD
-I  后面接天数，修改 shadow 第七字段(密码失效日期)
-m  后面接天数，修改 shadow 第四字段(密码最短保留天数)
-M  后面接天数，修改 shadow 第五字段(密码多久需要进行变更)
-W  后面接天数，修改 shadow 第六字段(密码过期前警告日期)
```
#### 45. 修改用户的相关属性
> usermod [-cdegGlsuLU] username
```
-c  后面接账号的说明，即 /etc/passwd 第五栏的说明栏，可以加入一些账号的说明。
-d  后面接账号的家目录，即修改 /etc/passwd 的第六栏；
-e  后面接日期，格式是 YYYY-MM-DD 也就是在 /etc/shadow 内的第八个字段数据啦！
-f  后面接天数为 shadow 的第七字段。
-g  后面接初始群组，修改 /etc/passwd 的第四个字段，亦即是GID的字段！
-G  后面接次要群组，修改这个使用者能够支持的群组
-l  后面接账号名称。亦即是修改账号名称， /etc/passwd 的第一栏！
-s  后面接 Shell 的实际档案，例如 /bin/bash 或 /bin/csh 等等。
-u  后面接 UID 数字啦！即 /etc/passwd 第三栏的资料；
-L  冻结密码
-U  解冻密码
```
#### 46. 查看用户相关的id信息，还可以用来判断用户是否存在
> id [username] 
#### 47. 查看登陆用户支持的群组， 第一个输出的群组为有效群组
> groups
#### 48. 切换有效群组
> newgrp
#### 49. 添加组
> groupadd [-g gid] 组名 
```
-g 设定添加组的特定组的id
```
#### 50. 修改组信息
> groupmod [-g gid] [-n group_name] 群组名
```
-g  修改既有的 GID 数字
-n  修改既有的组名
```
#### 51. 删除群组
> groupdel [groupname] 
#### 52.群组管理员
> gpasswd 
```
root管理员动作：
   -gpasswd groupname 设定密码
   -gpasswd [-A user1,...] [-M user3,...] groupname
       -A  将 groupname 的主控权交由后面的使用者管理(该群组的管理员)
       -M  将某些账号加入这个群组当中
   -gpasswd [-r] groupname
       -r  将 groupname 的密码移除
群组管理员动作：
   - gpasswd [-ad] user groupname 
       -a  将某位使用者加入到 groupname 这个群组当中
       -d  将某位使用者移除出 groupname 这个群组当中
```
#### 53. 修改个人信息
> chfn
#### 54.分割
> cut
```
-b ：以字节为单位进行分割。这些字节位置将忽略多字节字符边界，除非也指定了 -n 标志。
-c ：以字符为单位进行分割。
-d ：自定义分隔符，默认为制表符。
-f  ：与-d一起使用，指定显示哪个区域。
```
#### 55.排序
> sort
```
-n   依照数值的大小排序。
-o<输出文件>   将排序后的结果存入指定的文件。
-r   以相反的顺序来排序。
-t<分隔字符>   指定排序时所用的栏位分隔字符。
-k  选择以哪个区间进行排序。
```
####  56. 统计指定文件中的字节数、字数、行数, 并将统计结果显示输出
> wc
```
-l filename 报告行数
-c filename 报告字节数
-m filename 报告字符数
-w filename 报告单词数
```
#### 57. 去除文件中相邻的重复行
> uniq
```
-c或——count：在每列旁边显示该行重复出现的次数；
-d或--repeated：仅显示重复出现的行列；
-f<栏位>或--skip-fields=<栏位>：忽略比较指定的栏位；
-s<字符位置>或--skip-chars=<字符位置>：忽略比较指定的字符；
-u或——unique：仅显示出一次的行列；
-w<字符位置>或--check-chars=<字符位置>：指定要比较的字符。
```
#### 58. 显示指定磁盘文件的可用空间,如果没有文件名被指定，则所有当前被挂载的文件系统的可用空间将被显示
<p>功能是为文件在另外一个位置建立一个同步的链接，当在不同目录需要该问题时，就不需要为每一个目录创建同样的文件，通过 ln 创建的链接（link）减少磁盘占用量。
链接分类：软件链接及硬链接
软链接
</p>

 硬链接
 - 硬链接，以文件副本的形式存在。但不占用实际空间。
 - 不允许给目录创建硬链接
 - 硬链接只有在同一个文件系统中才能创建

 软链接
 - 软链接，以路径的形式存在。类似于Windows操作系统中的快捷方式
 - 软链接可以 跨文件系统 ，硬链接不可以
 - 软链接可以对一个不存在的文件名进行链接
 - 软链接可以对目录进行链接


> du
```
-a  显示全部文件系统
-h  文件大小友好显示
-l  只显示本地文件系统
-i  显示inode信息
-T  显示文件系统类型
```
>> 例子
````
#  给文件创建软链接,并显示操作信息
ln -sv source.log link.log
#  给文件创建硬链接,并显示操作信息
ln -v source.log link1.log
#  给目录创建软链接
ln -sv /opt/soft/test/test3 /opt/soft/test/test5
````
#### 59.显示每个文件和目录的磁盘使用空间
> du [选项][文件]
```
-h 方便阅读的方式
-s 只显示总和的大小
```
#### 60. 某一个文件在另外一个位置建立一个同步的链接
> ln [参数] [源文件或目录] [目标文件或目录]
```
-s  建立软连接   
-v  显示详细的处理过程
```
#### 61. 比较单个文件或者目录内容
> diff [参数] [文件1或目录1] [文件2或目录2]
```
-b 　不检查空格字符的不同。
-B 　不检查空白行。
-i  不检查大小写
-q  仅显示差异而不显示详细信息
eg: diff a b > parch.log 比较两个文件的不同并产生补丁
```
#### 62. 显示或设定系统的日期与时间
> date [参数]… [+格式]
```
%H 小时(以00-23来表示)。 
%M 分钟(以00-59来表示)。 
%P AM或PM。
%D 日期(含年月日)
%U 该年中的周数。
date -s “2015-10-17 01:01:01″ //时间设定
date +%Y%m%d         //显示前天年月日
date +%Y%m%d --date="+1 day/month/year"  //显示前一天/月/年的日期
date +%Y%m%d --date="-1 day/month/year"  //显示后一天/月/年的日期
date -d '2 weeks' 2周后的日期
```
#### 63. 查看日历
> cal [参数] 月份] [年份]
```
-1  显示当月的月历
-3  显示前、当、后一个月的日历
-m  显示星期一为一个星期的第一天
-s  （默认）星期天为第一天
-j  显示当月是一年中的第几天的日历
-y  显示当前年份的日历
```
#### 64. 列出当前进程的快照
> ps
```
a   显示所有的进程
-a  显示同一终端下的所有程序
e   显示环境变量
f   显示进程间的关系
-H  显示树状结构
r   显示当前终端的程序
T   显示当前终端的所有程序
-au 显示更详细的信息
-aux    显示所有包含其他使用者的行程 
-u  指定用户的所有进程
```
#### 65. 显示当前系统正在执行的进程的相关信息，包括进程ID、内存占用率、CPU占用率等
> top
> #### 66.  杀死进程
> kill [参数] [进程号]
> #### 67.  显示linux系统中空闲的、已用的物理内存及swap内存,及被内核使用的buffer
> free [参数]
#### 68. 对操作系统的虚拟内存、进程、CPU活动进行监控
> vmstat
#### 69. 对系统的磁盘操作活动进行监视,汇报磁盘活动统计情况，同时也会汇报出CPU使用情况
> iostat [参数] [时间t] [次数n](每隔t时间刷新一次，最多刷新n次）
```
-p[磁盘] 显示磁盘和分区的情况
```
#### 70. 重复执行某一命令以观察变化
> watch [参数] [命令]
```
-n  时隔多少秒刷新
-d  高亮显示动态变化
```
#### 71. 在一个指定的时间执行一个指定任务，只能执行一次
> at [参数] [时间]
```
HH:MM[am|pm] + number [minutes|hours|days|weeks] 强制在某年某月某日的某时刻进行该项任务
atq 查看系统未执行的任务
atrm n 删除编号为n的任务
at -c n 显示编号为n的任务的内容
```
#### 72.  定时任务调度
> crontab 
```
file    载入crontab
-e  编辑某个用户的crontab文件内容
-l  显示某个用户的crontab文件内容
-r  删除某个用户的crontab文件
```
#### 73.  查看和配置网络设备
> ifconfig [网络设备] [参数] 
#### 74. 显示和操作IP路由表
> route
#### 75. 测试与目标主机的连通性
> ping [参数] [主机名或IP地址]
```
-q  只显示最后的结果
```
#### 76. 显示与IP、TCP、UDP和ICMP协议相关的统计数据
> netstat 
> #### 77.  用于远程登录，采用明文传送报文，安全性不好
> telnet [参数] [主机]
#### 78. 远程文件拷贝
> rcp [参数] [源文件] [目标文件] 
```
-r  递归复制
-p  保留源文件的属性
usage: rcp –r remote_hostname:remote_dir local_dir
```
#### 79.  直接从网络上下载文件
> wget [参数] [URL地址]
```
-o FILE 把记录写到FILE文件中    eg : wget -O a.txt URL
wget --limit-rate=300k URL  限速下载
```
#### 80. 对数据行进行替换、删除、新增、选取等操作
> sed 
```
a   新增，在新的下一行出现
c   取代，取代 n1,n2 之间的行 eg: sed '1,2c Hi' ab
d   删除
i   插入，在新的上一行出现
```
#### 81.  合并文件，需确保合并的两文件行数相同
> paste 
```
-d  指定不同于空格或tab键的域分隔符
-s  按行合并，单独一个文件为一行
```
####   82 授予用户文件夹权
> chown 
```
 chown -R 用户名   文件夹
```
####  83  df  命令   
<p> 显示磁盘空间的使用情况</p>
```
-a 全部文件系统列表
-h 以方便阅读的方式显示信息
-i 显示inode信息
-k 区块为1024字节
-l 只显示本地磁盘
-T 列出文件系统类型
```
#### 84. 用命令行运行deb安装包
```
如果ubuntu要安装新软件，已有deb安装包（例如：iptux.deb），但是无法登录到桌面环境。那该怎么安装？答案是：使用dpkg命令。
dpkg命令常用格式如下：
sudo dpkg -I iptux.deb #查看iptux.deb软件包的详细信息，包括软件名称、版本以及大小等（其中-I等价于--info）
sudo dpkg -c iptux.deb #查看iptux.deb软件包中包含的文件结构（其中-c等价于--contents）
sudo dpkg -i iptux.deb #安装iptux.deb软件包（其中-i等价于--install）
sudo dpkg -l iptux #查看iptux软件包的信息（软件名称可通过dpkg -I命令查看，其中-l等价于--list）
sudo dpkg -L iptux #查看iptux软件包安装的所有文件（软件名称可通过dpkg -I命令查看，其中-L等价于--listfiles）
sudo dpkg -s iptux #查看iptux软件包的详细信息（软件名称可通过dpkg -I命令查看，其中-s等价于--status）
sudo dpkg -r iptux #卸载iptux软件包（软件名称可通过dpkg -I命令查看，其中-r等价于--remove）
注：dpkg命令无法自动解决依赖关系。如果安装的deb包存在依赖包，则应避免使用此命令，或者按照依赖关系顺序安装依赖包。
```
##### 85. Ubuntu 18.04修改默认源为国内源

```
修改阿里源为Ubuntu 18.04默认的源

备份/etc/apt/sources.list
#备份
cp /etc/apt/sources.list /etc/apt/sources.list.bak

在/etc/apt/sources.list文件前面添加如下条目
#添加阿里源
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

最后执行如下命令更新源
##更新
sudo apt-get update
sudo apt-get upgrade

另外其他几个国内源如下： 
中科大源
##中科大源
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse

163源
##163源
deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse

清华源
##清华源
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
```
##### 86 修改用户为sudo 免密码
```
1. 编辑 sudo vi /etc/sudoers
2. 找到%sudo	ALL=(ALL:ALL) ALL
，在下边添加类似的一行
luyanan ALL = NOPASSWD: ALL
```

#####   87 查看进程

```
ps -ef|grep java、tomcat、maven、nginx
```

##### 88. 查看运行端口号

```
netstat -tunlp|grep java、nginx
```

#####  89. 全局搜索

```
find / -name '文件名'
```

######  90 防火墙  iptable

######   修改防火墙开放端口

```
vim /etc/sysconfig/iptables
```

######   重启防火墙

```
systemctl restart iptables.service
```



