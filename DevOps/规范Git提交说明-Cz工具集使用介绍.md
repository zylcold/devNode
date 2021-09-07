#devops #git

在多人协作的项目中，如果Git的 **提交说明** 精准，在后期协作以及Bug处理时会变得有据可查，项目的开发可以根据规范的 **提交说明** 快速生成开发日志，从而方便开发者或用户追踪项目的开发信息和功能特性。 

本文主要内容：

* 介绍符合Angular规范（需要翻墙）的 **提交说明**
* 介绍 **提交说明** 工具集 [cz](https://github.com/commitizen/cz-cli) （适配器、校验以及日志）的使用方法

这里提供演示项目地址： [cz-example](https://github.com/ziyi2/cz-example) 。 

## Git的提交说明
[[Git提交说明 commit]]

没有规范的 **提交说明** 
* 很难阐述当前代码的提交性质（修复Bug、代码性能优化、新增功能或者发布版本等）。
 
查看Vue项目的Git **提交说明** （ `fix` 表明修复问题、 `feat` 表明新增功能...），它完全符合Angular规范： 
![[62bba7ade2ec38566dbdbf088ddd80d4.jpeg]]

手写符合规范的 **提交说明** 很难避免错误，可以借助 [工具](http://www.codercto.com/tool.html) 来实现规范的 **提交说明** 。 

## 规范的Git提交说明
* 提供更多的历史信息，方便快速浏览
* 可以过滤某些 `commit` ，便于筛选代码 `review`
* 可以追踪 `commit` 生成更新日志
* 可以关联 **issues**

[[Git提交说明规范结构]]

## Commitizen
[commitizen/cz-cli](https://github.com/commitizen/cz-cli) 是一个可以实现规范的 **提交说明** 的工具： 

> When you commit with Commitizen, you'll be prompted to fill out any required commit fields at commit time. No more waiting until later for a git commit hook to run and reject your commit (though that can still be helpful). No more digging through CONTRIBUTING.md to find what the preferred format is. Get instant feedback on your commit message formatting and be prompted for required fields.

提供选择的提交信息类别，快速生成 **提交说明** 。安装cz工具: 

```shell
npm install -g commitizen
```


## Commitizen适配器
### cz-conventional-changelog
如果需要在项目中使用 commitizen 生成符合AngularJS规范的 **提交说明** ，初始化 [[Cz工具集-cz-conventional-changelog]] 适配器：

接下来可以使用cz的命令 `git cz` 代替 `git commit` 进行 **提交说明** ： 

![[914a09f13dd5200ff08a6a25c8d799a7.jpeg]]

代码提交到远程的Github后，可以在相应的项目中进行查看，例如（这里使用 `feat` 不是很合适，只是一个示例）： 

![[94fd0f81ac432e81ebcd0f68f86b7abe.jpeg]]


### cz-customizable
如果想定制项目的 **提交说明** ，可以使用[[Cz工具集-cz-customizable]]

> cz-customizable will first look for a file called .cz-config.js，alternatively add a config block in your package.json。

官方提供了一个 `.cz-config.js` 示例文件 [cz-config-EXAMPLE.js](https://github.com/leonardoanalista/cz-customizable/blob/master/cz-config-EXAMPLE.js) ，如下所示： 

```javascript
'use strict';

module.exports = {

  types: [
    {value: 'feat',     name: 'feat:     A new feature'},
    {value: 'fix',      name: 'fix:      A bug fix'},
    {value: 'docs',     name: 'docs:     Documentation only changes'},
    {value: 'style',    name: 'style:    Changes that do not affect the meaning of the code\n            (white-space, formatting, missing semi-colons, etc)'},
    {value: 'refactor', name: 'refactor: A code change that neither fixes a bug nor adds a feature'},
    {value: 'perf',     name: 'perf:     A code change that improves performance'},
    {value: 'test',     name: 'test:     Adding missing tests'},
    {value: 'chore',    name: 'chore:    Changes to the build process or auxiliary tools\n            and libraries such as documentation generation'},
    {value: 'revert',   name: 'revert:   Revert to a commit'},
    {value: 'WIP',      name: 'WIP:      Work in progress'}
  ],

  scopes: [
    {name: 'accounts'},
    {name: 'admin'},
    {name: 'exampleScope'},
    {name: 'changeMe'}
  ],

  // it needs to match the value for field type. Eg.: 'fix'
  /*
  scopeOverrides: {
    fix: [
      {name: 'merge'},
      {name: 'style'},
      {name: 'e2eTest'},
      {name: 'unitTest'}
    ]
  },
  */
  // override the messages, defaults are as follows
  messages: {
    type: 'Select the type of change that you\'re committing:',
    scope: '\nDenote the SCOPE of this change (optional):',
    // used if allowCustomScopes is true
    customScope: 'Denote the SCOPE of this change:',
    subject: 'Write a SHORT, IMPERATIVE tense description of the change:\n',
    body: 'Provide a LONGER description of the change (optional). Use "|" to break new line:\n',
    breaking: 'List any BREAKING CHANGES (optional):\n',
    footer: 'List any ISSUES CLOSED by this change (optional). E.g.: #31, #34:\n',
    confirmCommit: 'Are you sure you want to proceed with the commit above?'
  },

  allowCustomScopes: true,
  allowBreakingChanges: ['feat', 'fix'],

  // limit subject length
  subjectLimit: 100
  
};

```

这里对其进行汉化处理（只是为了说明定制说明的一个示例）：

```javascript
'use strict';

module.exports = {

  types: [
    {value: '特性',     name: '特性:    一个新的特性'},
    {value: '修复',      name: '修复:    修复一个Bug'},
    {value: '文档',     name: '文档:    变更的只有文档'},
    {value: '格式',    name: '格式:    空格, 分号等格式修复'},
    {value: '重构', name: '重构:    代码重构，注意和特性、修复区分开'},
    {value: '性能',     name: '性能:    提升性能'},
    {value: '测试',     name: '测试:    添加一个测试'},
    {value: '工具',    name: '工具:    开发工具变动(构建、脚手架工具等)'},
    {value: '回滚',   name: '回滚:    代码回退'}
  ],

  scopes: [
    {name: '模块1'},
    {name: '模块2'},
    {name: '模块3'},
    {name: '模块4'}
  ],

  // it needs to match the value for field type. Eg.: 'fix'
  /*
  scopeOverrides: {
    fix: [
      {name: 'merge'},
      {name: 'style'},
      {name: 'e2eTest'},
      {name: 'unitTest'}
    ]
  },
  */
  // override the messages, defaults are as follows
  messages: {
    type: '选择一种你的提交类型:',
    scope: '选择一个scope (可选):',
    // used if allowCustomScopes is true
    customScope: 'Denote the SCOPE of this change:',
    subject: '短说明:\n',
    body: '长说明，使用"|"换行(可选)：\n',
    breaking: '非兼容性说明 (可选):\n',
    footer: '关联关闭的issue，例如：#31, #34(可选):\n',
    confirmCommit: '确定提交说明?'
  },

  allowCustomScopes: true,
  allowBreakingChanges: ['特性', '修复'],

  // limit subject length
  subjectLimit: 100

};

```

再次使用 `git cz` 命令进行 **提交说明** ：

![[4c57b99009d01953f757dda710cb2c34.jpeg]]

从上图可以看出此时的 **提交说明** 选项已经汉化，继续填写 **提交说明** ： 
![[1060444c51809923f340aa127f6c1b3d.jpeg]]

把代码提交到远程看看效果：
![[1d5d694cbe4c5b58b3d3858f0af69612.jpeg]]

## Commitizen校验
### commitlint
校验提交说明是否符合规范，使用校验工具 [[Cz工具集-commitlint]]

### @commitlint/config-conventional
安装符合Angular风格的校验规则，[[Cz工具集-config-conventional]]

### Git钩子工具
[[钩子工具-huksy]]

需要注意，使用该校验规则不能对 `.cz-config.js` 进行不符合Angular规范的定制处理，例如之前的汉化，此时需要将 `.cz-config.js` 的文件按照官方示例文件 [cz-config-EXAMPLE.js](https://github.com/leonardoanalista/cz-customizable/blob/master/cz-config-EXAMPLE.js) 进行符合Angular风格的改动。 

执行错误的提交说明：
![[b45ca9388bda82901426f5be8cbcc3c0.jpeg]]

执行符合Angular规范的提交说明：
![[189ca2ec60b2f2831eefa3295af3e034.jpeg]]

commitlint需要配置一份校验规则， [@commitlint/config-conventional](https://github.com/marionebl/commitlint/tree/master/@commitlint/config-conventional) 就是符合Angular规范的一份校验规则。 

### commitlint-config-cz
如果是使用 **cz-customizable** 适配器做了破坏Angular风格的提交说明配置，那么不能使用**@commitlint/config-conventional**规则进行提交说明校验，可以使用 [commitlint-config-cz](https://github.com/whizark/commitlint-config-cz) 对定制化提交说明进行校验。 

安装校验规则：

```shell
npm install commitlint-config-cz --save-dev
```

然后加入commitlint校验规则配置：

```shell
module.exports = {
  extends: [
    'cz'
  ]
};
```

这里推荐使用**@commitlint/config-conventional**校验规则，如果想使用cz-customizable适配器，那么定制化的配置不要破坏Angular规范即可。

### validate-commit-msg
除了使用 **commitlint** 校验工具，也可以使用 [validate-commit-msg](https://github.com/Frikki/validate-commit-message) 校验工具对cz提交说明是否符合Angular规范进行校验。 

## Commitizen日志

如果使用了 [cz](https://github.com/commitizen/cz-cli) 工具集，配套 [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog) 可以快速生成开发日志： 

```shell
npm install conventional-changelog -D
```

在 `pacage.json` 中加入生成日志命令： 

```shell
"version": "conventional-changelog -p angular -i CHANGELOG.md -s -r 0 && git add CHANGELOG.md"
```

You could follow the following workflow

* Make changes
* Commit those changes
* Pull all the tags
* Run the npm version [patch|minor|major] command
* Push

执行 `npm run version` 后可查看生产的日志 [CHANGELOG.md](https://github.com/ziyi2/cz-example/blob/master/CHANGELOG.md) 。 

注意要使用正确的 `Header` 的 `type` ，否则生成的日志会不准确，这里只是一个示例，生成的日志不是很严格。 

以上就是本文的全部内容，希望对大家的学习有所帮助，也希望大家多多支持 

[Cz工具集使用介绍 - 规范Git提交说明 | 码农网](https://www.codercto.com/a/72615.html)
