从已跟踪文件清单中(确切地说,是从暂存区域)中移除某个文件

```shell
$ git rm PROJECTS.md
rm 'PROJECTS.md'

$ git status
On branch master
Changes to be committed:

	(use "git reset HEAD <file>..." to unstage)
	
		deleted:    PROJECTS.md
```

下一次提交时,该文件就不再纳入版本管理了。 如果删除之前修改过并且已经放到暂存区域的话,则必须要用强制删除选项 `-f`


* 想把文件从 [[Git 仓库]]中删除(亦即从[[暂存区域]]移除),但仍然希望保留在当前[[工作目录]]中

```shell
$ git rm --cached README
```

