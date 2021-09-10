> 帮助设置控制 Git 外观和行为的配置变量


## 设置你的用户名称与邮件地址

```shell
$ git config --global user.name "John Doe" 
$ git config --global user.email johndoe@example.com
```

## 配置文本编辑器
（默认是Vim）
```
$ git config --global core.editor emacs
```

## 检查配置信息

使用 `git config --list`

## git config存储不同的位置
1.  `/etc/gitconfig`文件: 包含系统上每一个用户及他们仓库的通用配置。 修改/读取使用  `-system`  选项
2.  `~/.gitconfig`  或 `~/.config/git/config`文件:只针对当前用户。 修改/读取使用 `--global` 选项
3. 当前使用仓库的 Git 目录中的  config  文件(就是  `.git/config` )：针对该仓库


## 为命令设置别名
```shell

# git co == git checkout
$ git config --global alias.co checkout

# git br == git branch
$ git config --global alias.br branch


# git unstage fileA == git reset HEAD -- fileA
$ git config --global alias.unstage 'reset HEAD --'

```
👆 Git 只是简单地将别名替换为对应的命令

想要执行外部命令可以在命令前面加入  !  符号
```shell

# git visual == gitk
$ git config --global alias.visual '!gitk'
```
