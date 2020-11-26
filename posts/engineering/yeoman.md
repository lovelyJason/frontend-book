# yeoman

## 安装

使用yoman,要搭配特定的generator才能使用
```bash
yarn global add yo
yarn global add generator-node
yo node
```

当需要在已有的项目基础之上创建一些文件,可以使用sub generator

```bash
# cd 项目目录
yo node:cli
```

## 脚手架原理

启动脚手架应用之后,询问预设的问题,将得到的答案结合模板文件生成项目结构

首先要在package.json文件中添加bin字段,用于指定cli的入口文件,需要注意,如果是linux或macos环境下,需要指定入口文件的755权限

cli入口文件的开头要指定解释器,如`#!/bin/env node`

下载inquirer模块,这是node中发起命令行交互的模块

新建模板文件,可以使用ejs语法插入内容

```javascript
inquirer.prompt([{
  type: 'input',
  name: 'name',
  messge: 'project name'
}]).then(answers => {
  const tmplDir = Path.join(__dirname, 'templates')
  const destDir = process.cwd()
  // 将模板文件全部转换到目标路径
  fs.readdir(tmplDir, (err, files) => {
    if(err) throw err
    files.forEach(file => {
      ejs.renderFile(path.join(tmplDir, file), anwsers, (err, result) => {
        if(err) throw err
        fs.writeFileSync(path.join(destDir, file), result)
      })
    });
  })
})
```

## 基于yeoman创建generator

- 文件结构: generators > app > index.js

index.js
```javascript
const Generator = require('yeoman-generator')

module.exports = class extends Generator {
  prompting () {
    return this.prompt([{
      type: 'input',
      name: 'name',
      messge: 'message',
    }]).then(answers => {
      this.answers = answers
    })
  }
  writing () {
    const templates = []    // 模板文件路径的集合
    templates.forEach(val => {
      this.fs.copyTpl(
        this.templatePath(val),   // 模板文件路径
        this.destinationPath(val),    // 输出目录路径
        this.answers      // 模板数据上下文
      )
    })
  }
}
```

注意: 模板默认使用ejs模板标记输出数据,如果想原封不动输出ejs语法,需要添加%转义


