# webpack

webpack 4.x以后支持零配置使用webpack,默认会将src/index.js 打包到dist/main.js中

entry必须为相对路径,output.path必须为绝对路径,publicPath的末尾/不能省略,如dist/

## webpack打包后的代码分析

// TODO

如果代码不能折叠,打开vscode的设置,搜索folding,将折叠策略从indentation改为auto

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

js中加载source map的语法

```javascript
//# sourceMappingURL=jquery-3.4.min.map
```

此时开发人员工具开启了Source Map功能后,浏览器就会请求对应的sourcemap文件,就可以调试非压缩的源代码

webpack中配置Source Map,通过配置项devtool

![image-20201221135225631](/Users/jasonhuang/Library/Application Support/typora-user-images/image-20201221135225631.png)

eval: 不生成map文件,将打包后的每个模块中的代码放到eval中，并且添加source map路径

如`eval("...//# sourceaURL=webpack://webpack-demo/./src/main.js?")`

此配置只能定位到代码所在文件，而不能定位行数和列数

> webpack配置文件可以导出一个数组,数组中每个元素为一个打包配置,从而可以根据多套配置进行不同的打包任务

eval-source-map: 生成了souce map,可以定位行数和列数
cheap-eval-source-map: 阉割版的eval-source-map,只能定位行数
cheap-module-source-map: 没有经过loader加工过的源代码,而不带module是加工后的结果,会将es6特性进行转换
inline-source-map: source map文件以dataURL形式嵌入到代码当中,代码文件会变大很多
nosources-source-map: 点击错误进入后,看不到代码段,但是能定位到代码的行数和列数,方便生产环境保护源代码

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

### eslint

创建配置文件

```bash
npx eslint --init
```

交互式的选择你的eslint配置,随后会下载模块eslint,eslint-loader,eslint-plugin-vue等,如果选择了standard还会下载eslint-config-standard








