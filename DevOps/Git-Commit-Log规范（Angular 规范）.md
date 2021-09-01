整理自[《Commit message 和 Change log 编写指南》-阮一峰](https://link.jianshu.com/?t=http://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html)

# Commit message目的

commit message 应该清晰明了，说明本次提交的目的。

# Angular 规范

标准格式如下：

```xml
<type>(<scope>): <subject> #header
// 空一行
<body>
// 空一行
<footer> 
```

> Header 是必需的，Body 和 Footer 可以省略。

## 格式讲解

### Header

Header部分只有一行，包括三个字段：type（必需）、scope（可选）和subject（必需）。

#### type

用于说明本次commit的类别，只允许使用下面7个标识

-   `feat`：新功能（feature）
-   `fix`：修补bug
-   `docs`：文档（documentation）
-   `style`： 格式（不影响代码运行的变动）
-   `refactor`：重构（即不是新增功能，也不是修改bug的代码变动）
-   `test`：增加测试
-   `chore`：构建过程或辅助工具的变动

> 如果type为feat和fix，则该 commit 将肯定出现在 Change log 之中。其他情况（docs、chore、style、refactor、test）由你决定，要不要放入 Change log，建议是不要。

#### scope

用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。

#### subject

是 commit 目的的简短描述，不超过50个字符。

```undefined
以动词开头，使用第一人称现在时，比如change，而不是changed或changes
第一个字母小写
结尾不加句号（.）
```

### Body

用于对本次 commit 的详细描述，可以分成多行。下面是一个范例。

```php
More detailed explanatory text, if necessary.  Wrap it to 
about 72 characters or so. 

Further paragraphs come after blank lines.

- Bullet points are okay, too
- Use a hanging indent
```

有两个注意点：

1.  使用第一人称现在时，比如使用change而不是changed或changes。
2.  应该说明代码变动的动机，以及与以前行为的对比。

### Footer

Footer 部分只用于两种情况。

#### 不兼容变动

如果当前代码与上一个版本不兼容，则 Footer 部分以`BREAKING CHANGE`开头，后面是对变动的描述、以及变动理由和迁移方法。

```go
BREAKING CHANGE: isolate scope bindings definition has changed.

To migrate the code follow the example below:

Before:

scope: {
  myAttr: 'attribute',
}

After:

scope: {
  myAttr: '@',
}

The removed `inject` wasn't generaly useful for directives so there should be no code using it.
```

#### 关闭 Issue

如果当前 commit 针对某个issue，那么可以在 Footer 部分关闭这个 issue 。

```bash
Closes #234
```

也可以一次关闭多个 issue 。

```css
Closes #123, #245, #992
```

### Revert（可忽视）

还有一种特殊情况，如果当前 commit 用于撤销以前的 commit，则必须以`revert:`开头，后面跟着被撤销 Commit 的 Header。

```csharp
revert: feat(pencil): add 'graphiteWidth' option

This reverts commit 667ecc1654a317a13331b17617d973392f415f02.
```

Body部分的格式是固定的，必须写成This reverts commit <hash>.，其中的hash是被撤销 commit 的 SHA 标识符。

如果当前 commit 与被撤销的 commit，在同一个发布（release）里面，那么它们都不会出现在 Change log 里面。如果两者在不同的发布，那么当前 commit，会出现在 Change log 的Reverts小标题下面。

# 提交频率

关于什么时候提交一次：

每次你写完一个功能的时候，就应该做一次提交（这个提交是提交到本地的git库中）

> 当然，这里的写完表示的是你的这个功能是没有问题的；

# 参考文档

[Git 提交记录和分支模型](https://link.jianshu.com/?t=https://cattail.me/tech/2016/06/06/git-commit-message-and-branching-model.html)

  
  
作者：huanqiang  
链接：https://www.jianshu.com/p/c7e40dab5b05  
来源：简书  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。