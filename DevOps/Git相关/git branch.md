分支操作

## 创建分支
`git branch [branchName]`

**仅创建分支，不会自动切换到新分支中去**

## 分支管理
### 查看分支列表
`git branch`

```shell
$ git branch   

iss53 
* master   
testing


# * 字符:它代表现在检出的那一个分支
```

### 查看每一个分支的最后一次提交

运行 ` git branch -v ` 命令

### 查看已经合并到当前分支的分支列表

运行 `git branch -merged`

### 查看未合并到当前分支的分支列表

运行 `git branch --no-merged`

## 删除分支
`git branch -d [branchName]`

使用  `-D`  选项强制删除


