# gulp 食用方法

gulpjs 是一个前端构建工具，与 gruntjs 相比，gulpjs 无需写一大堆繁杂的配置参数，API 也非常简单，学习起来很容易.不同于 grunt,gulpjs 使用的是 nodejs 中 stream 来读取和操作数据，其速度更快。

## 安装

```bash
npm install -g gulp
```

全局安装 gulp 后，还需要在每个要使用 gulp 的项目中都单独安装一次。把目录切换到你的项目文件夹中，然后在命令行中执行：

```bash
npm install gulp
```

如果想在安装的时候把 gulp 写进项目 package.json 文件的依赖中，则可以加上--save-dev：

```bash
npm install --save-dev gulp
```

gulp 入口文件 gulpfile.js

需要注意的是 gulp4.0 变动较大,如

- gulp4.0 以前需要使用 task 定义任务,4.0 之后导出函数即可作为任务

- gulp4.0 不再支持同步任务,需要调用回调函数或其他方式标记任务完成

```javascript
exports.foo = (done) => {
  console.log(start);
  done();
};
```

此时输入`gulp foo`即可执行任务

使用`exports.default`可以定义默认认为,运行时不需要指定任务名称

## gulp 的组合任务

使用 series, parallel 分别开启串行和并行任务,可以相互嵌套的使用

```javascript
const { series, parallel }

const task1 = () => { /* do something */ }
const task2 = () => { /* do something */ }
const task3 = () => { /* do something */ }

exports.foo = series(task1, task2, task3)   // 所有任务会按顺序执行
exports.bar = parallel(task1, task2, task3)   // 所有顺序会同时执行
```

上文说到 gulp4.0 以后不再支持同步模式,gulp 中有三种方式指定任务完成

- 通过调用任务函数中的回调函数参数,也可以在函数参数中传递错误

```javascript
exports.foo = (done) => {
  done(new Error('task failed))
}
```

- 返回 Promise,也可以采用 async await 语法糖

```javascript
exports.promise = () => {
  return Promise.resolev(); // 不需要返回任何值,gulp会忽略
  // 也可以抛出错误
  return new Promise.reject(new Error("task failed"));
};
```

- 返回文件流

```javascript
exports.stream = (done) => {
  const readStream = fs.createReadStream("package.json"); // 创建文件读取流
  const writeStream = fs.createWriteStream("tmp.txt"); // 创建文件写入流
  readStream.pipe(writeStream);
  return readStream;

  // 其实gulp内部会监听文件流的end事件,一旦写入成功,调用done标记任务结束
  readStream.on("end", () => {
    done();
  });
};
```

## gulp 工作原理

gulp 是基于 node.js 中的 stream 完成数据读写与转换工作,核心原理就是创建文件读取流,文件转换流,文件写入流

通过一系列的插件,将读取到的文件流通过插件,pipe(导入,类似于数据从管道中加工一样)到插件的转换流中,最终将转换过的流 pipe 进写入流中

```javascript
const fs = require('fs')
const { Transform } = require('stream')

exports.default = () => {
  const readStream = fs.createReadStream('index.css')    // 创建文件读取流
  const writeStream = fs.createWriteStream('index.min.css')   // 创建文件写入流
  const trasnform = new Transform({   // 创建文件转换流
    trasnform: (chunk, encoding, callback) {
      // chunk: 读取流中的内容(Buffer)
      // do something, such as minify css
    }
  })
  readStream
    .pipe(trasnform)
    .pipe(writeStream)
  return readStream

  // 本示例中,这样通过gulp的自动化处理,就能将css文件进行压缩
}
```

gulp 中已经提供了包括文件读取流 gulp.src,写入流 gulp.dest,转换流(gulp 插件)的功能,基于 gulp 可以很轻松的完成前端构建过程中的任何操作

如,使用 gulp 插件完成压缩 css 的任务

```javascript
const { src, dest } = require("gulp");
const cleanCss = require("gulp-clean-css");

exports.default = () => {
  return src("src/*.css").pipe(cleanCss()).pipe(dest("dist"));
};
```

命令行中输入 gulp 即可将 src 目录下的所有 css 压缩到 dist 目录下

其中gulp.src(globs[, options])需要特殊说明一下,内部使用node-glob匹配文件(类似正则), options为可选参数,通常较少配置

`* `匹配文件路径中的0个或多个字符，但不会匹配路径分隔符，除非路径分隔符出现在末尾

`** `匹配路径中的0个或多个目录及其子目录,需要单独出现，即它左右不能有其他东西了。如果出现在末尾，也能匹配文件。

`? `匹配文件路径中的一个字符(不会匹配路径分隔符)

`[...] `匹配方括号中出现的字符中的任意一个，当方括号中第一个字符为^或!时，则表示不匹配方括号中出现的其他字符中的任意一个，类似js正则表达式中的用法



## gulp 插件

gulp 插件为以`gulp-开头的`npm 模块

### 样式编译

scss 使用 gulp-sass,less 使用 gulp-less

需要注意的是安装 gulp-sass 依赖于 node-sass,node-oss 是一个 c++模块,由于网络原因很难下载下来,需要单独配置镜像源

```bash
vim ~/.npmrc

## 添加镜像源
sass_binary_site=https://npm.taobao.org/mirrors/node-sass
```

```javascript
const gulp = require("gulp"),
  sass = require("gulp-sass");

const compileSass = () => {
  // 提供src的第二个参数,指定base,可以将输出的文件与源文件保持相同的目录结构;另外一个cwd也比较常用,表示src寻找路径时,从哪个路径下开始寻找
  // 另外,gulp-sass不会去转换以_开头的scss文件,会被认为这是其他scss所依赖的文件
  return gulp
    .src("src/assets/styles/*.sass", (base: "src"))
    .pipe(sass({ outputStyle: "expanded" })) // 默认右括号不会折行
    .pipe(gulp.dest("dist/css"));
};
```

执行`yarn gulp compileSass`,src/assets/style下的所有非_开头的sass文件会被编译成css文件到dist目录下

### 脚本编译

使用插件`gulp-babel`

需要同时下载`gulp-babel`, `@babel/core`,相应的 presets 如`@babel/preset-env`,此 preset 会将所有最新的 ECMAScript 新特性做出转换

因为 babel 本身只是一个负责转换 ECMAScript 的平台,具体的实施是通过插件去实现的,不清楚 presets 使用的见这里

```javascript
var gulp = require('gulp'),

const compileJs = () => {
  return gulp
    .src("src/assets/scripts/*.js", { base: "src" })
    .pipe(babel({ presets: ["@babel/preset-env"] }))
    .pipe(dest("dist"));
};
```

同理，执行`yarn gulp compileJs`,此时src/assets/scripts目录下所有的js文本会经过babel转换，将最新的ECMAScript特性转化为ES5代码到dist目录下

### 模板文件编译

根据模板引擎的不同,选择对应的插件,如 gulp-swig,gulp-front-matter,gulp-template 等

```javascript
var gulp = require('gulp'),
const swig = require("gulp-swig");

const data = {}; // 模板需要的数据

const compileTemplate = () => {
  return gulp
    .src("src/*.html", { base: "src" })
    .pipe(swig({ data }))
    .pipe(dest("dist"));
};
```

### 重命名

使用 `gulp-rename`

用来重命名文件流中的文件。用 gulp.dest()方法写入文件时，文件名使用的是文件流中的文件名，如果要想改变文件名，那可以在之前用 gulp-rename 插件来改变文件流中的文件名。

```javascript
var gulp = require('gulp'),
    rename = require('gulp-rename'),
    uglify = require("gulp-uglify");

gulp.task('rename', function () {
    gulp.src('js/jquery.js')
    .pipe(uglify())  //压缩
    .pipe(rename('jquery.min.js')) //会将jquery.js重命名为jquery.min.js
    .pipe(gulp.dest('js'));

    //关于gulp-rename的更多强大的用法请参考https://www.npmjs.com/package/gulp-rename

```

### 图片和字体文件转换

可以使用`gulp-imagemin`插件来压缩 jpg、png、gif 等图片。需要注意此模块内部也依赖于一些c++模块,下载比较困难

图片是无损的压缩,只是删除了一些元数据的信息,只有svg的字体文件可以通过`gulp-imagemin`进行压缩

```javascript
var gulp = require('gulp');
var imagemin = require('gulp-imagemin');
var pngquant = require('imagemin-pngquant'); //png图片压缩插件

const image = () => {
  return gulp.src('src/images/*')
      .pipe(imagemin({
          // 可以传入参数指定压缩图片的插件或者不传
          progressive: true,
          use: [pngquant()] // 使用pngquant来压缩png图片
      }))
      .pipe(gulp.dest('dist'));
  
}

```

gulp-imagemin的使用比较复杂一点，而且它本身也有很多插件，建议去它的项目主页看看文档

### 清理文件/文件夹

可以使用gulp插件`gulp-clean`或者非gulp插件`del`执行自定义操作

需要注意如果使用`gulp-clean`,需要配置src的allowEmpty为true,以防止需要清除的目录不存在而报错终止

```javascript
const gulp = require('gulp')
const del = require('del')

const clean = () => {
  return del(['dist'])    // del方法返回Promise,可以作为任务结束标志
}
```

### 自动加载插件

`gulp-load-plugins`这个插件能自动帮你加载package.json文件里的gulp插件,并且自动转换为驼峰命名,不用显式的去require每一个gulp插件

```javascript
const gulp = require('gulp');
//加载gulp-load-plugins插件，并马上运行它
const plugins = require('gulp-load-plugins')();
```

然后我们要使用gulp-rename和gulp-ruby-sass这两个插件的时候，就可以使用plugins.rename和plugins.rubySass来代替了,也就是原始插件名去掉gulp-前缀，之后再转换为驼峰命名。

 ### 其他插件

 代码复用: `gulp-file-include`, 自动升级版本号: `gulp-bump`

 ## 开发服务器

 使用模块browser-sync,提供一个web服务器,支持热更新

 ```javascript
const browserSync = require('browser-sync')
const br =browserSync.create()

 const server = () => {
  br.init({
    notify: false,    // 是否打开提示窗
    port: 8999,
    open: false,    // 是否默认打开浏览器
    files: 'dist/**',      // 监听的路径通配符,文件发生改变后,自动刷新浏览器;也可以使用br实例的reload方法
    server: {
      baseDir: 'dist',
      // 优于baseDir配置,假如html中有引入node_modules下的样式或脚本,需要根据routes映射路由;但是,只是在开发环境中有效,打包过后,dist目录下仍没有node_modules中的文件,这个后续再说
      routes: {
        '/node_modules': 'node_modules'
      }
    }
  })
}
 ```

 ## gulp.watch

 `gulp.watch()`用来监视文件的变化,当文件变化后,可以利用它执行相应的任务

 gulp.watch(glob[, opts], tasks)

 **glob**为要监视的文件匹配模式,可以传递字符串或者数组
 **opts**为一个可选的配置对象
 **tasks**为文件变化后要执行的任务,数组形式

 此外,gulp.watch还有另外一种用法

 gulp.watch(glob[, opts, cb])

cb为函数,表示监视的文件发生变化时,会调用这个函数;函数接收一个对象参数,该对象包含了文件变化的一些信息

`type`为属性变化的类型,可以是`added`, `changed`, `deleted`, `path`

需要注意的是,如果你使用了swig模板引擎,当你修改到模板源代码时,swig模板引擎的缓存机制会导致页面修改不生效
此时需要将swig选项中cache设置为false

另外,在开发过程中,有些文件不需要做出更改后的反馈,如图片,纯静态资源等,这些会加大开销,为优化
我们可以指定br.init()中server的baseDir参数,可以传递一个数组,当前一个元素找不动文件时,会去后后续路径寻找

## useref文件处理

通过构建注释可以完成引用文件的打包,合并,压缩,针对的是dist中文件而不是src中的文件

```html
  <!-- build:css assets/styles/vendor.css -->
  <link rel="stylesheet" href="/node_modules/..">
  <!-- endbuild -->
  <!-- build:css assets/styles/main.css -->
  <link rel="stylesheet" href="assets/styles/main.css">
  <!-- endbuild -->
```

配置useref

```javascript
const useref = () => {
  return src('dist/*.html', { base: 'dist' })
    .pipe(plugins.useref({ searchPath: ['dist', '.'] }))
    .pipe(dest('dist'))
}
```

其中searchPath中配置的选项表示模板文件中的引用路径首先会在dist目录下去寻找,其次在项目根目录下寻找

编译后的结果为模板文件中的引入路径会被替换为相应路径,且注释会被移除掉,如

```html
  <link rel="stylesheet" href="asset/styles/vendor.css">
```

编译后的css或js文件,是合并过的文件,但是未经压缩,我们可以这样

经过上述useref处理,未经压缩的文件为html,css,js,因此对应着三种文件的压缩

```bash
yarn add gulp-htmlmin gulp-uglify gulp-clean-css --dev
```

此处面临的问题是我们的读取流为三种格式的文件,需要分别进行不同的操作,可以借助`gulp-if`插件

```bash
yarn add gulp-if --dev
```

```javascript
const useref = () => {
  return src('dist/*.html', { base: 'dist' })
    .pipe(plugins.useref({ searchPath: ['dist', '.'] }))
    .pipe(plugins.if(/\.js$/), uglify())
    .pipe(plugins.if(/\.css$/), cleanCss())
    .pipe(plugins.if(/\.html$/), htmlmin())
    .pipe(dest('dist'))
}
```
这里要注意必须先执行一次编译命令再执行useref,因为此前构建时构建注释已经被删除

执行以上任务时,你会发现有些文件没有写入成功,这是因为这里的读取流和写入流都是dist目录下的,同时读写,产生冲突,可能导致有些内容写不进去的情况
我们需要将开发环境过程中的构建后的文件写入另外的路径,只有build后的文件才放入dist目录

## 封装工作流

如何提取一个可复用的自动化构建工作流





