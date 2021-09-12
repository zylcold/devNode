[[Git 是如何保存数据#Tag、分支、HEAD的表述]]

在 Git 中,它是一个指针,指向当前所在的本地分支。


# 举例

创建一个testing分支后

```shell
$ git branch testing
```

![[Pasted image 20210910164500.png]]

切换testing分支，[[git checkout#分支切换]]

```shell
$ git checkout testing
```

![[Pasted image 20210910164604.png]]

修改文件，再提交

```shell

$ vim test.rb 
$ git commit -a -m 'made a change'

```

![[Pasted image 20210910164714.png]]


**HEAD 分支随着提交操作自动向前移动**


再切回master分支
```shell
$ git checkout master
```

![[Pasted image 20210910164811.png]]