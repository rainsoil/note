

##   常用命令
### 1. 初始化仓库

```
git init   # 在当前目录新建一个git 仓库
git init  [project-name]   #新建一个目录,将其初始化为git仓库

```
### 2. 将文件添加到仓库中
```
git add 文件名 # 将工作区的某个文件添加到暂存区
git add -u  # 添加将所有被tracked文件中被修改或者删除的文件信息到暂存区,不处理untracked 文件
git add -A  #添加所有被tracked文件中被修改或者删除的文件信息到暂存区
git add . # 将当前工作区的所有文件都加入到暂存区
git add -i #  进入交互界面模式,按需添加文件到暂停区
git add -p  # 添加每个变化前,都会要求确认,对于同一个文件的多处变化,可以实现分次提交
git rm [file1] [file2] # 删除工作区文件,并且将这次删除放入暂存区
git rm --cached [file] # 停止追踪指定文件,但是该文件会保留在工作区
git mv [file-original] [file-renamed] # 改名文件,并且将这个改名放入暂存区
```
### 3. 将暂存区的文件提交到本地仓库
```
git  commit -m "提交说明" # 将暂存区内容提交到本地仓库
git commit -a -m "提交说明" # 跳过缓存区操作,直接把工作区的内容提交到本地仓库
git commit -v # 提交时显示所有diff信息
git commit --amend -m "提交说明" # 使用一次新的commit,替代上一次提交.如果代码没有新变化,则用来改写上一次的commit的提交信息

```
### 4. 查看仓库当前状态
```
git status
```
### 5. 比较文件异同
```
git diff #工作区与暂存区的差异
git diff 分支名 # 工作区与某分支的差异,远程分支这样写:remotes/origin/分支名
git diff HEAD # 工作区与HEAD指针指向的内容差异
git diff  提交id 文件路径 # 工作区某文件当前版本与历史版本的差异
git diff --stage # 工作区文件与上次提交的差异(1.6版本前用--cached)
git diff 版本TAG # 查看从某个版本后的改动内容
git diff 分支A 分支B # 比较从分支A 和分支B的差异(也支持比较两个TAG)
# git diff 分支A...分支B # 比较两分支在分开后的各自改动
# 另外：如果只想统计哪些文件被改动,多少行被改动,可以添加 --stat 参数
```
### 6. 查看历史记录
```
git log # 查看所有的 commit 记录(SHA-A校验和,作者名称,邮箱,提交时间,提交说明)
git log -p -次数 #查看最近的多少次的提交记录
git log --stat # 简略显示每次提交的内容更改
git log --name -only # 仅显示已修改的文件清单
git log --name-status # 显示新增,修改,删除的文件清单
git log --online # 让提交记录已精简的一行输出
git log -graph -all --online # 图形展示分支的合并历史
git log --author=作者 #查询作者的提交记录(和grep同时使用要加一个 --all --match参数)
git log --grep=过滤信息 # 列出提交信息中包含过滤信息的提交记录
git log -S 查询内容 # 和--grep 类似，S和查询内容间没有空格
git log fileName #查看某文件的修改记录
```
### 7. 代码回滚
```
git reset HEAD^ # 恢复成上次提交的版本
git reset HEAD^^ # 恢复上上次提交的版本,就是多个^，以此类推或用~次数
git reset --hard 版本号
   ---soft :只是修改HEAD指针指向,暂存区和工作区不变
   ---mixed：修改HEAD 指针指向,暂存区内容修饰,工作区不变
   --hard ：修改HEAD指针指向,暂存区内容丢失,工作区恢复以前状态
```
### 8. 同步远程仓库
```
git push -u origin master
```
### 9. 删除版本库文件
```
git rm 文件名
```
### 10. 版本库里的版本替换工作区的版本
```
git checkout --test.txt
```
### 11. 本地仓库内容推送到远程仓库
```
git remote add origin git@github.com:账号名/仓库名.git
 //添加多个远程仓库
 git remote add origin2 git@github.com:账号名/仓库名.git
 git remote add origin4 git@github.com:账号名/仓库名.git
```

### 12. 从远程仓库克隆项目到本地
```
git  clone git@github.com:git账号名/仓库名.git
git clone -b 分支名仓库地址  下载指定分支的代码
```
### 13.创建分支
```
git checkout -b dev
 -b 表示创建分支并切换分支
 上面一条命令相当于两条
 git breanch dev // 创建分支
 git checkout dev // 切换分支
 
 git branch --set-upstream [branch] [remote-branch] # 建立追踪关系，在现有分支与指定的远程分支之间
```
### 14. 查看分支
```
git branch
```
### 15. 合并分支
```
git merge dev
# 用于合并指定分支到当前分支
git merge origin/serverfix   #  合并远程分支
git merge --no-ff -m "merge with no -ff" dev
#  加上 --no-ff 参数就可以使用普通的模式合并,合并后的历史有分支,能看出来曾经做过合并
```
### 16. 删除分支
```
git branch -d dev # 删除远程分支
git push origin --delete [branch-name] # 删除远程分支
git branch -dr [remote/branch] #删除远程分支
```
### 17. 查看分支合并图

```
git log --grap --pretty=oneline --abbrev -commit 
```
### 18. 查看远程信息
```
git remote 
// -v 显示更详细的信息
```
### 19. 相关信息
```
# 安装完Git后的第一件做药的事情就是设置用户信息(global 可以秀海为local 在单独项目生效):
git config --global user.name "用户名" # 设置用户名
git config --global user.email "用户邮箱" # 设置邮箱
git config --global user.name #  查看用户名是否配置成功
git config --global user.email # 查看邮箱是否配置

# 其他查看配置相关
git config --global --list # 查看全局设置相关参数列表
git config --local --list # 查看本地设置相关参数列表
git config --system --list # 查看系统配置参数列表
git config --list # 查看所有Git的配置(本地+全局+系统)
git config --global color.ui true # 显示git相关颜色
```
### 20 . 撤销某次提交
```
git revert HEAD # 撤销最近的一个提交
git revert 版本号 # 撤销某次commit
git reset HEAD~n  # 撤销本地的第n次提交
```
### 21. 拉取远程分支到本地仓库
```
git checkout -b  本地分支  远程分支 # 会在本地新建分支,并自动切换到该分支

git fetch origin 远程分支:本地分支 #会再本地新建分支,单不会自动切换,还需要checkout
git branch --set-upstream 本地分支 远程分支#建立本地分支与远程分支的链接
```

### 22. 标签命令
```
git tag # 列出所有tag
git tag [tag] # 新建一个tag在当前commit
git tag [tag] [commit] # 新建一个tag在指定commit
git tag -d [tag] # 删除本地tag
git push origin :refs/tags/[tagName] # 删除远程tag
git show [tag] # 查看tag信息
git push [remote] [tag] # 提交指定tag
git push [remote] --tags # 提交所有tag
git checkout -b [branch] [tag] # 新建一个分支，指向某个tag
```
### 23.  同步远程仓库更新
```
git fetch origin master #从远程获取最新的到本地,首先从远程的origin的master 分支下载最新的版本到origin/master 分支上,然后比较本地的master 分支和origin/master 分支的差别,最后进行合并
git fetch 比git pull 更加安全
```
### 24. 查看git 的创建时间

```
git for-each-ref --sort=-taggerdate --format "%(tag) %(taggerdate)" refs/tags
```

###  25 去除已经被git  托管的文件

```
git rm -r --cached .
```



## 常见问题

### git add 提交到暂存区,出错怎么办?

一般代码的提交流程： **工作区** >  `git status` 查看状态  -> `git add .` 将所有修改加入暂存区-> `git  commit -m "提交描述"` 将代码提交到本地仓库 ->`git  push `将本地仓库代码更新到远程仓库

####  场景1

当你改乱了工作区某个文件的内容,还添加到了暂存区, 想直接丢弃工作区的修改时, 用命令` git checkout -- file`

```bash
// 丢弃工作区的修改
git  checkout -- <文件名>
```

####  场景2

当你不但改乱了工作区的某个文件, 还添加到了暂存区,想丢弃修改, 分为两步, 第一步使用命令 `git reset HEAD file`, 就回到了场景1 , 第二步按照场景1操作.

### git  commit 提交到本地仓库出错怎么办

#### 1. 提交信息出错

更改commit 信息

```ba
git commit --amend -m "新提交消息" 
```

#### 2.  漏提交

commit 时, 遗漏提交部分更新, 有两种解决方案

- 方案一: 再次 commit 

  ```bash
  git commit -m“提交消息”
  ```

 此时, git上会出现两次commit

- 方案二： 遗漏文件提交到之前的commit 上

  ```bash
  git add missed-file // missed-file 为遗漏提交文件
  git commit --amend --no-edit
  ```

###  提交错误文件,回退到上一个commit 版本, 再commit

#### git reset

删除指定的 commit 

```bash
// 修改版本库，修改暂存区，修改工作区

git reset HEAD <文件名> // 把暂存区的修改撤销掉（unstage），重新放回工作区。
// git版本回退，回退到特定的commit_id版本，可以通过git log查看提交历史，以便确定要回退到哪个版本(commit 之后的即为ID);
git reset --hard commit_id 
//将版本库回退1个版本，不仅仅是将本地版本库的头指针全部重置到指定版本，也会重置暂存区，并且会将工作区代码也回退到这个版本
git reset --hard HEAD~1

// 修改版本库，保留暂存区，保留工作区
// 将版本库软回退1个版本，软回退表示将本地版本库的头指针全部重置到指定版本，且将这次提交之后的所有变更都移动到暂存区。
git reset --soft HEAD~1
```

####  git revert

撤销某次操作, 此次操作之前和之后的commit 和 history 都会保留, 并且把这次撤销作为一次最近的提交。

```bash
// 撤销前一次 commit
git revert HEAD
// 撤销前前一次 commit
git revert HEAD^
// (比如：fa042ce57ebbe5bb9c8db709f719cec2c58ee7ff）撤销指定的版本，撤销也会作为一次提交进行保存。
git revert commit
```

`git revert `和 `git reset ` 的区别

-  `git revert` 是用一个新的commit 来回滚之前的commit,`git reset`是直接删除指定的 commit
- 在回滚这一操作上看, 效果差不多, 但是在日后继续merge 以前的老版本的时候有区别，因为 `git ervert` 是用一次你想的 commit  中和之前的提交, 因此日后合并老的branch时,  导致这部分改变不会再出现, 但是`git reset` 是之前把某些 commit 在某个branch 上删除, 因此和老的branch 再次 merge的时候, 这些被回滚的 commit 应该还会被引入.
- `git reset`是把HEAD 向后移动了一下, 而 `git revert`是HEAD 继续前进, 只是新的commit 内容 要和 revert 的内容正好相反, 能够抵消要被revert 的内容.

