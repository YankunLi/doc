## 建立本地分支与远端分支映射关系
```
git config --set-upstream-to {remote repository addr}/{remote branch name} {local branch name}
```

## 查看本地分支与远程分支的映射关系:
```
git branch -vv
```

## 切换分支
```
git checkout {branch name}           //该分支必须已经存在
git checkout -d {branch name}        //切换并创建分支
```

## 合并多次提交使提交历史更干净
```
git rebase -i {start point} [end point]
```

## 展示某次提交的改动内容
列举提交历史
```
git log
```
显示某次提交的内容
```
git show {commit-hashId}
```
显示某次提交对某个文件的改动
```
git show {commit-hashId} {filename}
```

## 暂存修改的内容
保存修改的内容,并添加描述信息
```
git stash save "desc message"
```
显示做了哪些改动，默认show第一个存储,如果要显示其他存贮，后面加stash@{$num}, 比如第二个 git stash show stash@{1}
```
git stash show
```
查看stash了哪些存储
```
git stash list
```
显示第一个存储的改动，如果想显示其他存存储，命令：git stash show  stash@{$num}  -p ，比如第二个：git stash show  stash@{1}  -p
```
git stash show -p
```
应用某个存储,但不会把存储从存储列表中删除，默认使用第一个存储,即stash@{0}，如果要使用其他个，git stash apply stash@{$num} ， 比如第二个：git stash apply stash@{1}
```
git stash apply
```
命令恢复之前缓存的工作目录，将缓存堆栈中的对应stash删除，并将对应修改应用到当前的工作目录下,默认为第一个stash,即stash@{0}，如果要应用并删除其他stash，命令：git stash pop stash@{$num} ，比如应用并删除第二个：git stash pop stash@{1}
```
git stash pop
```
丢弃stash@{$num}存储，从列表中删除这个存储
```
git stash drop stash@{$num}
```
删除所有缓存的stash
```
git stash clear
```
