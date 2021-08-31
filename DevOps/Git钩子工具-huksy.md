### 安装huksy

```
npm install husky --save-dev
```

在package.json中配置 `git commit` 提交时的校验钩子： 

```
"husky": {
  "hooks": {
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
  }  
}
```