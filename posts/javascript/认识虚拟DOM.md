# 虚拟DOM

虚拟DOM(Virtual DOM),用普通的js对象描述真实DO对象

## 为什么使用Virtual DOM

- 前端开发刀耕火种的时代已经结束

- 传统的MVC框架中,模板引擎可以简化视图操作,但是没有办法跟踪状态,每次都要重新向服务器发送页面的完整请求

- 创建一个真实DOM的开销非常大,而虚拟DOM灵活,开销小得多,用它来描述真实DOM,在复杂场景中,就显得更为高效

- 虚拟DOM可以耕种状态变化,内部使用diff算法,只更新变化的部分

- 跨平台,如浏览器平台渲染,服务端渲染SSR,原生应用Weex,React Native,小程序uni-app


目前流行的虚拟DOM库有Snabbdom, virtual-dom

## Snabbdom

Vue.js 2.x内部使用的虚拟DOM基于Snabbdom改造,Snabbdom使用ts开发,通过模块进行扩展,运行速度快

snabbdom是一个底层库,不用熟练掌握snabbdom用法

### 使用

当前版本2.1.0
```bash
npm install snabbdom
```

- init: 函数,接收两个参数,第一个一个参数为数组,数组中每个元素为Snabbdom的模块,内置模块或自定义模块
  第二个参数为domApi,将虚拟dom转化为指定平台的dom
  调用init返回一个高阶函数,通常命名为patch
- patch函数的作用是接收新旧VNode,进行对比,最终更新到真实dom上
  patch函数的第一个参数代表旧的VNode,也可以是DOM元素,内部会自动转换为VNode
- h: 函数,用来创建虚拟DOM,接收的第一个参数为标签+选择器,代表需要渲染的dom节点,第二个参数如果传递数组或者字符串,代表的是该节点dom的子节点包括文本节点;
  如果传递的是对象,代表该dom节点的属性,如style样式,on事件函数等

  如果第二个参数传递了对象形式,则第三个参数传入的是该dom节点的子节点

官方内置模块

- attributes: 使用setAttributes设置的属性
- props: 通过对象.属性设置的,不处理布尔值类的属性
- dataset
- class: 切换类名
- style: 行内样式
- eventlisteners: 注册和移除事件

```javascript
import { init } from 'snabbdom/build/package/init'
import { h } from 'snabbdom/build/package/h'

// 导入模块
import { styleModule } from 'snabbdom/build/package/modules/style'
import { eventListenersModule } from 'snabbdom/build/package/modules/eventlisteners'

const patch = init[
  styleModule,
  eventListenersModule
]

let vnode = h('div', [
  h('h1', { style: { backgroundColor: 'pink' } }),
  h('div', { on: { click: clickHandler } }, 'click me')
])

function clickHandler () {}

let app = document.querySelector('#app' )
patch(app, vnode)
```

此时插入这段js的html中的id为app的元素将被替换为新的dom

### 源码阅读

当以学习的角度,而不是作为开发者来克隆代码时,可以加上--depth=1的选项表示只克隆某个分支的最近一次commit,指定-b参数可以克隆指定tag

```bash
git clone https://github.com/snabbdom/snabbdom.git -b v2.1.0 --depth=1
```

先来介绍一下函数重载

在java,c#等语言中,可以在同一范围内声明几个功能类似的同名函数,这些函数的参数个数或参数类型不同

而在javascript中没有重载的概念,typescript中有重载的概念,但是重载的实现是通过代码的逻辑判断调整参数

**调试源码快捷键**

鼠标悬浮至函数,F12(或ctrl/command+鼠标左键)可定位到函数引用或声明处
alt+← 鼠标回到上一次的位置

## diff算法

### key的意义

不设置key,会最大程度重用dom,但有时会有问题,渲染错误;设置了key,进行排序或新增列表时,会移动或新建dom,提高渲染性能
