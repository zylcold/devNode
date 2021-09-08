远程 Git 仓库中的每一个文件的每一个版本都将被拉取下来

克隆仓库的命令格式是  `git clone [url]`

```shell
$ git clone https://github.com/libgit2/libgit2
```

当前目录下创建一个名为  `libgit2` 的目录,并在这个目录下初始化一个 .git  文件夹,从远程仓库拉取下所有数据放入  .git  文件夹,然后从中读取最新版本的文件的拷贝


自定义本地仓库的名字

```
$ git clone https://github.com/libgit2/libgit2 mylibgit
```