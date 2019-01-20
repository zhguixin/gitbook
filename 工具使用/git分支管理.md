---
title: Git分支管理
date: 2016-08-20
tags: Git
categories: 
---

### 创建分支
开发者user1 基于当前HEAD创建分支mydev：
```bash
git branch mydev
```
此时创建了分支并不会自动切换到刚刚创建的分支上，用`git branch`命令查看当前所在的分之：
```bash
git branch
```
执行git checkout 命令切换到新分支上:
```bash
git checkout mydev
```

### 同步到主分支
现在假设在`mydev`分支上修改并做了提交，为了将修改的内容同步到主分支上，需要进行：
先切换到主分支上，再运行`merge`命令。
```bash
git checkout master
git merge mydev
```
此时用`git status`查看当前状态，会提示你本地的master分支领先于服务器上的`origin/master`分支。可以通过：
```bash
git cherry
```
查看哪些提交领先。接着可以用：`git push`命令推送到服务器上。

### 删除分支
此时就可以删掉本地分支了：
```bash
git branch -D mydev
```

**附：分支管理常用的命令**
```bash
git branch 不带参数：列出本地已经存在的分支，并且在当前分支的前面加“*”号标记
git branch -r 列出远程分支
git branch -a 列出本地分支和远程分支
git branch 创建一个新的本地分支，需要注意，此处只是创建分支，不进行分支切换
git branch -m | -M oldbranch newbranch 重命名分支，如果newbranch名字分支已经存在，则需要使用-M强制重命名，否则，使用-m进行重命名。
git branch -d | -D branchname 删除branchname分支
git branch -d -r branchname 删除远程branchname分支
```