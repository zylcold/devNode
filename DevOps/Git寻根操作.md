#git 

## 引子
在git操作中，我们可以使用checkout命令检出某个状态下文件，也可以使用reset命令重置到某个状态，这里所说的“某个状态”其实对应的就是一个提交(commit).

我们可以把一个git仓库想象成一棵树，每个commit就是树上的一个节点。家家都有一本自己的祖谱。祖谱记录了一个家族的生命史，它不仅记录着该家族的来源、迁徙的轨迹，还包罗了该家族生息、繁衍、婚姻、文化、族规、家约等历史文化的全过程。类似的，每个git仓库都有一本自己的祖谱，仓库中commit ID的繁衍，HEAD指针的迁徙，分支的增加、更新，同样的记录着一个仓库从无到有的点点滴滴。

在git中，我们其实可以通过`^`和`~`来定位某个具体的commit，而不用每次都去敲繁琐的hash值。为了便于大家理解，先把结论放在前面：

1. `^`代表父提交,当一个提交有多个父提交时，可以通过在`^`后面跟上一个数字，表示第几个父提交，`^`相当于`^1`.
2. `~<n>`相当于连续的n个`^`.

[[Git如何查看当前分支提交ID的树状图？]]

## 二. 困惑

在使用git的过程中，你也许会有很多的困惑。

在使用reset或checkout命令的时候，需要一个`<commit>`参数，但是每次都输入commit hash值是一件比较麻烦的事情。首先你得去查询下日志，然后再用键盘将前面几位hash值输入。

又话说突然间，一堆带有hash值的符号出现在生活中，`HEAD^1~4`，`<commit>~3^2`，这些的含义区别是什么？

## 三. 解惑

既然commit形成的树状图，表明了各个commit之间的关系，那么我们也可以顺着这棵树去查询commit的值。一般情况下，一个commit都会有一个父提交，那么通过`<commit>^`这个表达式，就可以访问到其父提交的ID值；使用`<commit>~`也可以达到同样的功效哦。

我们知道每提交一次，HEAD就会自动移到版本库中最近的一次提交。那么`HEAD^`就代表了最近一次提交的父提交，`HEAD~`也是同样的道理；但是如果你想当然的认为^和~的用法相同，那就错了，其实它们的区别还是蛮大的。

## 四. 详解

我们来通过一个具体的例子，来讲解一下`^`和`~`的用法区别，同时在checkout或reset的过程中，看看HEAD和引用的变化。

[[Git如何查看HEAD和引用的值？]]

### master分支上初始化，并提交一次

在master分支上新建一个提交`c1`，生成commit ID 973c，这时候master引用指向973c，HEAD指向master引用。

```shell
$ git init
Initialized empty Git repository

$ echo c1 >> a

$ git add a

$ git commit
[master (root-commit) 973c5dd] c1
1 files changed,  1  insertions(+),  0  deletions(-)
create mode  100644 a

$ git log --oneline
973c5dd c1
```

对应的图如下所示：

[image:1B86C063-246A-448A-B7E5-133D14E6F640-1690-00000695C07E0AB8/T1YImFXzVeXXX9mMvu-254-87.jpg]
![[T1YImFXzVeXXX9mMvu-254-87 1.jpg]]

### 基于master新建br1分支，并提交两次

接下来在master分支基础上新建分支”br1”，并在”br1”上提交”c2”，commit ID为1c73，这时候HEAD指向br1，br1引用指向”c2”对应提交1c73.

```shell
$ git checkout -b br1
Switched to a new branch 'br1'

$ echo c2 >> b

$ git add b

$ git commit
[br1 1c7383c] c2
1 file changed, 1 insertion(+)
create mode 100644 b

$ git log --oneline
1c7383c c2
973c5dd c1
```

对应的图如下所示：

[image:5A48A9B8-7D79-4D42-82BF-DE71103520B2-1690-00000695C0641688/T1051FXwRcXXcAiK.U-229-168.png]
![[T1051FXwRcXXcAiK.U-229-168 1.png]]

在分支”br1”上，提交”c3”,commit ID为4927，此时HEAD指向br1，br1引用指向”c3”对应提交4927.

```shell
$ echo c3 >> b
$ git commit -a -m "c3"
[br1 4927c6c] c3
1 file changed, 1 insertion(+)
$ git log --oneline
4927c6c c3
1c7383c c2
973c5dd c1
```

对应的图如下所示：

[image:51EBA8AF-3F66-40B6-91FE-D09730DB682E-1690-00000695C050EE13/T1w7GEXDVeXXXFn2ZU-232-273.png]
![[T1w7GEXDVeXXXFn2ZU-232-273 1.png]]


### 切换到master分支，基于master分支新建br2分支，并提交两次

我们先切回到master分支，然后新建分支br2，先后提交”c4”和”c5”，对应的ID分别是”86ba”和”063f”，这时候HEAD指向br2，br2引用指向”c5”的对应提交063f.git 命令如下：

```shell
$ git chechout master
Switched to branch 'master'
$ git checkout -b br2
Switched to a new branch 'br2'
$ echo c4 >> c
$ git add c
$ git commit -m "c4"
[br2 86ba564] c4
1 file changed, 1 insertion(+)
create mode 100644 c
$ git log --oneline
86ba564 c4
973c5dd c1
$ echo c5 >> c
$ git commit -a -m "c5"
[br2 063f6e6] c5
1 file changed, 1 insertion(+)
$ git log --oneline
063f6e6 c5
86ba564 c4
973c5dd c1
```

对应的图如下所示：

[image:D893A7DB-4E3D-4EE0-8612-7488F57C3506-1690-00000695C03AF1D5/T1vlKFXztbXXaxZYEX-519-266.png]
![[T1vlKFXztbXXaxZYEX-519-266 1.png]]


### 切换到master分支，基于master分支创建br3分支，并提交两次

这个操作同分支br2上类似，先从br2分支切换到master分支，然后新建分支br3，分别提交”c6”和”c7”，对应的ID分别是”50f1”和”4f9c”，这时候HEAD指向br3，br2引用指向”c7”的对应提交4f9c，git 命令如下：

```shell
$ git chechout master
Switched to branch 'master'
$ git checkout -b br3
Switched to a new branch 'br3'
$ echo c6 >> d
$ git add d
$ git commit -m "c6"
[br3 50f14f6] c6
1 file changed, 1 insertion(+)
create mode 100644 d
$ git log --oneline
50f14f6 c6
973c5dd c1
$ echo c7 >> c
$ git commit -a -m "c7"
[br2 4f9ca79] c7
1 file changed, 1 insertion(+)
$ git log --oneline
4f9ca79 c7
50f14f6 c6
973c5dd c1
```

对应的图如下所示：

[image:6912EF44-BE83-4B9B-8C5F-35CB5887C05C-1690-00000695C01EA495/T1NkuFXCtbXXanz57W-763-261.png]
![[T1NkuFXCtbXXanz57W-763-261.png]]

### 切换到master分支，合并br1，br2和br3分支

先切换到master分支，然后合并br1 br2 br3，会新生成一个提交3b03.

```shell
$ git checkout master
$ git merge br1 br2 br3
3 files changed, 6 insertions(+)
create mode 100644 b
create mode 100644 c
create mode 100644 d
$ git log --oneline
3b0370b Merge braches 'br1' , 'br2' and 'br3'
4f9ca79 c7
50f14f6 c6
063f6e6 c5
86ba564 c4
4927c6c c3
1c7383c c2
973c5dd c1
```

这时候，运用git log –oneline –graph查看生成的树状图，如下所示.

[image:149F5B70-A697-4A11-80A3-F76A2FF7AB4E-1690-00000695C00A9AD6/T1nRuEXAhfXXc7WjM_-560-195.jpg]
![[T1nRuEXAhfXXc7WjM_-560-195.jpg]]

对应的图如下所示：

![[T1pfqsXwNiXXXpomUR-761-357.png]]

从上图分析

第1条线上的commit顺序是: 3b03→`4927`→1c73→973c
第2条线上的commit顺序是:3b03→`063f`→86ba→973c
第3条线上的commit顺序是:3b03→`4f9c`→50f1→973c


### `^<n>`操作
这3条线的从左至右的顺序非常重要，
`HEAD^1`对应的是第1条线的`4927`提交，
`HEAD^2`对应的是第2条线的`063f`提交，
`HEAD^3`对应的是第3条线的`4f9c`提交。

我们再来看看3b03对应节点的父提交，如下图所示：

[image:8303FC85-9859-4652-A55E-F225C715861F-1690-00000695BFE17398/T19h1DXqJhXXcUN6s_-560-166.jpg]
![[T19h1DXqJhXXcUN6s_-560-166.jpg]]

从图得知，3b03一共有三个父提交，分别是4927,063f,4f9c.

3b03没有第4个父提交，因此也没有第4条线，这时候访问HEAD^n(n>3)都会报错。


### `~<n>`操作
第1条线上的commit顺序是: 3b03→`4927`→`1c73`→`973c`
`HEAD~1`对应的是第1条线的`4927`提交，
`HEAD~2`对应的是第2条线的`1c73`提交，
`HEAD~3`对应的是第3条线的`973c`提交。


### ^n和~n的区别

`(<commit>|HEAD)^n`，指的是HEAD的第n个父提交（HEAD有多个父提交的情况下），如果HEAD有N个父提交，那么n取值为n < = N.

`(<commit>|HEAD)~n`，指的是HEAD的第n个祖先提交，用一个等式来说明就是：`(<commit>|HEAD)~n` = `(<commit>|HEAD)^^^….(^的个数为n)`.我们通过例子来验证一下吧。

我们沿用上面演示用的仓库，先检出到master分支，再使用`git checkout HEAD^2`，看看我们检出了哪个commit

```shell
$ git checkout master
$ git checkout HEAD^2
HEAD is now at 063f6e6... c5
```

我们发现”c5”对应的commit值063f正是3b03第二个父提交的commit 对应的图如下所示：

[image:2A40EC67-2DB9-4FE8-8492-F9BD0F2C4E75-1690-00000695BF91CE32/T1RruGXztaXXaHLMA2-765-337.jpg]
	![[T1RruGXztaXXaHLMA2-765-337.jpg]]

现在再切回master分支，git checkout master

然后使用git checkout HEAD^3，那么按照规律，就应该检出3b03的第三个父提交的commit，即”c7”的commit值4f9c.

```shell
$ git checkout master
Previous HEAD position was 063f6e6... c5
Switched to branch 'master'
$ git checkout HEAD ^ 3
HEAD is now at 4f9ca79... c7
```

对应的图如下所示：

[image:963167D3-C679-4085-A529-9FB895B6985C-1690-00000695BF7D89FD/T1Dk9FXqpbXXcW.JU7-770-369.jpg]
	
	![[T1Dk9FXqpbXXcW.JU7-770-369.jpg]]

果然没错，一切都在我们的预料之中！

现在验证下HEAD~的用法，切换到master分支，然后`git checkout HEAD~2`

```shell
$ git checkout master
$ git checkout HEAD~2
HEAD is now at 1c7383c... c2
```

[image:2352E44D-9ACF-4FE3-8E6A-C3D736BAB625-1690-00000695BF6583B8/T1j1WEXytfXXXO.VUN-756-358.png]
	![[T1j1WEXytfXXXO.VUN-756-358.png]]
	

这时候HEAD悄然来到了”c2”的commit 1c73，因此，HEAD~2 相当于HEAD的第一个父提交的第一个父提交。即`HEAD~2 = HEAD^^ = HEAD^1^1`, 符合预期！好开心的哟！


[[区别：git reset与git checkout]]

五.总结

1. `^`代表父提交,当一个提交有多个父提交时，可以通过在`^`后面跟上一个数字，表示第几个父提交，`^`相当于`^1`.
2. `~<n>`相当于连续的n个`^`


http://img02.taobaocdn.com/tps/i2/T1nRuEXAhfXXc7WjM_-560-195.jpg