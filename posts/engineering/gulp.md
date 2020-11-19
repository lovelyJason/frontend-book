# gulp食用方法

gulpjs是一个前端构建工具，与gruntjs相比，gulpjs无需写一大堆繁杂的配置参数，API也非常简单，学习起来很容易.不同于grunt,gulpjs使用的是nodejs中stream来读取和操作数据，其速度更快。

## 安装

```bash
npm install -g gulp
```
全局安装gulp后，还需要在每个要使用gulp的项目中都单独安装一次。把目录切换到你的项目文件夹中，然后在命令行中执行：
```bash
npm install gulp
```
如果想在安装的时候把gulp写进项目package.json文件的依赖中，则可以加上--save-dev：
```bash
npm install --save-dev gulp
```

gulp入口文件gulpfile.js

需要注意的是gulp4.0变动较大,如
- gulp4.0以前需要使用task定义任务,4.0之后导出函数即可作为任务

- gulp4.0不再支持同步任务,需要调用回调函数或其他方式标记任务完成

```javascript
  exports.foo = (done) => {
    console.log(start)
    done()
  }
```

此时输入`gulp foo`即可执行任务

使用`exports.default`可以定义默认认为,运行时不需要指定任务名称

## gulp的组合任务

使用series, parallel分别开启串行和并行任务

```javascript
const { series, parallel }

const task1 = () => { /* do something */ }
const task2 = () => { /* do something */ }
const task3 = () => { /* do something */ }

exports.foo = series(task1, task2, task3)   // 所有任务会按顺序执行
exports.bar = parallel(task1, task2, task3)   // 所有顺序会同时执行
```

上文说到gulp4.0以后不再支持同步模式,gulp中有三种方式指定任务完成

- 通过调用任务函数中的回调函数参数,也可以在函数参数中传递错误
```javascript
exports.foo = (done) => {
  done(new Error('task failed))
}
```
- 返回Promise,也可以采用async await语法糖
```javascript
exports.promise = () => {
  return Promise.resolev()    // 不需要返回任何值,gulp会忽略
  // 也可以抛出错误
  return new Promise.reject(new Error('task failed'))
}
```
- 返回文件流
```javascript
exports.stream = (done) => {
  const readStream = fs.createReadStream('package.json')    // 创建文件读取流
  const writeStream = fs.createWriteStream('tmp.txt')     // 创建文件写入流
  readStream.pipe(writeStream)
  return readStream

  // 其实gulp内部会监听文件流的end事件,一旦写入成功,调用done标记任务结束
  readStream.on('end', () => {
    done()
  })
}
```

## gulp工作原理

gulp是基于node.js中的stream完成数据读写与转换工作,核心原理就是创建文件读取流,文件转换流,文件写入流

通过一系列的插件,将读取到的文件流通过插件,pipe(导入,类似于数据从管道中加工一样)到插件的转换流中,最终将转换过的流pipe进写入流中

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

gulp中已经提供了包括文件读取流gulp.src,写入流gulp.dest,转换流(gulp插件)的功能,基于gulp可以很轻松的完成前端构建过程中的任何操作

如,使用gulp插件完成压缩css的任务

```javascript
const { src, dest } = require('gulp')
const cleanCss = require('gulp-clean-css')

exports.default = () => {
  return src('src/*.css')
    .pipe(cleanCss())
    .pipe(dest('dist'))
}
```

命令行中输入gulp即可将src目录下的所有css压缩到dist目录下




