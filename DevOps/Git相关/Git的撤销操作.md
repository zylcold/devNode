### 撤销刚提交的Commit信息
[[git commit]]

```shell
$ git commit --amend
```

### 撤销暂存的文件
[[git reset]]

`git reset HEAD <file>…`

```shell
$ git reset HEAD CONTRIBUTING.md
Unstaged changes after reset:
M CONTRIBUTING.md


$ git status
On branch master
Changes to be committed:
	(use "git reset HEAD <file>..." to unstage)
	
		renamed:    README.md -> README
		
Changes not staged for commit:
	(use "git add <file>..." to update what will be committed)
	(use "git checkout -- <file>..." to discard changes in working directory)
	
		modified:   CONTRIBUTING.md
```


### 撤消对文件的修改
[[git checkout]]

`git checkout -- <file>…`

```shell
$ git checkout -- CONTRIBUTING.md
```

`git checkout — [file]`  是一个危险的命令