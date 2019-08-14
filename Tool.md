---

typora-copy-images-to: ..\NoteBook-img
---

 [Table of Contents](#table-of-contents)
 [git](#git)

 * [git安装](#git%E5%AE%89%E8%A3%85)
   * [SSH 传输设置](#ssh-%E4%BC%A0%E8%BE%93%E8%AE%BE%E7%BD%AE)
 * [git 简介](#git-%E7%AE%80%E4%BB%8B)
 * [git使用](#git%E4%BD%BF%E7%94%A8)
   * [工作流](#%E5%B7%A5%E4%BD%9C%E6%B5%81)
   * [分支简介](#%E5%88%86%E6%94%AF%E7%AE%80%E4%BB%8B)
   * [分支合并策略](#%E5%88%86%E6%94%AF%E5%90%88%E5%B9%B6%E7%AD%96%E7%95%A5)
   * [git冲突](#git%E5%86%B2%E7%AA%81)
   * [分支推送](#%E5%88%86%E6%94%AF%E6%8E%A8%E9%80%81)
   * [储藏（Stashing）](#%E5%82%A8%E8%97%8Fstashing)
   * [命令理解：](#%E5%91%BD%E4%BB%A4%E7%90%86%E8%A7%A3)
   * [Git 命令一览](#git-%E5%91%BD%E4%BB%A4%E4%B8%80%E8%A7%88)
   * [reference：](#reference)
   * [小技能](#%E5%B0%8F%E6%8A%80%E8%83%BD)
     * [场景一[picture]](#%E5%9C%BA%E6%99%AF%E4%B8%80picture)
     * [场景二[TOC]](#%E5%9C%BA%E6%99%AF%E4%BA%8Ctoc)
* [github](#github)
* [draw\.io](#drawio)

# git

​		在windows环境下模拟linux环境



## git安装

### SSH 传输设置

Git 仓库和 Github 中心仓库之间的传输是通过 SSH 加密。

如果工作区下没有 .ssh 目录，或者该目录下没有 id_rsa 和 id_rsa.pub 这两个文件，可以通过以下命令来创建 SSH Key：

```
$ ssh-keygen -t rsa -C "youremail@example.com"
```

然后把公钥 id_rsa.pub 的内容复制到 Github "Account settings" 的 SSH Keys 中。

reference：<https://blog.csdn.net/qq_37512323/article/details/80693445>

## git 简介

Git是目前世界上最先进的分布式版本控制系统，可以帮助记录每次文件的改动，还可以让同事协作编辑，方便文件改动的记录和恢复

仓库，英文名**repository**，你可以简单理解成一个目录，这个目录里面的所有文件都可以被Git管理起来，每个文件的修改、删除，Git都能跟踪，以便任何时刻都可以追踪历史，或者在将来某个时刻可以“还原”。

目录下有一个`.git`的目录，这个开始见哦git版本库，这个目录可以用来跟踪管理版本库，没事千万不要手动修改这个目录里面的文件，不然改乱了，就把Git仓库给破坏了。

如果你没有看到`.git`目录，那是因为这个目录默认是隐藏的，用`ls -ah`命令就可以看见。



* git是分布式版本控制系统，开发人员在本地clone一个仓库，无需连接中心服务器即可完成代码的提交和修改
* 比方说你在自己电脑上改了文件A，你的同事也在他的电脑上改了文件A，这时，你们俩之间只需把各自的修改推送给对方，就可以互相看到对方的修改了
* 在实际使用分布式版本控制系统的时候，其实很少在两人之间的电脑上推送版本库的修改，因为可能你们俩不在一个局域网内，两台电脑互相访问不了，也可能今天你的同事病了，他的电脑压根没有开机。因此，分布式版本控制系统通常也有一台充当“中央服务器”的电脑，但这个服务器的作用仅仅是用来方便“交换”大家的修改，没有它大家也一样干活，只是交换修改不方便而已
* svn是集中式版本控制系统，开发人员需要连接到中心服务器上，才可以完成代码的提交和修改

## git使用

### 工作流

本地仓库与远程仓库的工作流：

![1565615025778](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565615025778.png)



新建一个仓库之后，当前目录就成为了工作区，工作区下有一个隐藏目录 .git，它就是 Git 的版本库。

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



`git clone`    将远程仓库clone到本地

当你从远程仓库克隆时，实际上Git自动把本地的`master`分支和远程的`master`分支对应起来了，并且，远程仓库的默认名称是`origin`。

```
$ git clone git@github.com:michaelliao/gitskills.git
```





Git 的版本库有一个称为 Stage 的暂存区以及最后的 History 版本库，History 存储所有分支信息，使用一个 HEAD 指针指向当前分支。

![1565598791056](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565598791056.png)

- git add files 把文件的修改添加到暂存区
- git commit 把暂存区的修改提交到当前分支，提交之后暂存区就被清空了
- git reset -- files 使用当前分支上的修改覆盖暂存区，用来撤销最后一次 git add files
- git checkout -- files 使用暂存区的修改覆盖工作目录，用来撤销本地修改

![1565598907169](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565598842310.png)



```shell
$ git add file1.txt
$ git add file2.txt file3.txt
$ git commit -m "add 3 files."
#commit一次性提交多个文件
```

可以跳过暂存区域直接从分支中取出修改，或者直接提交修改到分支中。

- git commit -a 直接把所有文件的修改添加到暂存区然后执行提交
- git checkout HEAD -- files 取出最后一次修改，可以用来进行回滚操作

![1565598890690](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565598890690.png)

### 分支简介

使用指针将每个提交连接成一条时间线，HEAD 指针指向当前分支指针。

![1565599137184](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565599137184.png)



新建分支是新建一个指针指向时间线的最后一个节点，并让 HEAD 指针指向新分支，表示新分支成为当前分支。

![1565599154337](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565599154337.png)



每次提交只会让当前分支指针向前移动，而其它分支指针不会移动。

![1565599174391](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565599174391.png)



合并分支也只需要改变指针即可。

![1565599192208](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565599192208.png)



**note：**

​		时间线是全局的，多个分支共享同一个时间线，停在自己指定的位置

​		head指针只有一个，用来表明当前操作的分支



### 分支合并策略

1. fast-foward模式

![1565682796969](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565682796969.png)



**上图所示的情况可以完成快速合并**

2. 禁用fast-foward模式

**如果上图禁用fast_foward模式呢**

```shell
git merge --no-ff -m "merge with no-ff" 分支名
```

```
$ git log --graph --pretty=oneline --abbrev-commit
*   e1e9c68 (HEAD -> master) merge with no-ff
|\  
| * f52c633 (dev) add merge
|/  
*   cf810e4 conflict fixed
...
```

图示如下：

![1565683100413](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565683095797.png)



3. 分支策略

在实际开发中，我们应该按照几个基本原则进行分支管理：

首先，`master`分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在`dev`分支上，也就是说，`dev`分支是不稳定的，到某个时候，比如1.0版本发布时，再把`dev`分支合并到`master`上，在`master`分支发布1.0版本；

你和你的小伙伴们每个人都在`dev`分支上干活，每个人都有自己的分支，时不时地往`dev`分支上合并就可以了。

所以，团队合作的分支看起来就像这样：

![1565683365947](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565683365947.png)



> 合并分支时，加上`--no-ff`参数就可以用普通模式合并，合并后的历史有分支，能看出来曾经做过合并，而`fast forward`合并就看不出来曾经做过合并。





### git冲突

当两个分支都对**同一个文件的同一行**进行了修改，在分支合并时就会产生冲突。

![1565599354276](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565599354276.png)



Git 会使用 <<<<<<< ，======= ，>>>>>>> 标记出不同分支的内容，只需要把不同分支中冲突部分修改成一样就能解决冲突。

```shell
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```

当Git无法自动合并分支时，就必须首先解决冲突。解决冲突后，再提交，合并完成。

解决冲突就是把Git合并失败的文件手动编辑为我们希望的内容，再提交。

用`git log --graph`命令可以看到分支合并图。



### 分支推送

1. 简介

`git remote`    查看远程库的信息

`git remote -v`    显示更详细的信息

`git push origin master`    把该分支上的所有本地提交推送到远程库。推送时，要指定本地分支，这样，Git就会把该分支推送到远程库对应的远程分支上

但是，并不是一定要把本地分支往远程推送，那么，哪些分支需要推送，哪些不需要呢？

- `master`分支是主分支，因此要时刻与远程同步；
- `dev`分支是开发分支，团队所有成员都需要在上面工作，所以也需要与远程同步；
- bug分支只用于在本地修复bug，就没必要推到远程了，除非老板要看看你每周到底修复了几个bug；
- feature分支是否推到远程，取决于你是否和你的小伙伴合作在上面开发。



当你的小伙伴从远程库clone时，默认情况下，你的小伙伴只能看到本地的`master`分支。不信可以用`git branch`命令看看：

```shell
$ git branch
* master
```

现在，你的小伙伴要在`dev`分支上开发，就必须创建远程`origin`的`dev`分支到本地，于是他用这个命令创建本地`dev`分支,本地和远程分支的名称最好一致

```shell
$ git checkout -b dev origin/dev
```

现在，他就可以在`dev`上继续修改，然后，时不时地把`dev`分支`push`到远程



2. 推送失败场景

你的小伙伴已经向`origin/dev`分支推送了他的提交，而碰巧你也对同样的文件作了修改，并试图推送：

因为你的小伙伴的最新提交和你试图推送的提交有冲突，解决办法也很简单，Git已经提示我们，先用`git pull`把最新的提交从`origin/dev`抓下来，然后，在本地合并，解决冲突，再推送

```shell
$ git pull
There is no tracking information for the current branch.
Please specify which branch you want to merge with.
See git-pull(1) for details.

    git pull <remote> <branch>

If you wish to set tracking information for this branch you can do so with:

    git branch --set-upstream-to=origin/<branch> dev
```

`git pull`也失败了，原因是没有指定本地`dev`分支与远程`origin/dev`分支的链接，根据提示，设置`dev`和`origin/dev`的链接：

```shell
$ git branch --set-upstream-to=origin/dev dev
Branch 'dev' set up to track remote branch 'dev' from 'origin'.
```

再pull：

```shell
$ git pull
Auto-merging env.txt
CONFLICT (add/add): Merge conflict in env.txt
Automatic merge failed; fix conflicts and then commit the result.
```

这回`git pull`成功，但是合并有冲突，需要手动解决，解决的方法和分支管理中的[解决冲突](http://www.liaoxuefeng.com/wiki/896043488029600/900004111093344)完全一样。解决后，提交，再push：

```shell
$ git commit -m "fix env conflict"
[dev 57c53ab] fix env conflict

$ git push origin dev
Counting objects: 6, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 621 bytes | 621.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0)
To github.com:michaelliao/learngit.git
   7a5e5dd..57c53ab  dev -> dev
```



**总结：**

因此，多人协作的工作模式通常是这样：

1. 首先，可以试图用`git push origin <branch-name>`推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！

如果`git pull`提示`no tracking information`，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to <branch-name> origin/<branch-name>`。

这就是多人协作的工作模式，一旦熟悉了，就非常简单。



### 储藏（Stashing）

在一个分支上操作之后，如果还没有将修改提交到分支上，此时进行切换分支，那么另一个分支上也能看到新的修改。这是因为所有分支都共用一个工作区的缘故。

可以使用 git stash 将当前分支的修改储藏起来，此时当前工作区的所有修改都会被存到栈中，也就是说当前工作区是干净的，没有任何未提交的修改。此时就可以安全的切换到其它分支上了。

```shell
$ git stash
Saved working directory and index state \ "WIP on master: 049d078 added the index file"
HEAD is now at 049d078 added the index file (To restore them type "git stash apply")

。。。

切换回dev分支
$ git checkout dev
#用于查看git吧stash内容存放的位置
$ git stash list
#进行stash恢复
$ git stash pop
```

`git stash apply`        恢复后，stash内容并不删除，你需要用`git stash drop`来删除；

`git stash pop`，恢复的同时把stash内容也删了



如：

​		当你接到一个修复一个代号101的bug的任务时，很自然地，你想创建一个分支`issue-101`来修复它，但是，等等，当前正在`dev`上进行的工作还没有提交，并不是你不想提交，而是工作只进行到一半，还没法提交，预计完成还需1天时间。但是，必须在两个小时内修复该bug，怎么办？

​		此时你就可以使用git stash功能来储藏工作现场



你可以多次stash，恢复的时候，先用`git stash list`查看，然后恢复指定的stash，用命令：

```shell
$ git stash apply stash@{0}
```



### 命令理解：

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

`git checkout -b `     创建并切换分支

`git branch`    命令查看当前分支

`git checkout`    切换分支

`git merge`    合并指定分支到当前分支

`git branch -d`    删除分支

**因为创建、合并和删除分支非常快，所以Git鼓励你使用分支完成某个任务，合并后再删掉分支，这和直接在`master`分支上工作效果是一样的，但过程更安全。**

`git branch -D <name>`    如果要丢弃一个没有被合并过的分支，可以通过`git branch -D <name>`强行删除

`git remote -v `   查看远程库信息



### Git 命令一览

![img](https://raw.githubusercontent.com/CyC2018/CS-Notes/master/notes/pics/7a29acce-f243-4914-9f00-f2988c528412.jpg)

比较详细的地址：http://www.cheat-sheets.org/saved-copy/git-cheat-sheet.pdf



### reference：

- [Git - 简明指南](http://rogerdudler.github.io/git-guide/index.zh.html)
- [图解 Git](http://marklodato.github.io/visual-git-guide/index-zh-cn.html)
- [廖雪峰 : Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)（使用过程即描述非常清晰）
- [Learn Git Branching](https://learngitbranching.js.org/)
- https://github.com/huster-mr/CS-Notes



### 小技能

#### 场景一[picture]

如果在本地仓库创建markdown文件，且文档中包含图片，同步到远程仓库中，远程仓库中的图片显示异常

​		解决方案：

1. 上传本地图片至github中

     		2. 点击github中图片的下载，将其连接复制到markdown本地文件中，如：https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565615025778.png
       		3. 更新markdown文档中的全部图片连接后，再次上传即可正常显示图片



####  场景二[TOC]

windows : github中不支持markdown 自动生成目录TOC

​		解决方案：

1. 下载gh-md-toc  https://github.com/ekalinin/github-markdown-toc.go/releases
2. 解压后，暴力将gh-md-toc的后缀更改为.exe
3. 将需要添加目录的md文件放至和gh-md-toc.exe同一目录下
4. 光标移至exe上，按住shift键，同时右击

![1565701016926](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565701016926.png)

5. 将exe拖拽进该cmd（防止出现闪退现象） 

![1565701073481](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565701073481.png)



![1565701387430](https://raw.githubusercontent.com/changyan17/NoteBook/master/pictures/1565701387430.png)





6. 生成Tool.md的目录代码（GFM格式），将其粘贴至github的文件中即可
7. 右键cmd菜单栏  ->  编辑  ->  标记  ->  选中需要复制的内容  ->  右键cmd菜单栏  ->  编辑  ->  复制即可



#### 场景三[gitignore]

如果在git仓库中包含了某些不能或者不想提交的文件，每次使用git  status时都会显示这些文件untracked files,此时可以通过配置.gitignore文件来进行忽略



不需要从头写`.gitignore`文件，GitHub已经为我们准备了各种配置文件，只需要组合一下就可以使用了。所有配置文件可以直接在线浏览：https://github.com/github/gitignore



**操作步骤：**

1. 在工作目录下创建.gitignore文件
2. 在该文件中添加想要忽略的文件信息



**忽略文件的原则是：**

1. 忽略操作系统自动生成的文件，比如缩略图等；
2. 忽略编译生成的中间文件、可执行文件等，也就是如果一个文件是通过另一个文件自动生成的，那自动生成的文件就没必要放进版本库，比如Java编译产生的`.class`文件；
3. 忽略你自己的带有敏感信息的配置文件，比如存放口令的配置文件。



# github

github相当于是一个代码仓库，一般一个项目对应一个仓库

* fork: B fork A的项目，相当于复制A的项目到自己的 github下
* pull request：B更新了项目，pull request A，等待A查看，A确认后，可以合并B的更新
* watch：关注项目，当项目更新可以接收到通知
* issue：讨论问题



# draw.io

* **打开draw.io之后，可以搜索图形，选中拖拽至画图区**

![1565748724306](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565748724306.png)

![1565748749613](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565748749613.png)





* **可以通过上方的菜单栏选择sql生成或者其他方式，直接利用代码插入图表等**

![1565748880398](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565748880398.png)



![1565748898079](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565748898079.png)



![1565748930910](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1565748930910.png)



* 操作本地图片时，可以直接将本地图片拖拽至工作区即可