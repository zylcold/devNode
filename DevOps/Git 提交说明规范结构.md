#git 

### Git 提交说明结构
Git **提交说明** 可分为三个部分： `Header` 、 `Body` 和 `Footer` 。 

```
<Header> 

<Body> 

<Footer>
```

### Header
Header 部分包括三个字段 `type`(必需)、 `scope`(可选)和 `subject`(必需)。 

```
<type>(<scope>): <subject>
```

> Vue源码的 **提交说明** 省略了 scope 。 

#### type
`type` 用于说明 `commit` 的提交性质。 

 值 | 描述 
---| ---
feat |新增一个功能   
fix |修复一个Bug   
docs |文档变更   
style |代码格式（不影响功能，例如空格、分号等格式修正）   
refactor |代码重构   
perf |改善性能   
test | 测试   
build | 变更项目构建或外部依赖（例如scopes: webpack、gulp、npm等）   
ci | 更改持续集成软件的配置文件和package中的scripts命令，例如scopes: Travis, Circle等   
chore |变更构建流程或辅助工具   
revert |代码回退

#### scope
`scope` 说明 `commit` 影响的范围。 `scope` 依据项目而定，例如在业务项目中可以依据菜单或者功能模块划分，如果是组件库开发，则可以依据组件划分。 

> 提示： `scope` 可以省略。 

#### `subject`
`subject` 是 `commit` 的简短描述。 

### `Body`
`commit` 的详细描述，说明代码提交的详细说明。 

### `Footer`
如果代码的提交是 **不兼容变更** 或 **关闭缺陷** ，则 `Footer` 必需，否则可以省略。 

#### 不兼容变更
当前代码与上一个版本不兼容，则 `Footer` 以 **BREAKING CHANGE** 开头，后面是对变动的描述、以及变动的理由和迁移方法。 

#### 关闭缺陷
如果当前提交是针对特定的issue，那么可以在 `Footer` 部分填写需要关闭的单个 issue 或一系列issues。 