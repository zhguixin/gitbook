在工作中可以通过命令：
```bash
$ git log --author zhguixin
```
查看某一作者的提交记录，进一步的话可以通过`git diff`来比较不同提交记录时各个版本的差异。具体解释如下：
`git diff --cached` 显示索引区与git仓库之间的差异
`git diff HEAD` 显示当前工作目录与git仓库之间的差异
`git diff HEAD^` 比较上次提交
`git diff HEAD~2` 比较上2次提交

`--diff-filter=[ACDMRTUXB*]`
        显示指定状态的文件：Added (A), Copied (C), Deleted (D), Modified (M), Renamed (R), changed (T), are Unmerged (U), are Unknown (X)

`git diff --stat` 列出文件

`git diff -- filename`    只对比给定的文件

历史提交对比：
```$ git diff commit```       将所指定的某次提交与当前工作目录进行对比。

```$ git diff commit1 commit2``` 将2次提交的内容进行对比
等价于
```$ git diff commit1..commit2``` 如果省略任意一个commit，默认将使用HEAD代替

commit可以是简写的commit哈希值，也可是是HEAD。其中HEAD代表最后一次提交，HEAD^代表最后一次提交的父提交，HEAD~1等价于HEAD^,HEAD~2为倒数第二次提交，以此类推。