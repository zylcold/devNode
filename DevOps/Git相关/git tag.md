## 列出标签
`git tag`

```shell
$ git tag
v0.1
v1.3
```

## 可以使用特定的模式查找标签
`-l`使用特定模式查找

```shell
$ git tag -l 'v1.8.5*'
v1.8.5 
v1.8.5-rc0 
v1.8.5-rc1 
v1.8.5-rc2 
v1.8.5-rc
```

## 创建标签

Git 使用两种主要类型的标签:[[轻量标签]]与[[附注标签]]

### 创建附注标签
`git tag -a [tag] -m [msg]`

```shell
$ git tag -a v1.4 -m 'my version 1.4'
```


### 创建轻量标签
`git tag [tag]`

```shell
$ git tag v1.4-lw
```

### 对过去的提交打标签
`git tag [tag] [commit]`

```shell
$ git tag -a v1.2 9fceb02
```

### 推送标签
![[git push#共享标签]]

### 检出标签
![[git checkout#检出标签]]