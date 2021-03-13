# Vuex数据流管理

我们知道,Vue中的两个核心功能为数据驱动和组件化,数据可以称为状态,每个组件可以有自己的状态,同时可以有全局的共享状态,以及局部的共享状态,而全局的共享状态,在Vue中的体现即是Vuex

状态管理实际上就是通过状态集中管理和分发,解决多个组件共享状态的问题

其中设计到的概念有

- `state`,驱动应用的数据源
- `view`,以声明方式将state映射到视图
- `actions`,用户交互导致的状态变更

这里先复习一下Vue中组件间通信的各种方式

## vue组件通信方式

- 父组件传值给子组件

  父组件通过动态属性给子组件传递数据或者表达式,子组件在选项props中接收

- 子组件传值给父组件

  使用自定义事件

- 不相关组件传值

  使用自定义事件,不同于子传父,使用eventbus作为事件总线,分发事件

- 通过ref获取子组件实例

------

## Vuex使用

Vuex的使用就不过多介绍了,官网也挺详细的,主要就是`store`中`state`负责存储状态,`mutations`负责同步更新状态,`actions`负责异步更新状态(内部仍然是通过commit提交mutation),`modules`可以指定子模块防止仓库过大难以拆分;在模板中,通过`$store`的`state`可以获取状态,`commit`提交`mutation`,`dispatch`派发`action`

另外为了简化以上操作,可以使用`mapState`,`mapGetters`,`mapMutions`,`mapActions`将对应的值或函数映射到`computed`,`methods`中

需要说明一下的是,其实视图中是可以直接更改`$store.state`中的数据,无疑破坏了使用Vuex的初衷,我们期望所有状态的变更是通过提交`mutations`和派发`actions`的,可以在`Vuex.Store()`初始化的时候设置`strict`为`true`,即可开启校验,如果直接更新state,会抛出错误;注意不要在生产环境中开启严格模式,这个选项会消耗较多性能

这里之所以更新状态分为同步和异步,是为了让我们的程序可以方便调试,能够更好的使用Vue devtools的时光旅行功能进行调试跟踪状态

## 应用场景 && 使用中的问题

购物车模块是最常见的使用场景,一般是,登陆状态下,从接口获取数据,没有登陆的状态下,也需要保存用户购买的数据,此时商品的增减和价格的统计,会在多个页面频繁出现以及计算,数据传递比较麻烦,此时使用Vuex比较方便.需要注意,Vuex的数据是存储在内存当中,刷新页面会导致数据丢失,解决方式也很简单,在Vuex中state初始值可以先从localstorage中获取

假设我们Vuex中,购物车的数据读写放在子模块'cart'中,cartProducts代表着购物车中的商品列表

```javascript
const state = {
  cartProducts: JSON.parse(window.localstorage.getItem('cardProducts')) || []
}
```

同理,state状态的改变也应该把数据更新到localstorage,我们需要在所有mutations中执行一遍`localstorage.set()`,比较麻烦,所幸Vuex插件提供了更为便捷的方案

> Vuex插件就是一个函数,该函数接收一个store的参数

1. 创建插件

```javascript
const p = store => {
  store.subscribe((mutation, state) => {
    // 这里cart模块中所有mutaion的操作都是改变购物车中的相应数据,如商品个数,价格等
    if(mutaion.type.startsWith('cart')) {
      window.localstorage.setItem('cartProducts', JSON.stringify(state.cart.cartProducts))
    }
  })
}
```

`subscribe`作用是订阅mutaion,mutation执行完毕后,会执行插件中的逻辑,插件的初始化需要在store的初始化之前

mutation参数有两个属性,分别为payload,type,其中type的格式为'命名空间/mutaion名',如`cart/addToCart`,我们可以借助这个订阅指定的mutation

2. 在store中创建插件

```javascript
new Vuex.store({
  // state,getters,mutations,actions,modules略
  plugins: [p]
})
```

合理的建议是, 不是所有项目都适合使用Vuex,几个页面之间的简单传值就可以达到目的,使用Vuex会适得其反,只有中大型项目,管理的状态比较多的时候,重复的ui也比较多,这时候使用Vuex就能集中处理数据的分发,避免数据传递,路由跳转带来的复杂性

### 手写一个简易的Vuex

学会了Vuex的使用,接下来咱手动实现一下Vuex加深下理解

我们知道,Vue的插件,本质上是一个函数或者对象,如果是对象,要对外暴露一个install方法,才能被Vue.use()所注册,这一点可以在Vue的源码中看到

```typescript
export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```

写之前,先分析下Vue插件应该有哪些功能

- 要对外暴露`install`方法和`Store`属性,供外部初始化
- 模板中可以通过`$store`放到到`store`中传入的选项,因此`install`方法中要将选项注入Vue实例的`$store`中
  但是这里我们无法再`install`中直接获取Vue实例,可以借助`Vue.mixin()`混入`beforeCreate`在所有Vue实例注入逻辑,VueRouter中也是这么处理的
- Store类的实现: 接收出`Vuex.Store()`传入的对象,对象中有`state`,`getters`,`mutations`等,且state是响应式数据,还应该具有两个方法,`dispatch`负责派发`action`,`commit`负责提交`mutation`

```javascript
let _Vue = null     // 缓存全局Vue变量

class Store {
  constructor (options) {
    const {
      state = {},
      getters = {},
      mutations = {},
      actions = {}
    } = options
    this.state = _Vue.observable(state)
    // getters只有在访问是会去计算值,实际上就是将getters中每个值的getter访问器返回给外部
    this.getters = Object.create(null)
    Object.keys(getters).forEach(key => {
      Object.defineProperty(this.getters, key, {
        get: () => {
          return getters[key](state)    // 使用getters的时候,函数的第一个参数就是state
        }
      })
    })
    // Vuex Store中可以通过$store._mutations访问到所有的mutaion,且应该是一个不对外暴露的数据
    this._mutations = mutations
    this._actions = actions
  }

  comit (type, payload) {
    this._mutations[type](this.state, payload)
  }
  
  dispatch (type, payload) {
    const context = this
    this._acitions[type](context, payload)
  }
}
function install (Vue) {
  _Vue = Vue
  _Vue.mixin({
    beforeCreate() {
      // 在Vue的根实例的原型中注入$store,从而所有组件中都能够使用
      if (this.$options.store) {
        _Vue.prototype.$store = this.$options.store
      }
    }
  })
}
export default {
  Store,
  install
}
```

