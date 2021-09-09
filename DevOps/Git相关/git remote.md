管理自己的[[远程仓库]]


## 查看远程仓库
`git remote` 列出你指定的每一个远程服务器的简写, `-v`  ,额外显示其对应的 URL

```shell
$ git remote -v
origin https://github.com/schacon/ticgit (fetch)
origin https://github.com/schacon/ticgit (push)

```

## 添加远程仓库
`git remote add <shortname> <url>` 添加一个新的远程 Git 仓库,同时指定一个简写


## 查看远程仓库
`git remote show [remote-name]`查看某一个远程仓库的更多信息

```shell
$ git remote show origin
* remote origin   
	Fetch URL: https://github.com/schacon/ticgit   
	Push  URL: https://github.com/schacon/ticgit   
	HEAD branch: master   
	Remote branches:     
		master                               tracked     
		dev-branch                           tracked   
	Local branch configured for 'git pull':     
		master merges with remote master   
	Local ref configured for 'git push':     
		master pushes to master (up to date)
```
会列出远程仓库的 URL 与跟踪分支的信息


## 远程仓库的移除与重命名
`git remote rename <old> <new>`


## 移除一个远程仓库
`git remote rm <name>`




