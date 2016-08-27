---
title: 聊聊用 Git 协同合作的事儿
date: 2016-08-27 09:12:02
tags:
- Git
- Teamwork
---

说起来在来到公司之前一直没有好好用 Git 来管理分支。之前做一些课程设计和玩具项目的时候虽然有用到 Git，但是也仅限于本地仓库的提交而已。这样做更多的只是留下一个可供追寻的路径（ History ），没有太多的“管理”职能。在多人协同合作的项目中，Git 终于发挥出它强大的优势。

<!-- more -->

所以我猜想，是不是有不少像之前的我那样子的学生/个人开发者，一直只是把 Git 当做一个跟踪工具来用（偶尔出现懒得修改的问题，直接回滚，真是好厉害好方便的样子）。
这篇文章会分享一些我们团队目前接触到的 Git 配置和使用方式（希望正在看文章的你已经对 Git 的指令有一些基础认识了），内容会比较分散，但基本上是围绕着我们现在的开发过程来描述的。对于真•程序员来说，这些方法应该是烂熟于胸了（虽然有的方法我是最近才知道...），不过也希望可以起到一定的参考作用。因为团队还比较小，很多东西还在探索阶段，所以难免会有疏漏和考虑不周的地方，欢迎能看到各位一起交流探讨~

## 配置工作环境
### 需求
现有的工作环境&流程大概是这样的：
1. **upstream** 指向服务器上的公共代码区，里面一般分为发布分支 *master* 和开发分支 *develop*。
2. **origin** 是从 **upstream** 上 `fork` 出来的，为了跟 **upstream** 上的 *develop* 作区分，一般会从 *origin/develop* 上 `checkout` 一个*姓名-develop*分支出来作为自己的开发分支。
3. 每次推代码之前都要从 *upstream/develop* 上 `fetch` 最新的代码，并在本地进行合并和冲突的解决。解决完之后先 `push` 到自己的 *origin/姓名-develop* 分支上，然后向主仓库的对应分支提交 Merge Request。

为了满足上面的需求，配好之后本地上看到的大概是这个样子：
![demand](/uploads/team-working-with-git/demand.png)￼

### 动手
公司的代码托管服务用的是 GitLab，操作起来跟 Github 基本一致，想要了解的可以上它[官网](https://about.gitlab.com/)看看。

`fork` 那一步可以直接在 GitLab 上面操作，点几下鼠标就好了。主要是在本地配置 **upstream** 这一步。
把 `fork` 过来的自己的项目 `clone` 下来之后，就相当于有了一整个 **origin** 的代码，切分支什么的都好办了，问题是要这样子拉最新代码：

```bash
$ git fetch upstream
```

这就需要先执行以下的指令，把远程分支的信息拉到本地：

```bash
$ git remote add upstream <分支URL>
```

## 好用的 stash
在改了一大堆零碎的东西，却又还没有改完的情况下，如果需要回滚到这次修改之前有两种做法：

1. commit 当前所有修改，然后 reset 到上一个 commit
2. stash 当前所有修改，用完再 stash pop 回来

个人比较喜欢用第二种，虽然也有用第一种方法，然后 commit 的 comment 是 “Just a save before huge change” 的时候。

> 更好的做法应该是：把要做的任务切分得更小一些，多进行提交。这样不但可以更好的从 log 中看到开发的进程，也可以更好地管理每一个小任务所对应的代码。
> 也可以更好地应对产品经理各种功能调整的需求了 :）

### 选择性 stash
`git stash --patch` 和 `git stash -p` 让你可以选择需要 stash 的条目/文件，执行这条指令之后，命令行会逐一过一遍所有修改块。

注意是**所有修改块**，用 y/n 可以选择是否要 stash 当前的代码块，如果想要过掉当前的文件的话，也可以用 a/d 来选择是否要 stash 当前文件。

> 每一次 stash 之后，Git 都会把 stash 掉的东西保存为一次记录，有点类似于一次没有描述的提交。不同的是，stash 不会出现在历史记录里面，在提供了灵活性的同时，也有一定的风险（鬼知道代码经历了什么）

## 远程分支的一些事
配好了环境，也在本地鼓捣了一番，也该看看怎么对远程分支搞破坏了。

### 远程分支追踪
在本地分支与远程分支不同步时，有时候会想要干掉本地分支，再重新把远程分支拉一个下来。

> 这个例子比较硬...难道不应该看看为什么不同步吗...？

做这个操作之前，首先要确保本地已经具备远程的所有信息：

```bash
$ git fetch
```

然后直接用 `checkout` 创建分支，并指向远程分支就好了，比如把远程 develop 分支拉下来，对应上本地的 develop 分支：

```bash
$ git checkout -b develop origin/develop
```

接着你可以这样查看到分支的关系：

```bash
$ git branch -vv
```

会看到这样的东西：
![branchvv](/uploads/team-working-with-git/branchvv.png)￼

或者这样看：

```bash
$ git remote show origin
```

结果长这个样子：
![remote_show](/uploads/team-working-with-git/remote_show.png)￼

### 如果远程不认识我...
进行了一系列分支调整之后，通过 `git remote show origin` 的指令可能会发现缺少了某些分支，这时候就需要手动去配置一下分支的关系。

分支的相关配置都保存在 `.git/config` 这个目录下，直接 `cat` 一下可以看到所有分支的参数设置，其中一条的参数大概长这个样子：
![branch_info](/uploads/team-working-with-git/branch_info.png)￼

这个地方是在告诉 Git，本地的 liyiran-develop 分支对应的是 origin 上的 liyiran-develop 分支，也就是说**本地** liyiran-develop 的所有 *不加参数* 的 push 和 pull 操作都应该对应在**远程** liyiran-develop。如果没有想要配置的那个分支，直接按照这个格式手动加上去就好了。

或者，你可以通过 git 指令来操作：

```bash
$ git config branch.master.remote origin  // 将 remote 参数的内容设置为 origin
$ git config branch.master.merge refs/heads/master // 将 merge 参数的内容设置为 refs/heads/master
```

## 结语
这边其实也只是用到了一些皮毛，还有像 `cherry-pick`、`rebase` 这类大杀器都没有涉及到，希望能起到一个抛砖引玉的作用。
