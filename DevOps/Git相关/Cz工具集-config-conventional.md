@commitlint/config-conventional

符合Angular风格的校验规则

```shell
npm install --save-dev @commitlint/config-conventional 
```

在项目中新建 `commitlint.config.js` 文件并设置校验规则： 
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional']
};
```