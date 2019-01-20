---
title: Git常用的操作指令 
date: 2016-08-05
tags: Git
categories: 
---

### 修改最后一次提交
有时候我们提交完了才发现漏掉了几个文件没有加，或者提交信息写错了。想要撤消刚才的提交操作，可以使用`--amend` 选项重新提交：
```bash
$ git commit --amend -m"修改 提交 说明"
```
此命令将使用当前的暂存区域快照提交。如果刚才提交完没有作任何改动，直接运行此命令的话，相当于有机会 重新编辑提交说明，但将要提交的文件快照和之前的一样。

启动文本编辑器后，会看到上次提交时的说明，编辑它确认没问题后保存退出，就会使用新的提交说明覆盖刚才失误的提交。

如果刚才提交时忘了暂存某些修改，可以先补上暂存操作，然后再运行 `--amend` 提交：
```bash
$ git commit -m 'initial commit'
$ git add forgotten_file
$ git commit --amend
```
上面的三条命令最终只是产生一个提交，第二个提交命令修正了第一个的提交内容。

### git revert
`git revert` 是 **撤销** 某次操作，此次操作之前和之后的commit都会被保留，并且 **会把这次撤销**作为一次最新的提交；

```bash
$ git revert HEAD                  撤销前一次 commit
$ git revert HEAD^               撤销前前一次 commit
$ git revert commit （比如：fa042ce57ebbe5bb9c8db709f719cec2c58ee7ff）撤销指定的版本，撤销也会作为一次提交进行保存。
```

- git revert是提交一个新的版本，将需要revert的版本的内容再反向修改回去，版本会递增，不影响之前提交的内容。

- git reset 是撤销某次提交，但是此次之后的修改都会被退回到暂存区；git reset是还原到指定的版本上，这将扔掉指定版本之后的版本

#### 打patch

通过`git diff` 生成一个patch包：zgx.patch

```bash
$ git diff > zgx.patch
$ git apply zgx.patch
```

如果有冲突，会failed。可以使用：

```bash
git apply --reject zgx.patch
```

这时候会生成一些xxx.rej的文件，就是冲突的地方，不能合并进库，没有冲突的地方都会合并到库中，根据xxxx.rej 解决冲突，删除xxx.rej，就可以重新提交了 。



```bash
git clean -df 删除本地修改的文件和目录
git ck . 清空本地所有修改

git cherry-pick --abort
git log --author=zhangguixin

git stash       #先将本地修改存储起来
git pull          #暂存了本地修改之后，就可以pull了
git stash pop     #还原暂存的内容
```

