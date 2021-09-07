#git #git/cmd 

checkout只会移动HEAD指针，reset会改变HEAD的引用值。

![[T1pfqsXwNiXXXpomUR-761-357.png]]

## reset

在master分支上，当前提交为3b03，使用git reset –hard HEAD^，将master重置到HEAD的父提交；该命令也可以写成`git reset –hard HEAD^1`

```shell
$ git reset --hard HEAD^
HEAD is now at 4927c6c c3
```

对应的图如下所示：
![[T1sBqFXtlcXXXFjZMI-754-342.png]]


```
这时候，HEAD还是指向master分支，但是master引用的commit值已经变成了4927，即3b03的第一个父提交的ID.
```


## reset
然后，我们再重置到”c8”的commit”3b03”，`git reset –hard 3b03`，然后使用命令`git checkout HEAD~`，git 操作如下：

```shell
$ git reset --hard 3b03
HEAD is now at 3b0370b Merge branches 'br1' , 'br2' and 'br3'

$ git checkout HEAD~
HEAD is now at 4927c6c... c3
```

对应的图如下所示：
![[T1swGGXr4aXXXGCwoP-760-360.jpg]]

这时候，HEAD指向了commit 4927，即3b03的第一个父提交ID，但是master引用还是对应的3b03.

`reset <commit>`的时候，HEAD不变，但是HEAD指向的引用值会变成相应的`<commit>`值；`checkout <commit>`的时候，HEAD直接变成`<commit>`值，但原来引用中保存的值不变。