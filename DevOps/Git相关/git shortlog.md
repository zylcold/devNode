使用  git shortlog  命令可以快速生成一份包含从上次发布之后项目新增内容的修改日志(changelog)类文档。 它会对你给定范围内的所有提交进行总结;

```shell
$ git shortlog --no-merges master --not v1.0.1
```
这份整洁的总结包括了自 v1.0.1 以来的所有提交,并且已经按照作者分好组