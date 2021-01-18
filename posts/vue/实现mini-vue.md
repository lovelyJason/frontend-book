# 实现mini-vue

## vue-router

在Hash模式中调用router.push(url)时,push内部会判断当前浏览器是否支持window.history.pushState改变地址栏,否则会通过window.location改变地址栏

## compiler

```javascript
class Comoiler {
  constructor (vm) {
    this.el = vm.$el
    this.vm = vm
    this.compile(this.el)
  }
  compile (el) {

  }
  compileText (node) {
    let reg = /\{\{(.+?)\}\}/
    let value = node.textContent
    if(reg.test(value)) {
      let key = RegExp.$1.trim()
      node.textContent = value.replace(reg,this.vm[key])
    }
  }
  compileElement () {

  }
  isDirective (attrName) {
    return attrName.startWith('v-')
  }
  isTextNode (node) {
    return node.nodeType === 3
  }
  update (node, key, attrName) {
    let updateFn = this[attrName + 'Updater']
    updateFn && updateFn.call(this, node, this.vm[key], key)    // 注意this指向
  }
  textUpdater (node, value) {
    node.textContent = value
  }
  modelUpdater (node, value) {
    node.value = value
  }
}
```

## Observer

```javascript
class Observer {
  constructor (data) {
    this.walk(data)
  }
  walk (data) {
    if(!data || typeof data !== 'object') {
      return
    }
    Object.keys(data).forEach(key => {
      this.defineReactive(data, key, data[key])
    })
  }
  defineReactive (obj, key, value) {
    let that = this
    let dep = new Dep()
    this.walk()
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get () {
        // 将来在Watcher中,会访问到旧值,进而将某个数据依赖的Watcher实例添加入发布者Dep中
        Dep.target && dep.addSub(Dep.target)
        return value  // 闭包
      },
      set (newValue) {
        if(newValue === obj[key]) {
          return
        }
        value = newValue
        // 对data中的数据进行赋值时,有可能将普通的原始类型数据赋值为对象类型,需要继续给对象中的属性进行响应式处理
        that.walk(newValue)
        // data中的数据发生变化时,调用发布者的通知方法,通知所有Watcher更新视图
        dep.notify()
      }
    })
  }
}


```

## Watcher

![uml.drawio](/Users/jasonhuang/projects/frontend-book/posts/engineering/uml.drawio.png)

```javascript
class Watcher {
  construcor (vm, key, cb) {
    this.vm = vm
    this.key = key
    // 回调函数负责更新视图
    this.cb = cb

    // 记录当前watcher对象到Dep类的静态属性target中
    Dep.target = this
    // 触发get方法,get方法中会调用addSub
    this.oldValue = vm[key]
    Dep.target = null
  }
  // 数据发生变化更新视图
  uppdate () {
    // 此时值已经发生变化了
    let newValue = this.vm[this.key]
    if(this.oldValue === newValue) {
      return
    }
    this.cb(newValue)
  }
}
```

vue中指令和插值表达式依赖于响应式数据,所有视图中依赖响应式数据的地方,都会有一个watcher对象,当数据改变时,Dep对象会通知所有Watcher对象更新视图.因此在Compiler中更新dom时应该实例化Watcher并传入更改dom的回调

Compiler
```javascript
compileText (node) {
  let reg = /\{\{(.+?)\}\}/
  let value = node.textContent
  if(reg.test(value)) {
    let key = RegExp.$1.trim()
    node.textContent = value.replace(reg,this.vm[key])

    // 创建Watcher对象
    new Watcher(this.vm, key, newValue => {
      node.textContent = newValue
    })
  }
}

textUpdater (node, value, key) {
  node.textContent = value
  new Watcher(this.vm, key, newValue => {
    node.textContent = value
  })
}

modelUpdater (node, value, key) {
  node.value = value
  new Value(this.vm, key, newValue => {
    node.value = valeu
  })
}

```

此时数据发生变化了,视图也会发生变化,但是双向绑定的功能还未完善
