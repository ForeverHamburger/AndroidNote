# 【Git】极简流教程教你如何多人协作开发

笔者在使用git的过程中踩了很多坑，自己用还好，只需要add和commit就好啦（不是）。

但是想要多人协作开发的时候，每次推上去拉下来都提心吊胆：万一把别人的分支弄坏怎么办，冲突不会解决怎么办？

这篇博客教你最简单、不一定最好用的与他人合作/合并代码流程。

> 参考：
>
> [十分钟学会正确的github工作流，和开源作者们使用同一套流程_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV19e4y1q7JJ/?spm_id_from=333.337.search-card.all.click&vd_source=e82cc2aea0e4c86710c97cb4273a2830)

## 新建个人分支

![image-20250126114951831](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261149892.png)

首先可以把复杂的流程想象成三个仓库之间的交互，Github在这里代指远端仓库，Git仓库即本地的仓库，本地则代指还未提交至Git仓库的代码。

首先我们工作的第一步肯定是将Github上面的代码拉取到本地Git仓库。

> 命令：git clone
>
> 命令将远程仓库克隆到本地。这会创建一个本地仓库的副本，包含所有的分支和历史记录。

![image-20250126121808680](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261218750.png)

而后，我们就可以开始新建个人分支，而后在个人的分支上开始开发了。

> git checkout -b xxx

这一步相当于在本地复制了一个新的工作空间，你可以在这个分支上自由地进行开发，而不会影响到其他分支的代码。

![image-20250126122133284](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261221316.png)

这一步相当于复制了remote的仓库到本地的your分支上。

这时假设咱们哐哐哐写了一堆东西。

![image-20250126122336178](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261223211.png)

在提交之前，可以用

> git diff：查看自己的代码与上次提交的区别是什么

该命令查看当前代码与上次提交之间的差异。这可以帮助你了解自己做了哪些修改，避免提交错误的代码。

而后使用

> git add 文件名

将需要提交的文件提交至暂存区，或者也可以直接用

> git add .

将所有文件都提交到暂存区，最终确认可以提交之后使用

> git commit -m “叽里咕噜说啥呢”

将提交以及提交相关的信息提交至个人分支即可。

![image-20250126122646535](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261226577.png)

最后再用

> git push origin your分支

将本地的你的git分支上传至github上的git即可。

## 如果发现远端有更新

多人开发的时候肯定不止咱一个人会往上提交，如果在我们想要提交的时候发现master分支又多了更新怎么办呢？

我们首先肯定需要将master分支的新的更新首先拉取下来，测试一下自己的个人分支是否出现问题。

此时我们首先切换回master分支，使用

> git checkout master

![image-20250126123132708](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261231746.png)

这时我们磁盘中的代码状态是初始提交的状态，而非更新过的状态，接着我们将远端的更新拉取下来，将远端的master同步至本地。

> 这一步使用命令：
>
> git pull origin master

![image-20250126123415372](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261234413.png)

这样我们本地的代码就同远端github一样，而后再切换回我们的your分支。

> git checkout your分支

![image-20250126123625647](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261236688.png)

此时本地的状态与先前一致，即保留自己写的更新，但并没有网上Git仓库的更新。

为了同步master的代码，我们使用

> git rebase master

变基命令的原理是，将我们自己写的修改先抛到一边，先将master的代码挪过来，而后在master的基础上，将我们的修改尝试加入到master中。

这一步可能出现rebase conflict冲突，需要自行手动选择需要哪一段代码。

![image-20250126123945434](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202501261239482.png)

变基完成后的状态是这样。

而后我们需要使用

> git push -f origin your分支

将我们本地的git分支push到github上。

## 将个人分支合并到master

到这一步基本就大功告成了，最后我们需要将个人的分支提交到master分支上，这一步在github上操作即可。

也就是pull request。

> - 在 GitHub 上，通过 Web 界面创建一个 Pull Request（PR），将你的个人分支的代码合并到主分支。
> - 在 PR 页面上，你可以查看代码的差异、添加描述和评论，并请求其他开发者进行代码审查。
> - 代码审查是一个重要的环节，可以帮助发现潜在的问题、改进代码质量，并确保团队成员对代码的变更有清晰的了解。

> **Squash and merge**：将个人分支上的所有提交合并成一个提交，然后合并到主分支。这种方式可以使主分支的历史记录更加简洁清晰。
>
> **Create a merge commit**：创建一个新的合并提交，将个人分支的代码合并到主分支。这种方式保留了个人分支的所有提交历史，但会使主分支的历史记录变得复杂。
>
> **Rebase and merge**：将个人分支的代码基于主分支进行变基，然后合并到主分支。这种方式可以使主分支的历史记录保持线性，但需要开发者对变基操作有较好的理解。

一般会选择squash and merge，也就是将某一分支上的所有改变合并成一个改变，放入master分支。