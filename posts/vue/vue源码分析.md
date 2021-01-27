# vue源码分析

先来定个小目标,阅读源码的过程必然是枯燥的,没有目标的阅读将是无章可循

- Vue.js的静态成员和实例成员是怎么初始化的
- 首次渲染的过程
- 数据响应式原理

克隆下来vue代码

```bash
git clone https://github.com/vuejs/vue
```

可以看到项目结构如下
<pre>.
├── BACKERS.md
├── LICENSE
├── README.md
├── benchmarks
│   ├── big-table
│   ├── dbmon
│   ├── reorder-list
│   ├── ssr
│   ├── svg
│   └── uptime
├── dist      打包的各种结果版本
│   ├── README.md
│   ├── vue.common.dev.js
│   ├── vue.common.js
│   ├── vue.common.prod.js
│   ├── vue.esm.browser.js
│   ├── vue.esm.browser.min.js
│   ├── vue.esm.js
│   ├── vue.js
│   ├── vue.min.js
│   ├── vue.runtime.common.dev.js
│   ├── vue.runtime.common.js
│   ├── vue.runtime.common.prod.js
│   ├── vue.runtime.esm.js
│   ├── vue.runtime.js
│   └── vue.runtime.min.js
├── examples      示例
│   ├── commits
│   ├── elastic-header
│   ├── firebase
│   ├── grid
│   ├── markdown
│   ├── modal
│   ├── move-animations
│   ├── select2
│   ├── svg
│   ├── todomvc
│   └── tree
├── flow
│   ├── compiler.js
│   ├── component.js
│   ├── global-api.js
│   ├── modules.js
│   ├── options.js
│   ├── ssr.js
│   ├── vnode.js
│   └── weex.js
├── package.json
├── packages
│   ├── vue-server-renderer
│   ├── vue-template-compiler
│   ├── weex-template-compiler
│   └── weex-vue-framework
├── scripts
│   ├── alias.js
│   ├── build.js
│   ├── config.js
│   ├── feature-flags.js
│   ├── gen-release-note.js
│   ├── get-weex-version.js
│   ├── git-hooks
│   ├── release-weex.sh
│   ├── release.sh
│   └── verify-commit-msg.js
├── src
│   ├── compiler    将模板编译成render函数
│   ├── core        vue的核心部分
│   ├── platforms   和平台相关的代码,包括web和weex
│   ├── server      服务端渲染
│   ├── sfc         单文件组件
│   └── shared
├── test
│   ├── e2e
│   ├── helpers
│   ├── ssr
│   ├── unit
│   └── weex
├── types
│   ├── index.d.ts
│   ├── options.d.ts
│   ├── plugin.d.ts
│   ├── test
│   ├── tsconfig.json
│   ├── typings.json
│   ├── umd.d.ts
│   ├── vnode.d.ts
│   └── vue.d.ts
└── yarn.lock
</pre>

vue2.x使用的静态类型检查器是flow,vscode中可以通过相关插件来使用flow进行类型检查,flow也提供了cli工具

在文件开头通过`// @flow`或`/* @flow */`即可开启flow的类型检查
```javascript
/* @flow */
function sum(a: number, b: number): number {
  return a + b
}
sum(2 + '2')

```

## 调试前的准备

在Vue.js中,使用的打包工具为rollup,rollup适合那些第三方库和框架的打包,只处理js文件,更轻量级,React.js同样也是使用的rollup打包,它不会像webpack打包那样生成冗余代码.关于rollup的使用,可以查看我的这一篇文章

- 安装依赖,有些包可能需要挂梯子才能成功下载

```bash
npm  i
```

- 设置sourcemap,方便我们调试

在根目录下package.json中的dev脚本添加参数 --sourcemap

- 执行`npm run dev`, 使用rollup打包, -w参数监听文件变化自动执行打包; -c设置配置文件, --environment指定环境变量从而生成不同的打包结果

```JSON
{
"dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev"
}    
```

此时dist目录下会生成vue.js和vue.js.map文件

选择example目录下的grid项目进行测试,浏览器打开index.html,可以看到source面板如下

![](https://cdn.jsdelivr.net/gh/lovelyJason/cdn-gallery/img/tp/20210113172442.png)

当开启了sourcemap会显示源代码所在路径,即src目录;如果移除掉vue.js文件末尾的sourcemap引用,此时调试面板不会出现src目录

开启sourcemap需要浏览器支持,souremap文件中可以定义源代码所在的路径,如axios.min.map中将源码映射到webpack://./lib目录下.所以我们有时候在一个简单的项目中调试axios,都没引入webpack但是sourcemap中有了webpack的字样,就是这个原理

soucemap更多内容可参考阮一峰文章

前面说到vue2.x是采用的flow进行类型检查,而在js中采用类型注解的写法,vscode中是会显示警告波浪线,为了解决语法报错,需要在vscode中添加配置项`"javascript.validate.enable": true`,不检查js的语法问题.如果vscode有安装过eslint插件,并设置保存校验时也会提示报错,也需关掉

另外js中如果有泛型等复杂的写法,如

```javascript
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }
```
由于js中没有这种写法,导致后续代码丢失高亮效果,安装`Babel JavaScript`插件会解决该问题,但f12跳转函数定义处仍然不可用,这个问题就无法解决了

## 不同构建版本Vue的区别

这里先贴一下Vue官方的详细解释

<https://cn.vuejs.org/v2/guide/installation.html#%E6%9C%AF%E8%AF%AD>

在vue源码项目下运行`npm run build`即可打包各种构建版本的vue到dist目录下

关于CommonJS和ES6模块之间的差异参考<>

使用@vue/cli创建的项目,使用的vue 版本为vue.runtime.esm.js
关于这一点可以查看其中webpack的配置,使用`vue inspect > output.js`可以将webpack配置项输出到output.js中方便查看,需要注意的是此配置不是可用的webpack配置,不能作为配置文件使用

------

大体看一下vue在哪些地方导出过,观察项目结构,不难发现,导出vue的地方一共有四处,分别是

- 与平台无关的
  src/core/index.js:
  初始化Vue静态成员

  src/core/instance/index.js:
  定义Vue构造函数,混入Vue的实例成员

- 与平台相关的
  src/platforms/web/runtime/index.js:
  给Vue.config注册了平台相关的方法,指令,组件,以及给原型挂载__patch(),$mount()方法
  __patch__()方法是将虚拟dom转化为真实dom,$mount是将dom进行挂载到界面上来

  src/platforms/web/entry-runtime-with-compiler:
  打包入口,并且是包含编译器完整版的vue.在这个文件中,重写了runtime中的$mount()方法,增加了把模板编译成render函数的功能
  还给Vue增加了compiler方法,可以帮助我们手动将模板转化为render函数


```javascript
var vm = new Vue({
  el: '#app',
  msg: 'hello vue'
})
```

此时调用栈的顺序为

## 响应式原理

进入这部分源码之前先看一下以下常见问题

- 重新给属性赋值为一个对象,是否是响应式的
- 给数组元素赋值,视图是否会更新
- 修改数组length,视图是否会更新
- 数组的api如push等是否触发视图更新

init.js > initState() > initData()

在initData中调用proxy()把data成员注入vue实例,然后调用observer(data, true)

在defineReactive给属性定义了getter和setter,getter中进行依赖收集,在我们访问这个属性的时候会进行依赖收集,

实际上就是把依赖该数据的watcher对象添加到Dep的subs数组中,将来数据发生变更时,会调用Dep的notify()方法通知所有Watcher更新


