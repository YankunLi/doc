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
