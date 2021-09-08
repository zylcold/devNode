对比两个版本之间的差异

```shell
$ git diff
diff --git a/CONTRIBUTING.md b/CONTRIBUTING.md
index 8ebb991..643e24f 100644

--- a/CONTRIBUTING.md 
+++ b/CONTRIBUTING.md

...
```


[[工作目录]]中当前文件和[[暂存区域]]快照之间的差异
```shell
$ git diff
```


查看已暂存的将要添加到下次提交里的内容
```shell
git diff --staged
```

注意：git diff 本身只显示尚未暂存的改动