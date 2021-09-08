不像其它的 VCS 系统,Git 并不显式跟踪文件移动操作。 如果在 Git 中重命名了某个文件, 仓库中存储的元数据并不会体现出这是一次改名操作。



```shell
$ git mv file_from file_to
```
