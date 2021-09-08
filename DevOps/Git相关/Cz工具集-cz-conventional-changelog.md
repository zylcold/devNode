```shell
commitizen init cz-conventional-changelog --save --save-exact
```

如果当前已经有其他适配器被使用，则会报以下错误，此时可以加上 `--force` 选项进行再次初始化 
```shell
Error: A previous adapter is already configured. Use --force to override
```

初始化命令主要进行了3件事情

1. 在项目中安装 **cz-conventional-changelog** 适配器依赖
2. 将适配器依赖保存到 `package.json` 的 `devDependencies` 字段信息
3. 在 `package.json` 中新增 `config.commitizen` 字段信息，主要用于配置cz工具的适配器路径：
```javascript

"devDependencies": {
 "cz-conventional-changelog": "^2.1.0"
},
"config": {
  "commitizen": {
    "path": "./node_modules/cz-conventional-changelog"
  }
}

```
