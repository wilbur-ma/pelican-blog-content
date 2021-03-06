Title: Git进阶教程
Date: 2013-08-25 12:14
Update: 2014-06-25 10:38
Tags: git, 教程

Git的常用命令和场景可参考[Git快速使用指南](/note/git-quick_reference.html)，在这里介绍进一步的使用和部分生僻的命令。

## Git概念和技巧
### Index vs. Working tree
使用git reset命令的时候不容易搞懂Index和Working tree这两个概念，下面简要总结一下。

![git index & working tree](/static/images/note/git-index-working-tree.png)

(图片取自stackoverflow上的一个[回答](http://stackoverflow.com/questions/3689838/difference-between-head-working-tree-index-in-git)，来源未知)

#### Git index
有时候也被称作Staging area和Cache，git add等命令会将文件改动暂存在index中(.git/index)，运行git status后，列在Changes to be committed后面的改动都位于index上。

Git index中的改动可以用git commit提交到本地仓库中。

#### Git working tree
也被称作Working directory，git add之前的文件改动都是在working tree中进行的，运行git status后，列在Changes not staged for commit和Untracked files后面的改动都位于working tree上。

### 提交范围

详细内容可参考[Commit Ranges](http://git-scm.com/book/ch6-1.html#Commit-Ranges)。

通常用于git log和git diff，主要有两种格式，以下面的提交历史为例:

    A - B - C - D - E - F

* double dot `C..E`: E能追溯到且C不能追溯到的提交

        D - E
    
    `git log C..E`等同于`git log ^C E`和`git log E --not C`

* triple dot `C...E`: E或C能追溯到且不能同时被E和C追溯到的提交，即`C xor E`

        D - E

以下面的提交历史为例:

    A - B - C - D - E - F
             \
              G - H - I

* `F..I`: G H I
* `F...I`: D E F G H I

另外，需要说明的是:

1. `git log`的选项`--not`和`^`符号可以构造比`..`更复杂的筛选条件。`--not`和`^`的区别在于`--not`对之后(下一个`--not`之前)的所有提交（包括`^`前缀的提交)都有效，而`^`仅对其之后的一个提交有效，例如

    * `--not F H --not I`: I
    * `--not F H ^I`: I
    * `^F H ^I`: 无
    * `^F F`: 无
    * `D H --not C`: D G H

2. 在`git log`中使用`...`时，使用`--left-right`选项可以直观的输出结果

        $ git log --left-right F...I
        > I
        > H
        > G
        < F
        < E
        < D


## Git进阶使用场景

并不常见的使用场景。
### git reset
#### git reset --soft
既没修改index也没修改working tree，只是将当前HEAD指针放到ORIG_HEAD里，然后将HEAD指向目标Commit(默认为HEAD)。

如果要撤销`git reset --soft`，只需要使用`git reflog`找到最近的一次提交，然后对其再做一次reset即可，比如`git reset --soft HEAD@{0}`。

#### git reset --mixed
重置index，不修改working tree。通常用于将git add的改动移出index，不会修改文件内容。

#### git reset --hard
重置index和working tree，完全恢复到目标comit(默认是HEAD)的状态。所有未commit或stash的改动都会**丢失**，使用此命令需多加小心。

### detached HEAD state

有时需要返回之前的提交状态，此时会处于“detached head“状态。

	mkdir git-test && cd $_ && git init  # 初始化一个新的git仓库
	touch a.txt && git add $_ && git commit -m "first commit"   # 添加文件a.txt并提交
	touch b.txt && git add $_ && git commit -m "second commit"  # 添加文件b.txt并提交
	git log --oneline  # 查看提交日志
	# 输出如下(##开头)
	## 8083b9e second commit
	## 15b7a79 first commit
	git branch         # 查看分支情况
	# 输出如下(##开头)
	## * master
	# 现在签出first commit
	git checkout 15b7a79
	# 输出如下(##开头)
	## Note: checking out '15b7a79'.
	## You are in 'detached HEAD' state.
	## ...

此时即处于detached head状态，当前的文件回到first commit后的样子。

	git branch
	# 输出如下所示(##开头)
	## * (no branch)
	## master

最后，可以用如下命令回到second commit状态。
    git checkout master

## Git生僻命令
### git fsck
检查git对象是否有效和能否被链接到。

使用`git fsck`的一个例子是，当不小心drop或clear了stash列表时，可以这样找回(代码取自stackoverflow上的一个[回答](http://stackoverflow.com/questions/89332/recover-dropped-stash-in-git)):

    for ref in `git fsck --unreachable | grep commit | cut -d' ' -f3`; do git show --summary $ref; done | less

`git fsck`的速度会比较慢，上面的命令会列出比较多的commit，因为`git stash`的默认提交信息都是`WIP on ...`，所以根据提交信息即可找到丢失的stash提交。

### git reflog
记录了HEAD指针的完整改动历史，通常用于于查找“丢失”的commit和跨分支查看提交历史。当因为某些操作导致某些commit“丢失”时，可以使用`git reflog`配合`git reset --hard`来恢复到之前的提交状态。

### git rev-list
按时间逆序列出提交对象，常用于查找涉及到某些文件的提交的hash。例如，查找所有关系到文件readme的提交：

    git rev-list HEAD -- readme

更多功能参考`man git-rev-list`。

### git rebase
rebase操作可以将两个分支上的差异合并到另一个分支上，和merge操作不同的是，rebase最后形成线性的历史。

    git rebase --onto O U B

上面的命令将U和B之间的差异`U..B`在分支O上重新做一遍。通常我们用的是这个命令的简化版本，例如

    A - B - C - D - E   (master)
            \
             F - G - H  (hotfix)

在分支`hotfix`上运行命令`git rebase master`，实际上就是运行`git rebase --onto master master hotfix`，最后生成如下的线性提交历史

    A - B - C - D - E - F' - G' - H' (hotfix)
                    \
                     (master)

实际应用中，当需要提交pull request时，最好在自己的分支上rebase下主分支并解决发生的conflicts，这样仓库管理员就可以直接用`git merge hotfix`来接受你的pr，而不会遇到任何冲突。

#### 注意事项
使用git rebase需要注意的一点就是**不要rebase已经提交到远程仓库的代码**。

#### 撤销rebase
需要撤销rebase引入的变更时，比较简便的方法是使用git reflog。首先通过`git reflog`和`git log HEAD@{XX}`找到恢复点，然后使用git reset --hard。如果rebase之后还没有做过其他会变更提交历史的git操作，直接执行`git reset --hard ORIG_HEAD`即可。

举例说明，假设master分支的提交记录如下（`#`后为注释内容）：

    * 2014-02-19 0101db8 update a3 # A2
    * 2014-02-19 6a741e8 update a2 # A1
    * 2014-02-19 d2e4ed4 update a  # A
    * 2014-02-19 f4d5abd init

test分支的提交记录如下：

    * 2014-02-19 82137d8 update b3 # B2
    * 2014-02-19 74bfa49 update b2 # B1
    * 2014-02-19 d2e4ed4 update a  # A
    * 2014-02-19 f4d5abd init

直观的分支图如下：

    A--A1--A2 (分支master)
     \
      \B1--B2 (分支test)

现在执行命令`git rebase test^1 master`[^1]，则分支图变为:

    A--B1--A1'--A2' (分支master)
     \
      \B1--B2       (分支test)

如果想撤销rebase操作，执行`get reflog`输出如下（节选）：

    691342e HEAD@{3}: rebase finished: returning to refs/heads/master
    691342e HEAD@{4}: rebase: update a3
    1ce3d6a HEAD@{5}: rebase: update a2
    74bfa49 HEAD@{6}: checkout: moving from test to 74bfa49cadc0bf53bd471e330e3c769509018c74^0
    0101db8 HEAD@{7}: reset: moving to HEAD^

上面的日志中，HEAD@{3} ~ HEAD@{6}属于rebase操作，我们要做的就是找到rebase操作前的那个提交，然后执行

    git log HEAD@{7}           # 查看日志以确认是正确的恢复点
    git reset --hard HEAD@{7}  # 撤销rebase操作

## 图形化界面(GUI)
git有许多很好的图形化界面，比如gitk, qgit和tig等。

### big-picture
[git-big-picture](https://github.com/esc/git-big-picture)可以将git仓库的分支和标签之间的衍生关系以图片的形式展示出来，对于直观上了解代码的历史很有帮助。

    git big-picture -o pic.png

下面是[pelican](https://github.com/getpelican/pelican)的big-picture（部分）:

![big picture of pelican default](/static/images/pelican/pelican-3.3.0-big-picture-default.jpg)

默认显示所有的分支，标签和首次提交，所以上图看上去很乱，可以通过如下选项只显示标签之间的关系（部分）

    git big-picture --no-branches --no-roots -o pic.png

![big picture of pelican tags](/static/images/pelican/pelican-3.3.0-big-picture-tags.jpg)

## 阅读资料

*  [Git Tip of the Week: Detached Heads](http://alblue.bandlem.com/2011/08/git-tip-of-week-detached-heads.html)

[^1]: test^1表示仅rebase到提交B1

