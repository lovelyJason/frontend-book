# webpack

webpack 4.x以后支持零配置使用webpack,默认会将src/index.js 打包到dist/main.js中

entry必须为相对路径,output.path必须为绝对路径,publicPath的末尾/不能省略,如dist/

## webpack打包后的代码分析

// TODO

如果代码不能折叠,打开vscode的设置,搜索folding,将折叠策略从indentation改为auto

注意webpack-cli 3.x版本打包后的结果中自调用函数参数modules为数组,4.x为对象

## loader

常用loader分类

- 编译转换,如css-loader
- 文件操作,如file-loader
- 代码检查,如代码风格检查类

### file-loader

文件资源加载器file-loader,拷贝资源到输出目录

### data urls和url-loader

`data:[<mediatype>][;base64],<data>`

`<data>指文件内容`,如

`data:text/html;charset=UTF-8,<h1>hello world</h1>`

使用url-loader需要同时下载file-loader,因为大的文件还是会采取file-loader打包

------
需要注意的是webpack不会转换ES6特性,之所以处理import和export是因为模块打包需要
需要配置babel-loader平台 + 不同的插件

```javascript
{
  test: /\.js$/,
  use: {      // use也可以指定为一个路径,如自定义的loader路径
    loader: 'babel-loader',
    options: {
      presets: ['@babel/preset-env']
    }
  }
}
```

## webpack加载资源的方式

- ES6 module的import和export
- CommonJS
  需要注意使用require导入一个默认模块需要require('...').default
- AMD的define和require

loader加载的非JavaScript也会触发资源加载,如css中@import和url函数,html标签的src属性

html-loader会转换html文件并且默认会加载img的src属性,如果需要加载a标签的href属性,需要配置loader选项

```javascript
{
  test: /\.html/,
  use: {
    loader: 'html-loader',
    options: {
      attrs: ['a:href']
    }
  }
}
```

## 开发一个markdown-loader

loader实际上是一组管道

![image-20201218174314460](/Users/jasonhuang/Library/Application Support/typora-user-images/image-20201218174314460.png)

规则: 导出一个函数,函数接收输入,返回输出,且返回数据,如果是普通的字符串,则要交给下一个loader,否则必须返回javascript代码形式的字符串

```javascript
const marked = require('marked')

module.exports = source => {
  const html = marked(source)
  return `module.exports = ${JSON.stringify(html)}`
  // return `export default {JSON.stringify(html)}`
}
```

## webpack插件机制

loader专注于资源的加载,而插件解决的是其他自动化操作

- clean-webpack-plugin
  清除目录
  
- html-webpack-plugin
  自动生成html

  此插件会在输出目录中生成一个html替换过变量后的文件
  
  可以通过简单的选项配置替换html中的内容,也可以制定模板,用于高度自定义
```javascript
plugins: [
  new HtmlWebpackPlugin({
    title: '',
    meta: {
      viewport: 'width=device-width'
    },
    template: './src/index.html'
  })
]
```

src/index.html
```html
<div class="container">
  <h1><%= htmlWebpackPlugin.options.title %></h1>
</div>
```

htmlWebpackPlugin是插件提供的预置变量,也可以通过另外的属性添加自定义变量

同时输出多个页面文件,可以实例化多个实例

- copy-webpack-plugin
传入数组指定拷贝的路径,可以是通配符/路径

通常不在开发环境中使用,而是在上线之前中使用.开发环境中访问静态资源通常是配置devServer的contentBase
```javascript
new CopyWebpackPlugin([
])
```

需要注意,假如要把public下的文件拷贝至输出目录,如果包含一个index.html,和html-webpack-plugin如果生成的路径一致,会造成覆盖

### 插件原理

通过webpack在生命周期的钩子中挂载函数实现自定义逻辑

webpack规定插件必须是一个函数或者一个包含apply方法的对象

```javascript
class MyPlugin {
  apply (compiler) {
    // tap 注册钩子函数
    compiler.hooks.emit.tap('MyPlugin', compilation => {
      // compilation为此次打包的上下文
      // compilation.assets为此次打包的资源对象,每个资源的source方法返回当前资源的内容
      for(const name in compilation.assets) {
        if(name.endsWith('.js')) {
          const contents = compilation.assets[name].source()
          const withoutComments = contents.replace(/\/\*+\*\//g, '')
          compilation.assets[name] = {
            source: () => withoutComments,
            size: () => withComments.length   // webpack要求
          }
        }
      }
    })
  }
}
```

## webpack开发优化

### 监听文件自动执行打包

通过webpack --watch选项

### 自动刷新浏览器

- 使用工具browser-sync

```bash
browser-sync dist --file "**/*"
```

效率比较低,频繁的磁盘操作

**webpack-dev-server**

访问静态资源时,通常开发环境中不去使用拷贝插件,而是配置devServer的contentBase

### 代理Api

```javascript
devServer: {
  proxy: {
    '/api': {
      // 此时http://localhost:3000/api/users会被代理到http://xxx.com/api/users
      target: 'http://xxx.com',
      // 重写代理路径
      pathRewrite: {
        '^api': ''
      },
      changeOrigin: true

    }
  }
}
```

### Source Map

js中加载source map的语法,注意很多库源码中已去除了此注释

```javascript
//# sourceMappingURL=jquery-3.4.min.map
```

此时开发人员工具开启了Source Map功能后,浏览器就会请求对应的sourcemap文件,逆向解析出来源代码,就可以调试非压缩的源代码

webpack中配置Source Map,通过配置项devtool

![image-20201221135225631](/Users/jasonhuang/Library/Application Support/typora-user-images/image-20201221135225631.png)

- eval: 不生成map文件,将打包后的每个模块中的代码放到eval中，并且添加source map路径

如`eval("...//# sourceaURL=webpack://webpack-demo/./src/main.js?")`

此配置只能定位到代码所在文件，而不能定位行数和列数

> webpack配置文件可以导出一个数组,数组中每个元素为一个打包配置,从而可以根据多套配置进行不同的打包任务

- hidden-source-map: 生成source map文件,但代码中不通过注释引入
- eval-source-map: 生成了souce map,可以定位行数和列数,使用行内方式嵌入source map
- cheap-eval-source-map: 阉割版的eval-source-map,只能定位行数,会经过loader转换
  如将const变为var
- cheap-module-eval-source-map: 没有经过loader加工过的源代码,而不带module是加工后的结果,会将es6特性进行转换
- inline-source-map: source map文件以dataURL形式嵌入到代码当中,代码文件会变大很多
- nosources-source-map: 点击错误进入后,看不到代码段,但是能定位到代码的行数和列数,方便生产环境保护源代码

总的来说

- eval: 是否使用eval执行代码
- cheap: 是否包含行信息
- module: 是否能得到loader转换前的代码

#### 使用选择

**开发模式**

`cheap-module-eval-source-map`

- 保证自己一行代码数不要过长,只需要定位到行即可
- 经过loader转换后的代码差异较大
- 首次打包虽然比较慢,但是重写打包比较快

**生产模式**

`none`或者`nosources-source-map`

不会暴露源代码

### HMR

webpack-dev-server已经集成了,运行时指定--hot选项或者配置webpack的配置文件

```javascript
const webpack = require('webpack')

module.exports = {
  devServer: {
    hot: true
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin()
  ]
}
```

需要注意修改js文件仍会导致浏览器刷新,这是因为webpack中的HMR并不是开箱即用,需要手动处理模块热替换逻辑

之所以css中有热更新效果,是因为style-loader中已经处理了这部分逻辑,而js的逻辑没有共通规律

module.hot.accept用于注册模块更新后的处理函数,第一个参数接收模块路径,第二个参数接收处理函数
```javascript
module.hot.accept('./a.js', () => {
  console.log('a模块热更新了')
})
```

注意问题

- 处理HMR的代码报错会导致自动刷新,难以发现错误,将hot: true改为hotOnly: true
- 使用module.hot的api应判断此条件是否存在,否则如果没引用HRM插件会报错

### webpack不同环境下的配置

- 根据环境导出不同的配置
```javascript
module.exports = (env, args) => {
  // env环境参数
  if(env === 'production') {
    // ...
  }
}
```

```bash
yarn webpack --env prodution
```
- 一个环境对应一个配置

使用`webpack-merge`合并配置文件

### Tree Shaking

生产模式下即mode为`production`时会自动开启,构建后的代码会移除掉dead-code

```javascript
optimization: {
  usedExports: true,
  minimize: true,   // 搭配usedExports使用
  // 普通的模块打包,会将每个模块放到一个单独函数中
  concatenateModules: true        // Scope Hoisting: 尽可能将所有模块合并输出到一个函数中
}
```

#### Tree Shaking与babel

webpack打包的代码必须使用esm,而babel-loader会将代码从esm转换为commonjs(如果使用了babel的插件如`@babel/preset-env`),从而导致tree shaking失效

而最新版的babel-loader已经关闭了esm转换的插件,tree shaking仍然是可用的,当配置了babel强制转换commonjs,此时tree shaking会失效

```javascript
module.exports = {
  module: {
    rules: [
      test: /\.js$/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: [
            [ '@babel/preset-env', modules: 'commonjs' ]
          ]
        }
      }
    ]
  }
}
```

### sideEffects

副作用: 模块执行时除导出成员外做的事情,一般用于npm包标记是否有副作用

当在index.js中集中导入了其他模块,然后在别的地方引入此处的index.js中某一个模块,会将index.js中引入的所有模块都会打包进去

```javascript
module.exports = {
  optimization: {
    sideEffects: true   // 生产模式下自动开启,指的是功能
  }
}
```

开启此特性之后,webpack在打包之前会先检查当前项目package.json中有没有sideEffect标识(可以为字符串或者数组),如果没有副作用即为false,即表示当前package.json所在项目当中所有代码都没有副作用

> 注意

使用sideEffects时,确保代码真的没有副作用

### 代码分割Code Splitting

#### 多入口打包

常用于多页应用.设置webpack.config.js中的entry设置为一个对象,而不是数组(如果是数组,就是把多个文件打包到一起,仍然是一个入口)

```javascript
module.exports = {
  entry: {
    index: './src/index.js',
    login: './src/login.js'
  },
  output: {
    filename: '[name].bundle.js'
  },
  plugins: [
    new  HtmlWebpackPlugin({
      // ...
      chunks: ['index']   // 设置每个页面注入哪些js文件
    })
  ]
}
```

**提取公共模块**

项目中经常会有公共模块的引用,如api,utils,甚至框架等等,如果每个用到的地方都去引用一次,将会导致应用体积巨大,提取公共模块是必要的的

```javascript
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all'     // 所有的公共模块提取到公共的bundle当中
    }
  }
}
```

#### 动态打包

动态导入的模块会被自动分包,以实现按需加载

相比于多入口的方式,动态导入更加灵活,我们可以通过代码的逻辑控制需不需要加载模块,什么时候加载

```javascript
if(/* 匹配到文章路由 */) {
  import('./posts/posts').then({ default: posts }) => {

  }
}
```

**魔法注释**

给分包产生的bundle命名,默认是序号.js

```javascript
import(/* webpackChunkName: 'posts' */).then()
```

如果几个模块的命名是相同的,则会被打包到同一个bundle中

### MiniCssExtractPlugin

提取css到单个文件,可以实现css模块的按需加载

```javascript
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = {
  module: {
    rules: [
      {
        test: '/\.css$/,
        use: [
          // 'style-loader', 此时不再需要
          MiniCssExtractPlugin.loader,
          'css-loader'
        ]
      }
    ]
  },
  plugins: [
    new MiniCssExtractPlugin()
  ]
}
```

建议,除非css文件过大,才需要提取到一个文件中,否则收益只能是适得其反

### OptimizeCssAssetsWebpackPlugin

压缩输出的CSS文件

通过以上css分包,此时以production的mode去执行打包,理论上css是要被压缩的,实际上并没有

这是因为webpack内置的压缩插件仅仅是针对于js的压缩其他资源要其他插件支持

```javascript
const OptimizeCssAssetsWebpackPlugin = require('optimize-css-assets-webpack-plugin')

module.exports = {
  plugins: [
    new OptimizeCssAssetsWebpackPlugin()
  ]
}

```

注意,此插件官方建议的配置是在opimization中,以便可以统一的进行配置

```javascript
const OptimizeCssAssetsWebpackPlugin = require('optimize-css-assets-webpack-plugin')

module.exports = {
  optimization: {
    minimizer: [
      new OptimizeCssAssetsWebpackPlugin()
    ]
  }
}

```

但是如果这种自定义的方式,js压缩又会失效,内部的js压缩插件会被覆盖掉,此时需要手动添加

```javascript
const OptimizeCssAssetsWebpackPlugin = require('optimize-css-assets-webpack-plugin')
const TerserWebpackPlugin = require('terser-webpack-plugin')

module.exports = {
  optimization: {
    minimizer: [
      new TerserWebpackPlugin()
      new OptimizeCssAssetsWebpackPlugin()
    ]
  }
}
```

此时以production的mode执行打包则会进行js和css的压缩,如果以普通方式打包,则不会开启压缩

### 输出文件名hash

生产模式下,文件名使用哈希防止文件更新后资源缓存的问题

webpack中filename和绝大部分插件的filename属性都支持hash的占位符

三种哈希
```javascript
module.exports = {
  output: {
    // 项目级别的
    filename: '[name]-[hash:8],bundle.js'   // 数字控制哈希位数
    // chunkhash, 同一路的打包文件哈希值为一样,
    // 文件内容级别的哈希,不同的文件有不同的哈希,contenthash,解决缓存问题的最好方式
  }
}
```

如果是由于引入路径的文件哈希值的变动,也会导致其自身哈希值被动的得到改变

## lint

lint,即代码检查.关于项目中的代码检查,在多人协作时显得尤为重要,而繁杂的配置让很多开发人员望而却步.

### eslint

**创建配置文件**

```bash
npx eslint --init
```

当然你也可以在项目下的package.json中指定eslintConfig字段来进行配置

交互式的选择你的eslint配置,随后会下载模块eslint,eslint-loader,eslint-plugin-vue等,如果选择了standard还会下载eslint-config-standard

.eslintrc.js
```javascript
module.exports = {
  env: {    // 环境不是互斥的
    browser: false, 
    es6: true   // 或则es2015
  },
  extends: [      // 有顺序
    'eslint:recommended',
    'standard',   // 来自于eslint-config-standard
  ]
  rules: {
    'react-jsx/uses-react': 2   // 开启设置'error'或2, 可选项有'off','warn','error'
  },
  plugins: [
    'react'     // eslint-plugin-react,然后就可以使用插件中配置的规则
  ],
  parseOptions: {     // 设置语法解析器
    ecmaVersion: 2015   // 检测语法是否可用,当前环境的具体成员是否可用要通过env配置
  },
  globals: {    // 可以使用的全局变量,如jquery

  }
}
```

如果你使用vue开发,需要用到eslint-plugin-react插件,该插件中已经导出了两个通用配置,分给为recommended和all,此时可以可以使用语法`plugin:[插件名称]/[配置名称]`继承使用

```javascript
module.exports = {
  extends: [
    'standard',   // 来自于eslint-config-standard
    'plugin:react/recommended'
  ]
}



```
**如何保持团队代码风格的统一性**

我会从以下四个层层递进的角度去分析

1. 在编辑器的角度来看:

首先我们要考虑到不同开发小伙伴使用的编辑器不同,不同编辑器对代码风格有不同的习惯,如是否最后一行留空,此时就需要editorConfig了,参考下文的editorConfig部分

2. 在编写代码的角度看

如果是vscode,安装eslint插件,如果使用vue开发,推荐安装vetur插件,然后配置vue的格式化工具为vetur

安装此插件时需要在项目下同步安装eslint依赖,插件依赖于此模块.

如果是第一次操作，代码中会有波浪线，点击快速修复，vscode会弹窗提示你是否允许eslint插件访问node_modules下的eslint模块将其作为依赖,允许即可，也可以应用到所有地方

![image-20201230165839540](/Users/jasonhuang/Library/Application Support/typora-user-images/image-20201230165839540.png)

注意eslint插件配置项随着插件版本的更新可能略有不同,且eslint只能设置保存时格式化,不能alt+shift+f,这一点和prettier有所不同

```json
{
  // "eslint.autoFixOnSave": true语法已废弃
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  // 配置需要校验的文件类型
  "eslint.validate.probe": [
    "vue",
    "javascript"
  ],
  "eslint.codeActionsOnSave.mode": "problems",
  "editor.formatOnSave": false      // 关闭vscode插件的保存格式化
}

```

需要注意,需要关闭`editor.formatOnSave`,如果你使用了vscode的内置格式化或者prettier的格式化,并设置了保存自动格式化之后,会出现保存以后修复两次,如果和eslint的规则冲突,还会显示警告,因此以上的配置应该被保存到工作区当中,即.vscode/settings.json中,并添加git追踪

此时你编写的代码中,只要是不符合eslint规则的会给出智能提示,保存时会自动修复能被修复的部分

另外standard官方也提供了cli和vscode插件,个人不建议安装,插件过多可能互相影响,冲突,eslint本身已经够用


3. 在webpack构建的角度来看:

如果你使用官方的cli工具`@vue/cli`和`create-react-app`,生成项目结构时会以交互式的方式询问你项目的配置,如对于@vue/cli 4.x版本,`vue create [project]`时,有以下询问

`Pick additional lint features: Lint on save, Lint and fix on commit`

`Lint on Save`表示是否在文件保存的时候lint代码,只是lint,不是修复,如果出现了不符合规范的代码,则会在控制台警告或者报错,需要自己手动修复,或者使用eslint --fix修复;如果使用eslint-loader来启动eslint,设置`fix: true`即可在保存代码触发webpack重新构建的同时进行fix格式化(非保存的概念)
注意: eslint-loader已被官方废弃,<https://npmjs.com/package/eslint-loader>,采用了插件的写法.现在的@vue/cli 4.x版本仍采用的是eslint-loader

`Lint and fix on commit`表示是否在git commit之前lint,参考以下第四点

4. 在git提交之前的角度来看:

以上两个步骤已然让你团队中的代码风格保持了通用性和一致性,然后总会有意外发生,比如有小伙伴没有安装插件,代码开发也没有按照规范化, 打包前也没有执行lint操作提提交到仓库的代码仍未进行格式化,此时就需要第三步的校验了.需要使用到的工具为`husky`和`lint-staged`.通过git hook在代码提交前强制lint

关于git hook:

即git的钩子,git操作时会触发的任务,如`pre-commit`,代表commit到本地仓库之前的钩子,通常为shell脚本

使用`git init`初始化的项目仓库.git目录中有个hooks目录,里面有git hook的编写示例

使用npm模块husky可以方便的进行git hook的配置而不需编写shell脚本

在package.json中配置

```json
"husky": {
  "scripts": {
    "lint": "eslint --ext .js,.vue src"
  },
   "gitHooks": {
    "pre-commit": "lint-staged"
  },
  "lint-staged": {
    "*.{js,jsx,vue}": [
      "vue-cli-service lint",
      "git add"
    ]
  }
}

```

此时git commit之前,会进行lint,然后再添加到暂存区,再进行commit即可

经过以上四个步骤,整个项目的风格一定是保持统一且规范化的,但是我们也不要依赖于工具,需要自己养成良好习惯,写出规范的代码


### prettier

```bash
npx prettier . --write   # 默认会将格式化后的结果输出到终端
```

整合prettier和eslint,需要做到

- 禁用eslint的formatting rules,让prettier接管
- lint执行时调用prettier格式化,再检查code-quality类规则

`eslint-plugin-prettier`: 配置eslint使用prettier对代码格式化
`eslint-config-prettier`: 关闭一些不必要的或者是与prettier冲突的lint选项

### editorconfig

控制不同编辑器的项目编码规范,优先级比编辑器自身设置要高,如果没有配置,则采用浏览器配置,多人开发项目时十分有用且必要.webstorm中默认支持,vscode需要安装插件Editconfig fom VS Code提供支持,否则默认不会直接解析.该插件会读取.editconfig中定义的规则,并覆盖user或workspace settings.json中对应配置.这里说一下user和workspace的区别,workspace的配置只会在当前项目中生效,此配置会保存于项目根目录下的.vscode/settings.json

@vue/cli初始化的项目就生成了此配置文件

**配置语法**

editorconfig配置文件采取INI格式,斜杠/作为路径分隔符,#或者;作为注释
属性不区分大小写

路径通配符规则：

| *        | 匹配除/之外任意字符串      |
| -------- | -------------------------- |
| **       | 匹配任意字符串             |
| {a, b,c} | 匹配任意给定字符串         |
| [name]   | 匹配指定字符串，如Makefile |

.editorconfig

```ini
# 最顶层配置文件,最近的配置文件拥有优先权,设为true,停止向上查找
root = true

# 表示在对应的后缀文件应用规则
[*.{js,jsx,ts.tsx,vue}]
# 编码格式
charset = utf-8
# 设置缩进风格
indent_style = space
# 设置缩进深度,如果设置以上属性为tab,则此属性默认为tab_width
indent_size = 2
# 去除换行行首的任意空白字符
trim_trailing_whitespace = true
# 文件最后一行空白行结尾
insert_final_newline = true

# 设置子目录下的规则可以覆盖上面的
[lib/**.js]
indent_style = space
# 设置确切文件
[{package.json,.travis.yml}]


```

使用注意: editconfig和prettier一样,都是用来配置格式化代码的,配置规则要和lint工具相符,否则会出现格式化代码以后不能通过lint工具校验的情况

