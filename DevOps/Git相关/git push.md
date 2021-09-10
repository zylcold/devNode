推送到远程仓库

`git push [remote-name] [branch-name]`

```shell
$ git push origin master
```

## 共享标签
默认情况下,  git push  命令并不会传送标签到远程仓库服务器上。 在创建完标签后你必须显式地推送标签到共享服务器上。

使用`git push origin [tagname]`，将特定标签推送

使用`--tags` 推送多个标签

```shell
$ git push origin --tags
```