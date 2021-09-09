将[[暂存区域]]的**快照**提交到[[Git 仓库]]

```shell
$ git commit
```

会启动文本编辑器以便输入本次提交的说明

修改默认的文本编辑器，见[[git config#配置文本编辑器]]


## 将提交信息与命令放在同一行
在  commit  命令后添加  -m  选项

```shell

$ git commit -m "Story 182: Fix benchmarks for speed"

[master 463dc4f] Story 182: Fix benchmarks for speed
2 files changed, 2 insertions(+)
create mode 100644 README
```

提交后它会告诉你,当前是在哪个分支(  master )提交的,本次提交的完整 [[SHA-1 散列]] 校验和是什么(  463dc4f ),以及在本次提交中,有多少文件修订过,多少行添加和删改过。



## 跳过使用暂存区域
给  git commit  加上  -a  选项,Git 就会自动把所有已经跟踪过的文件暂存起来一并提交

```shell

$ git commit -a -m 'added new benchmarks'
[master 83e38c7] added new benchmarks

1 file changed, 5 insertions(+), 0 deletions(-)

```


相关阅读
[[Git的撤销操作]]

