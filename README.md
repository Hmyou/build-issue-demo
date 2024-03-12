```
node v20.5.1
pnpm v8.6.12
```

# 项目目录说明
taro init 初始化了两个项目，webpack5-sass-vue3-typescript-nutui，区别只有项目目录名不同，一个name-no-demo（不带taro）一个name-taro-demo（带taro）

# 问题
分别进入不同目录执行构建
```
cd name-taro-demo && pnpm build:weapp
cd name-no-demo && pnpm build:weapp
```

查看两个目录中dist/vendors.js文件，搜索const 空格 关键词，发现name-no-demo中比name-taro-demo的数量多很多（大部分都是vue构建出来的）

# 说明
问题出在 taro/packages/taro-webpack5-runner/src/webpack
/MiniWebpackModule.ts

https://github.com/NervJS/taro/blob/a169d74b136a2cc7864f228ea06a9edfd85cead6/packages/taro-webpack5-runner/src/webpack/MiniWebpackModule.ts#L198

```javascript
getScriptRule () {
    const { sourceDir, config } = this.combination
    const { compile = {} } = this.combination.config
    const rule: IRule = WebpackModule.getScriptRule()

    if (compile.exclude && compile.exclude.length) {
      rule.exclude = [
        ...compile.exclude,
        filename => /css-loader/.test(filename) || (/node_modules/.test(filename) && !(/taro/.test(filename)))
      ]
    } else if (compile.include && compile.include.length) {
      rule.include = [
        ...compile.include,
        sourceDir,
        filename => /taro/.test(filename)
      ]
    } else {
      rule.exclude = [filename => /css-loader/.test(filename) || (/node_modules/.test(filename) && !(/taro/.test(filename)))]
    }

    if (config.experimental?.compileMode === true) {
      rule.use.compilerLoader = WebpackModule.getLoader(path.resolve(__dirname, '../loaders/miniCompilerLoader'), {
        platform: config.platform.toUpperCase(),
        template: config.template,
        FILE_COUNTER_MAP,
      })
    }

    return rule
  }
```

- 问题1：如果业务项目目录名包含taro，构建时无论是否在node_modules中都会命中taro-webpack5-runner内部的逻辑，不会被exclude识别。（不过这个刚好是想要的效果，taro现状本身好像也可以不用列进exclude？希望有配置项可以完全自定义控制exclude和include；或者类似以前vue-cli中transpiledependencies: true全部编译）
- 问题2：当设置include时就无法使用exclude，因为taro-webpack5-runner中getScriptRule方法的else if逻辑。但实际情况会有exclude项目内部sdk.min.js文件的同时，再配置include某个具体依赖包