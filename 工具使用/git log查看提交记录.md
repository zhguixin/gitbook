---
title: git log查看提交记录
date: 2016-08-27
tags: Git
categories: 
---
### git log 查看提交记录，参数：
-n (n是一个正整数)，查看最近n次的提交信息
```bash
git log -2 #查看最近2次的提交历史记录
```

### 查看具体提交信息
```bash
git log file #查看file文件的提交记录
git log file1 file2 #查看file1和file2文件的提交记录
git log file_dir\ #查看file_dir下所有文件的提交记录
git log --author=zhguixin #查看某一作者的提交记录
```

### git blame
如果你要查看文件的每个部分是谁修改的, 那么 git blame 就是不二选择. 只要运行'git blame [filename]',你就会得到整个文件的每一行的详细修改信息:包括SHA串,日期和作者.

### 重命名本地分支
```bash
 git br -m devel develop
```

当前用户的Git配置文件放在用户主目录下的一个隐藏文件`.gitconfig`中

####git 配置别名：

```bash
$ git config --global alias.co checkout
$ git config --global alias.ci commit
$ git config --global alias.br branch
$ git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```