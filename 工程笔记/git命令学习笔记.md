# git学习笔记，精简常用版

[TOC]

## 使用流程

每步代码中选择使用，不全用，如第一步两个代码是互斥的，只用其中一个。

```bash
# step1 建立一个仓库或者下载一个仓库
git init
git clone
# step2 切分支
git branch
git checkout -b
git fetch
git pull
# step3 修改提交等操作
git status
git add
git commit
git log
git reset
# step4 合并分支提交
git rebase
git merge
git branch -d
# step5 提交修改
git push
```



## 分支管理建议

一个仓库应有master、release、develope、hotfix、$author等多个分支

- release：当有新的发布版本时将发布版从master合并进release分支

- master：当有稳定版本时从develope分支合并进master分支，当要发布时将该分支合并进releash

- hotfix：当有bug修复时从release创建hotfix分支，然后修复后合并进release分支作为补丁发布
- develope：各个developer在自己的分支上完成工作后合并进该分支，测试稳定后再从该分支合并进mashter分支
- $author：每个developer都应该有一个自己的分支，平时在自己的分支上工作，完成新功能并测试后再合并进develope分支
- 每个用户自己有新的功能需要添加了根据自己需求从各自主分支（$author)再新建分支完成新功能，完成后合并会自己的分支，debug也同理。

合并分支时尽量用git rebase或者git merge --squash合并commit，只留有意义的commit，保持分支和日志的整洁。

平时应多commit，少push，但合并时却不应该所有commit都合并进去。

## 我的zsh别名

``` bash
alias gau="git add -u"
alias gck="git checkout"
alias gcm="git commit -m"
alias gba="git branch -a"
alias gb="git branch"
alias glg="git log --graph"
alias glo="git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --all"
```

## 名词解释

- HEAD是当前所在版本的位置，比如有b1,b2,b3三个分支，当前git checkout 到了b1最后一次commit的位置，则HEAD就指向b1最后一次提交的版本的位置。

- HEAD~n代表从HEAD往回算n个commit，这n个按时间排序往回找，其他合并进来分支的commit也算，比如b2合并进b1了两个commit最新，那就算这两个。

  > git rebase中HEAD~n表示从HEAD往回算n个commit的集合
  >
  > git reset中HEAD~n表示从HEAD往回算第n个commit的个体

- origin是远程仓库的代号，如果有多个远程仓库，要修改origin为目标仓库，只有一个的话一般就写origin就行，这个参数用于fetch push pull等命令
- 暂存区、跟踪等不再解释了

## git init：初始化一个新的本地仓库代码

常用：

``` bash
git init
```

一个仓库的初始状态，建立一个本地仓库要么`git init`要么`git clone`

## git status：查看当前状态

常用：

```bash
git status
```

例：

``` bash
# git status
tracked:
...
untracked:
...
```

untracked下面的是已修改，未git add的文件，需要git add或者修改`.gitignore`文件设置忽略

tracked下面的是已修改并git add了的文件

当untracked下面没有文件，tracked下面有文件时即可git commit

untracked和tracked下面都没有文件时显示clean，表示当前分支时干净的

只有干净的分支可以执行push、pull、merge、rebase等操作，不干净也想执行需要git stash或者git reset

## git stash：暂存当前修改

常用：

```bash
git stash
git stash pop
```



用于使当前分支clean，然后执行merge等操作，然后使用`git satsh pop`弹出暂存的修改。

## git log：查看日志

常用：

```bash
git log		# 简单查看日志
git log $branchx	# 只查看branchx的日志
git log $branchx --graph	#图形化查看分支branchx的日志
git log --graph	#图形化查看所有分支的日志
glo log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --all # 很全的图像化显示，也很乱
```

--graph图形化日志详解：

``` bash
# 举个简单点的例子，下面的日志是使用log前一行出现的，test1分支只合并过test2分支
# git log test1 --graph
*   commit 28eead0714fd8a6ff04c478f2bab51530ddbe951
|\  Merge: 8ff1ddb 22c0639
| | Author: yjp's Ubuntu16 <yjp@hust.edu.cn>
| | Date:   Wed Jan 16 23:37:55 2019 -0800
| | 
| |     merge test2 to test1
| |   
| * commit 22c0639248caeaab5360315a86d0b971d6ee4b75
| | ...
| |     edit6
| * commit c09df24a24f9f9ea2040385405c98eb2ab1bc68c
| | ...
| |     edit5
* | commit 8ff1ddb0c6ad53d9bb7b7e55ad26a5f491cb15cf
| | ...
| |     new1
* |   commit 9b531eb1741bbe34e9ea1ee4b5899bafbe020af2
|\ \  Merge: 72cca80 9168e5f
| |/  Author: yjp's Ubuntu16 <yjp@hust.edu.cn>
| |     Merge branch 'test2' into test1
```

为缩短长度，`...`部分是省略了的，28eead分支的信息没做省略。9b531e处test1和test2合并过，可以认为之前两者都一样，也可以认为test2从这里刚从test1中分出。

> test1的commit：new1
>
> test2的commit：edit5 edit6

从日志可以看到左侧有两个`| |`的线，第一根代表test1分支，第二根代表其他和test1分支有交互（合并、分离等操作）的分支，这个只有test2合并过，其他分支可能情况复杂很多，并且多根线时只知道第一根是git log后面跟的哪个分支，其他分支是哪个分支只能通过Author分辨了，如果Author还一样就只能根据日志猜了。

`| |`中间的`*`代表了跟`*`同行的那个commit ID 是提交到哪个分支上的。

`\ /`斜杠代表了合并和新分支。

直接用git log，不加--graph参数是没有这个图形化的东西的，只有一个个commit，根本无法分辨分支情况，没有git log branchx中的branchx会很多个分支杂在一起，也不好分辨。

## git add：添加修改

常用:

```bash
git add -u
git add .
git add $filename # filename是要跟踪的文件名
```

## git commit：提交

常用：

```bash
git commit -m $str # str是一提交的日志消息，记得加双引号
```

## git branch：建立新分支

常用：

```bash
git branch -a	# 查看本地和远端所有分支
git branch $branchx # 新建一个名为branchx的分支
git branch -d $branchx	# 删除分支branchx
git branch -D $branchx	# 强制删除分支branchx，该分支没有clean或没合并到其他分支时，想删除需要-D
```

## git checkout：切换分支

常用：

```bash
git checkout $branchx		# 切换到branchx分支
git checkout -b $branchx	# 新建branchx分支并切换过去
```

## git merge：合并分支

常用：

``` bash
# 合并branchx分支到当前分支，自动选择ff或no-ff，默认ff
git merge $branchx
# 使用fast forward模式合并branchx分支到当前分支，这个模式下合并后branchx下的所有commit日志都会合并进来，并和本分支原本的日志根据commit时间前后排序，但是合并不会产生新的commit
git merge --ff $branchx
# 使用no fast forward模式合并branchx分支到当前分支，合并会产生新的commit，其他同ff模式
git merge --no-ff $branchx
# 合并branchx分支到当前分支，但是不提交，也就是全部以git add前的形式存在，需要自己修改冲突、add、commit，不推荐，因为所有文件都在，不知道哪个有冲突，要一个个查看，很麻烦。
# 这个参数主要是为了舍弃branchx分支中没有意思的分支，保持日志的干净整洁，但是比起这条命令，更推荐先使用git rebase整理好分支branchx再git merge合并
git merge --squash $branchx 
git merge --abort	# 放弃本次合并
```

合并时如果出现冲突需要自行手动合并，冲突文件会在git status中显示为未add的文件，打开冲突文件，其中冲突部分有如下形式：

```
<<<<<<< $commitID1
...
=======
...
>>>>>>> $commitID2
```

合并是从commitID2合并进commitID1，即在commitID1所在分支下使用`git merge commitID2`命令，上半部`...`是commitID1中的代码，下半部`...`是commitID2中的代码，不需要的一部分`...`和`<<<`,`===`,`>>>`都删了然后保存退出即可。

> 直接在中断使用vim或者gedit改就行，其实挺方便的，一般冲突也不多，同一个文件不同行修改都不会产生冲突，只有同行修改才会冲突
>
> commitID[12]名字一般是合并的两个分支的名字，也有可能是HEAD、commitID等。
>
> HEAD等详见名词解释一节。

合并时也可能出现`git rebase`中那种界面，操作方式类似记事本，而不是vim，下面有按键提示，修改日志消息后`Ctrl-X`退出，然后输入大写`Y`，剩下的按回车就行。

## git reset：版本回退

常用：

``` bash
# --hard会丢弃所有修改，即所有写了的代码没commit的全丢了，不想丢可以git stash或者改用其他参数
git reset --hard HEAD # 回退到HEAD指针的位置
git reset --hard HEAD~n # 回退到HEAD指针往前数n个commit的位置
git reset --hard $commitID # 回退到commitID所代表的版本出，commitID可以通过git log查到
```

--hard参数会强制丢掉所有未提交的内容，不想丢掉的话可以使用soft或者mixed参数，mixed是默认参数。

## git rebase：压缩多次commit

常用：

``` bash
git rebase -i HEAD~n # 修改HEAD往前数n个commit，这n个commit的日志
git rebase --abort
git rebase --continue
```

使用`git rebase -i HEAD~n`开始压缩commit，使用该命令后出现如下界面：

```bash
pick ce23814 edit3
pick 5e39047 edit4 edit5
pick fc12f4a edit6
pick 0ab9c57 edit8

# Rebase cdda324..0ab9c57 onto cdda324 (4 command(s))
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
^G Get Help  ^O Write Out ^W Where Is  ^K Cut Text  ^J Justify   ^C Cur Pos
^X Exit      ^R Read File ^\ Replace   ^U Uncut Text^T To Spell  ^_ Go To Line

```

这里是对test2分支进行压缩，之前测试已经压过一次，所以少了edit7，合并了edit4和5。

merge合并的分支日志不会显示，除非是用no-ff建立新节点。

下面的注释已经提示的很清楚了：

- pick：保持当前commit不变

- reword：修改当前日志的信息，节点不变

- edit：嘛玩意，有生词，也没用到，先不写了

- squash：**常用**，将这个commit加到上面一个commit中，即两个commit合并，并且保留两个的日志信息，会出现一个新的界面让你重新编写日志信息，编写玩按下面的提示ctrl+？（不记得了）保存即可。

- fixup：**常用**，类似squash，但是这个commit的日志信息不要了直接丢弃，并合并两个commit

- 后俩没用过不说了

  > 其实也就用过pick reword squash fixup四个，后两个合并时是跟上一个合并，也就是跟时间轴上时间早的一个合并，例如edit5设为squash后和edit4合并了。

改完之后，根据下面的提醒`^X Exit`，按`Ctrl-X`退出，如果设置了squash，则修改日志消息（有#的行不会显示，不用管）然后按照下面提示退出，然后输入大写`Y`表示yes，剩下的按回车就行。

强制退出：`Ctrl Z`

问题：

- 只能在本地这么做，如果对push到远程仓库的分支进行压缩，再提交时会出现冲突，因为前面的commit日志对不上了，只能使用git push --force强制把远程的分支整个改成现在的，风险比较大，会使之前的分支全部丢失。
- 所以一定要本地rebase完才能push，push后的部分不能rebase了。
- 合并后合并的commit就永久丢失了，所以要慎用，并且留下的commit合并到其他分支时也会合并过去，这点不如`git merge --squash`整洁。

## git clone：从远程分支获取仓库

常用：

```bash
git clone x	# x为远程仓库网址，别忘了带双引号，从远程仓库获取一个代码仓库，会clone下来master分支，想要其他分支需要使用git fetch命令
```

## git fetch：从远程分支获取分支

常用：

``` bash
git fetch origin branchx	# 从远程仓库origin获取分支branchx
```

## git push：提交到远程服务器

常用：

```bash
git push origin branchx:branchy	# 将本地的分支branchx上传到远程分支origin上并命名为branchy，origin和branch[xy]的解释同git fetch
git push origin branchx	# 默认上传本地分支到远程同名分支
```

## git pull：从远程服务器更新

``` bash
git push origin branchx	# 从远程仓库origin更新本地分支branchx，在git fetch后使用，用于更新分支，如果发生冲突需要手动合并，origin和branchx的解释同git fetch
```





