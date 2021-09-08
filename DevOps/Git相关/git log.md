查看提交历史


已该项目源代码举例
```shell
git clone https://github.com/schacon/simplegit-progit
```


```shell

$ git log 
commit ca82a6dff817ec66f44342007202690a93763949 
Author: Scott Chacon <schacon@gee-mail.com> 
Date:   Mon Mar 17 21:52:11 2008 -0700     

	changed the version number
	
commit ca82a6dff817ec66f44342007202690a93763949 
Author: Scott Chacon <schacon@gee-mail.com> 
Date:   Mon Mar 17 21:52:11 2008 -0700     

	changed the version number

```

git log  会按提交时间列出所有的更新,最近的更新排在最上面

## 常用的参数

`-p`  			用来显示每次提交的内容差异

`-[n]` 		仅显示最近n次提交

`--stat`  每次提交的简略的统计信息

`--pretty`  可以指定使用不同于默认格式的方式展示提交历史， oneline，short ,  full  和  fuller，[[git log#format 详细使用|format]]
```shell
$ git log --pretty=oneline

ca82a6dff817ec66f44342007202690a93763949 changed the version number
```
`--graph` ASCII字符串来形象地展示你的分支、合并历史
```shell
$ git log --pretty=format:"%h %s" --graph

* 2d3acf9 ignore errors from SIGCHLD on trap
*  5e3ee11 Merge branch 'master' of git://github.com/dustin/grit 
|\ 
| * 420eac9 Added a method for getting the current branch
* | 30e367c timeout code and tests 
* | 5a09431 add timeout protection to grit 
* | e1193f8 support for heads with slashes in them 
|/ 
* d6016bc require time for xmlschema
*  11d191e Merge branch 'defunkt' into local
```


## 限制输出
* 按照时间作限制的选项，比如  `--since`  和 `--until`

所有最近两周内的提交
```shell
$ git log --since=2.weeks
```

* 列出那些添加或移除了某些字符串的提交

想找出添加或移除了某一个特定函数的引用的提交
```shell
$ git log -Sfunction_name
```

* 路径(path)限制，用两个短划线(--)隔开之前的选项和后面限定的路径名

只关心某些文件或者目录的历史提交, 可以在 git log 选项的最后指定它们的路径,
```shell
$ git log -Sfunction_name -- ./Demo
```

### 常用的选项
![[Pasted image 20210908185743.png]]


## 其他选项
![[Pasted image 20210908185345.png]]



### [[format 详细使用]]
format,可以定制要显示的记录格式
```shell
$ git log --pretty=format:"%h - %an, %ar : %s"

ca82a6d - Scott Chacon, 6 years ago : changed the version number

```


![[Pasted image 20210908184829.png]]

[[作者(author) 和 提交者(committer) 之间有何差别]]