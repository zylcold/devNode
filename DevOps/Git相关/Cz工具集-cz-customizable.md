### cz-customizable
 [cz-customizable](https://github.com/leonardoanalista/cz-customizable) 适配器： 

> Suitable for large teams working with multiple projects with their own commit scopes. When you specify the scopes in your .cz-config.js, cz-customizable allows you to select the pre-defined scopes. No more spelling mistakes embarrassing you when generating the changelog file.

安装适配器

```shell
npm install cz-customizable --save-dev
```

将之前符合Angular规范的 **cz-conventional-changelog** 适配器路径改成 **cz-customizable** 适配器路径： 

```javascript
"devDependencies": {
  "cz-customizable": "^5.3.0"
},
"config": {
  "commitizen": {
    "path": "node_modules/cz-customizable"
  }
}
```
