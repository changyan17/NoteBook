# 工具使用

## git

​		在windows环境下模拟linux环境

### git安装

#### SSH 传输设置

Git 仓库和 Github 中心仓库之间的传输是通过 SSH 加密。

如果工作区下没有 .ssh 目录，或者该目录下没有 id_rsa 和 id_rsa.pub 这两个文件，可以通过以下命令来创建 SSH Key：

```
$ ssh-keygen -t rsa -C "youremail@example.com"
```

然后把公钥 id_rsa.pub 的内容复制到 Github "Account settings" 的 SSH Keys 中。

reference：<https://blog.csdn.net/qq_37512323/article/details/80693445>

### git 简介

Git是目前世界上最先进的分布式版本控制系统，可以帮助记录每次文件的改动，还可以让同事协作编辑，方便文件改动的记录和恢复

仓库，英文名**repository**，你可以简单理解成一个目录，这个目录里面的所有文件都可以被Git管理起来，每个文件的修改、删除，Git都能跟踪，以便任何时刻都可以追踪历史，或者在将来某个时刻可以“还原”。

目录下有一个`.git`的目录，这个开始见哦git版本库，这个目录可以用来跟踪管理版本库，没事千万不要手动修改这个目录里面的文件，不然改乱了，就把Git仓库给破坏了。

如果你没有看到`.git`目录，那是因为这个目录默认是隐藏的，用`ls -ah`命令就可以看见。



* git是分布式版本控制系统，开发人员在本地clone一个仓库，无需连接中心服务器即可完成代码的提交和修改
* 比方说你在自己电脑上改了文件A，你的同事也在他的电脑上改了文件A，这时，你们俩之间只需把各自的修改推送给对方，就可以互相看到对方的修改了
* 在实际使用分布式版本控制系统的时候，其实很少在两人之间的电脑上推送版本库的修改，因为可能你们俩不在一个局域网内，两台电脑互相访问不了，也可能今天你的同事病了，他的电脑压根没有开机。因此，分布式版本控制系统通常也有一台充当“中央服务器”的电脑，但这个服务器的作用仅仅是用来方便“交换”大家的修改，没有它大家也一样干活，只是交换修改不方便而已
* svn是集中式版本控制系统，开发人员需要连接到中心服务器上，才可以完成代码的提交和修改

### git使用

#### 工作流

本地仓库与远程仓库的工作流：

![1565615025778](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565615025778.png)



新建一个仓库之后，当前目录就成为了工作区，工作区下有一个隐藏目录 .git，它就是 Git 的版本库。

Git 的版本库有一个称为 Stage 的暂存区以及最后的 History 版本库，History 存储所有分支信息，使用一个 HEAD 指针指向当前分支。

![1565598791056](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565598791056.png)

- git add files 把文件的修改添加到暂存区
- git commit 把暂存区的修改提交到当前分支，提交之后暂存区就被清空了
- git reset -- files 使用当前分支上的修改覆盖暂存区，用来撤销最后一次 git add files
- git checkout -- files 使用暂存区的修改覆盖工作目录，用来撤销本地修改

![1565598907169](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565598907169.png)

```shell
$ git add file1.txt
$ git add file2.txt file3.txt
$ git commit -m "add 3 files."
#commit一次性提交多个文件
```

可以跳过暂存区域直接从分支中取出修改，或者直接提交修改到分支中。

- git commit -a 直接把所有文件的修改添加到暂存区然后执行提交
- git checkout HEAD -- files 取出最后一次修改，可以用来进行回滚操作

![1565598890690](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565598890690.png)



#### 分支

使用指针将每个提交连接成一条时间线，HEAD 指针指向当前分支指针。

![1565599137184](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565599137184.png)

新建分支是新建一个指针指向时间线的最后一个节点，并让 HEAD 指针指向新分支，表示新分支成为当前分支。

![1565599154337](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565599154337.png)

每次提交只会让当前分支指针向前移动，而其它分支指针不会移动。

![1565599174391](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565599174391.png)

合并分支也只需要改变指针即可。

![1565599192208](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565599192208.png)

**note：**

​		时间线是全局的，多个分支共享同一个时间线，停在自己指定的位置

​		head指针只有一个，用来表明当前操作的分支

#### 冲突

当两个分支都对**同一个文件的同一行**进行了修改，在分支合并时就会产生冲突。

![1565599354276](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565599354276.png)

Git 会使用 <<<<<<< ，======= ，>>>>>>> 标记出不同分支的内容，只需要把不同分支中冲突部分修改成一样就能解决冲突。

```shell
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```



#### 储藏（Stashing）

在一个分支上操作之后，如果还没有将修改提交到分支上，此时进行切换分支，那么另一个分支上也能看到新的修改。这是因为所有分支都共用一个工作区的缘故。

可以使用 git stash 将当前分支的修改储藏起来，此时当前工作区的所有修改都会被存到栈中，也就是说当前工作区是干净的，没有任何未提交的修改。此时就可以安全的切换到其它分支上了。

```
$ git stash
Saved working directory and index state \ "WIP on master: 049d078 added the index file"
HEAD is now at 049d078 added the index file (To restore them type "git stash apply")
```

该功能可以用于 bug 分支的实现。如果当前正在 dev 分支上进行开发，但是此时 master 上有个 bug 需要修复，但是 dev 分支上的开发还未完成，不想立即提交。在新建 bug 分支并切换到 bug 分支之前就需要使用 git stash 将 dev 分支的未提交修改储藏起来。



#### Git 命令一览

![img](https://raw.githubusercontent.com/CyC2018/CS-Notes/master/notes/pics/7a29acce-f243-4914-9f00-f2988c528412.jpg)

比较详细的地址：http://www.cheat-sheets.org/saved-copy/git-cheat-sheet.pdf

#### 命令理解：

`git init`    初始化一个Git仓库

`git add <file>`  可反复多次使用，添加多个文件  添加文件到暂存区

`git commit -m <message> `   添加文件到仓库的当前分支

`git status `   要随时掌握工作区的状态

`git diff`    如果`git status`告诉你有文件被修改过，用其可以查看修改内容

`git diff HEAD -- readme.txt  `    查看工作区和版本库里面最新版本的区别

`git log`    命令显示从最近到最远的提交日志,在Git中，用`HEAD`表示当前版本

```
$ git log --pretty=oneline
1094adb7b9b3807259d8cb349e7df1d4d6477073 (HEAD -> master) append GPL  //前面的数字是commit id （版本号）
e475afc93c209a690c39c13a46716e8fa000c366 add distributed
eaadf4e385e865d25c48e7ca9c8395c3f7dfaef0 wrote a readme file
```

`git reset --hard HEAD^`    上一个版本就是`HEAD^`,往上100个版本写100个`^`比较容易数不过来，所以写成`HEAD~100`  该命令在更新head指针的同时，同步更新仓库中的文件

```shell
#只要上面的命令行窗口还没有被关掉，你就可以顺着往上找啊找啊，找到那个append GPL的commit id是1094adb...，于是就可以指定回到未来的某个版本
#从A版本回退到B版本后，再由B版本前进到A版本的状态
$ git reset --hard 1094a

#Git的版本回退速度非常快，因为Git在内部有个指向当前版本的HEAD指针，当你回退版本的时候，Git仅仅是把HEAD从指向append GPL
```

`git reflog`    git reflog

> - `HEAD`指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令`git reset --hard commit_id`。
> - 穿梭前，用`git log`可以查看提交历史，以便确定要回退到哪个版本。
> - 要重返未来，用`git reflog`查看命令历史，以便确定要回到未来的哪个版本

`git checkout -- readme.txt`    把`readme.txt`文件在工作区的修改全部撤销

> 一种是`readme.txt`自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；
>
> 一种是`readme.txt`已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。
>
> 总之，就是让这个文件回到最近一次`git commit`或`git add`时的状态。
>
> **note:**     `git checkout -- file`命令中的`--`很重要，没有`--`，就变成了“切换到另一个分支”的命令，我们在后面的分支管理中会再次遇到`git checkout`命令。



> 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`。
>
> 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD <file>`，就回到了场景1，第二步按场景1操作。
>
> 场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考[版本回退](https://www.liaoxuefeng.com/wiki/896043488029600/897013573512192)一节，不过前提是没有推送到远程库。



 **注意：从来没有被添加到版本库就被删除的文件，是无法恢复的！**



`git rm`    从版本库中删除该文件



如果本地仓库有内容，远程github中的仓库没有内容，需要

1. 关联本地仓库与远程仓库

```shell
git remote add origin git@github.com:michaelliao/learngit.git
#远程库的名字就是origin，这是Git默认的叫法，也可以改成别的，但是origin这个名字一看就知道是远程库
```

2. 将本地仓库的内容推送到远程仓库

```shell
git push -u origin master
#实际上是把当前分支master推送到远程
#由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。
```

从现在起，只要本地作了提交，就可以通过命令：

```shell
$ git push origin master
```

lll



#### reference：

- [Git - 简明指南](http://rogerdudler.github.io/git-guide/index.zh.html)
- [图解 Git](http://marklodato.github.io/visual-git-guide/index-zh-cn.html)
- [廖雪峰 : Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)（使用过程即描述非常清晰）
- [Learn Git Branching](https://learngitbranching.js.org/)



## github

github相当于是一个代码仓库，一般一个项目对应一个仓库

* fork:BforkA的项目，相当于复制A的项目到自己的github下
* pull request：B更新了项目，pull request A，等待A查看，A确认后，可以合并B的更新
* watch：关注项目，当项目更新可以接收到通知
* issue：讨论问题
* 



## draw.io



