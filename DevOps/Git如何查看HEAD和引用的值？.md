#git #git/cmd

可以通过命令来查看HEAD和引用的值，也可以通过当前仓库下的.git目录去访问。当前分支为master时，我们查看HEAD的值，命令如下：

```shell
$ cat .git/HEAD
ref: refs/heads/master
```

然后，我们可以查看master引用的值

```shell
$ cat .git/refs/heads/master
3b0370b....... # hash code
```