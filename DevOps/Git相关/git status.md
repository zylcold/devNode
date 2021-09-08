> 检查当前文件状态


* 如果在克隆仓库后立即使用此命令,会看到类似这样的输出:

```shell
$ git status 
On branch master 
nothing to commit, working directory clean
```

分支名是 master
所有已跟踪文件在上次提交后都未被更改过
当前目录下没有出现任何处于未跟踪状态的新文件


## 状态简览
使用  `git status -s`  命令或 `git status --short`  命令,你将得到一种更为紧凑的格式输出

```shell

$ git status -s  
 M README 
MM Rakefile 
A  lib/git.rb 
M  lib/simplegit.rb 
?? LICENSE.txt

```

标记 | 说明
--|--
?? | 未追踪文件
A | 新添加到[[暂存区域]]中的文件
M(右) | 该文件被修改了但是还没放入[[暂存区域]]
M(左)|该文件被修改了并放入了[[暂存区域]]


## 文件状态的理解
* 创建一个新的 README 文件

```shell

$ echo 'My Project' > README

$ git status
On branch master
Untracked files: 
	(use "git add <file>..." to include in what will be committed)

		README

nothing added to commit but untracked files present (use "git add" to track)
```

新建的 README 文件出现在  Untracked files  下面。


* 使用命令 [[git add]]  开始跟踪一个文件

```shell

$ git add README


$ git status
On branch master
Changes to be committed:

	(use "git reset HEAD <file>..." to unstage
	
		new file:   README

```


文件出现在  Changes to be committed  下面的,就说明是已暂存状态


* 修改一个已追踪的文件，CONTRIBUTING.md

```shell
$ git status
On branch master
Changes to be committed:

	(use "git reset HEAD <file>..." to unstage
	
		new file:   README


Changes not staged for commit:

	(use "git add <file>..." to update what will be committed)
	(use "git checkout -- <file>..." to discard changes in working directory)
	
		modified:   CONTRIBUTING.md
```

文件  CONTRIBUTING.md  出现在  Changes not staged for commit  这行下面,说明已跟踪文件的内容发生了变化,但还没有放到暂存区


* 再次修改README文件

```shell
$ git status
On branch master
Changes to be committed:

	(use "git reset HEAD <file>..." to unstage
	
		new file:   README


Changes not staged for commit:

	(use "git add <file>..." to update what will be committed)
	(use "git checkout -- <file>..." to discard changes in working directory)
	
		modified:   CONTRIBUTING.md
		modified:   README
		
```

README  文件同时出现在暂存区和非暂存区


![[Git的文件状态#Git文件生命周期]]