# 手写 Promise 源码

实现 Promise 源码,首先要知道 Promise 的功能应该是什么样的

- Promise 是一个类,实例化的时候需要传递执行器,执行器会立即执行
- Promise 有三种状态,一旦确定就不可更改
- resolve 和 reject 函数式用来改变状态的
- then 方法被定义在原型对象中,判断状态调用相应函数
- then 成功回调需要传递成功或失败的值

- 当有异步操作时,当前为等待状态,需要存储回调函数后续执行
- then 方法可以多次调用和链式调用

**源码实现**

```javascript
const PENDING = "pending";
const FULFILLED = "fulfilled";
const REJECTED = "rejected";

class Promise {
  // 每个Promise对象都有自己的状态和成功/失败的值,因此需要定义在实例对象上
  status = PENDING;
  value = undefined;
  error = undefined;
  successCallback = undefined;
  failCallback = undefined;
  constructor(executor) {
    executor(this.resolve, this.reject);
  }
  // 不能定义为普通函数,否则resolve/reject自己调用自己,内部this为window或undefined;箭头函数会让此处this指向其实例对象
  resolve = (value) => {
    if (this.status !== PENDING) return;
    this.status = FULFILLED;
    // 保存成功的值
    this.value = value;
  };
  reject = (error) => {
    if (this.status !== PENDING) return;
    this.status = REJECTED;
    this.error = error;
  };
  then(successCallback, failCallback) {
    if (this.status === FULFILLED) {
      successCallback(this.value);
    } else if (this.status === REJECTED) {
      failCallback(this.error);
    }
  }
}
```

当实例化一个 Promise 对象的时候,其中有异步操作如耗时,ajax 等,不能立即 resolve.等待中的状态无法得知调用成功或失败回调

**异步**

需要给 pendding 状态的回调存储下来,并在 resolve 和 reject 下进行调用

```javascript
 resolve = (value) => {
    if (this.status !== PENDING) return;
    this.status = FULFILLED;
    // 保存成功的值
    this.value = value;

    // 判断成功回调是否存在
    this.successCallback && this.successCallback();
  };
  reject = (error) => {
    if (this.status !== PENDING) return;
    this.status = REJECTED;
    this.error = error;

    this.failCallback && this.failCallback();
  };
  then(successCallback, failCallback) {
    if (this.status === FULFILLED) {
      successCallback(this.value);
    } else if (this.status === REJECTED) {
      failCallback(this.error);
    } else {
      // 等待状态,需要存储回调函数后续执行
      this.successCallback = successCallback;
      this.failCallback = failCallback;
    }
  }


// 调用
var p = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(100)
  }, 200)
})

p.then(res => {
  console.log(res)
})
p.then(res => {
  console.log(res)
})
```

**then 方法多次调用**
当同一个 Promise 对象多次调 then 方法时(非链式调用),需要注意异步状态下,需要把传入的多个回调存储起来,按顺序调用

```javascript
successCallback = []
failCallback = []

resolve = (value) => {
  if (this.status !== PENDING) return;
  this.status = FULFILLED;
  // 保存成功的值
  this.value = value;

  // 判断成功回调是否存在
  // this.successCallback && this.successCallback();
  while (this.successCallback.length) {
    this.successCallback.shift()(this.value)
  }
};

reject = (error) => {
  if (this.status !== PENDING) return;
  this.status = REJECTED;
  // 保存成功的值
  this.error = error;

  // 判断成功回调是否存在
  // this.failCallbacj && this.failCallbacj();
  while (this.failCallbacj.length) {
    this.failCallbacj.shift()(this.error)
  }
};

then (successCallback, failCallback) {
  if(this.status === FULFILLED) {
    successCallback(this.value)
  } else if(this.status === REJECTED) {
    failCallback(this.error)
  } else {      // 等待状态,需要存储回调函数后续执行
    this.successCallback.push(successCallback)
    this.failCallback.push(failCallback)
  }
}

```

**链式调用**

```javascript
then (successCallback, failCallback) {
  let promise2 = new Promise((resolve, reject) => {   // 返回一个全新的Promise
    if(this.status === FULFILLED) {
      let x = successCallback(this.value)   // 前面then方法回调返回的值传递给下一个then方法回调中
      resolve(x)
    } else if(this.status === REJECTED) {
      failCallback(this.error)
    } else {      // 等待状态,需要存储回调函数后续执行
      this.successCallback.push(successCallback)
      this.failCallback.push(failCallback)
    }
  })
  return Promise2
}
```

**判断 then 方法回调中返回的是普通值还是 Promise 对象**

```javascript
if (this.status === FULFILLED) {
  let x = successCallback(this.value); // 前面then方法回调返回的值传递给下一个then方法回调中
  parsePromise(x, resolve, reject); // ? 当返回的Promise中有异步操作怎么办?
}

function parsePromise(x, resolve, reject) {
  if (x instanceof Promise) {
    // x.then((value => resolve(value), error => reject(error))
    x.then(resolve, reject); 
  } else {
    resolve(x);
  }
}
```

tips: 不能在 then 方法回调中返回当前 then 方法所返回的 Promise 对象,否则发生循环调用,捕捉此情况并把异常抛出来

如

```javascript
var promise = new Promise((resolve, reject) => {
  resolve(1);
});

var p1 = promise.then((value) => {
  return p1; // 返回自身不被允许,错误会被捕获
});
```

解决方案

```javascript
then (successCallback, failCallback) {
  let promise2 = new Promise((resolve, reject) => {   // 返回一个全新的Promise
    if(this.status === FULFILLED) {
      // 注意,还未实例化成功,此时是获取不到promise2的,包裹将以下代码包裹到setTimeout即可
      let x = successCallback(this.value)   // 前面then方法回调返回的值传递给下一个then方法回调中
      parsePromise(promise2, x, resolve, reject)
    } else if(this.status === REJECTED) {
      failCallback(this.error)
    } else {      // 等待状态,需要存储回调函数后续执行
      this.successCallback.push(successCallback)
      this.failCallback.push(failCallback)
    }
  })
  return Promise2
}

function parsePromise (promise2, x, resolve, reject) {
  if(promise2 === x) {
    return reject(new TypeError('Chaining cycle detedted for promise #<Promise>'))
  }
  if(x instanceof Promise) {
    x.then(resolve, reject)
  } else {
    resolve(x)
  }
}

```

### 错误捕获

**执行器中的错误**

```javascript
class Promise {
  constructor(executor) {
    try {
      executor(this.resolve, this.reject);
    } catch (error) {
      this.reject(error);
    }
  }
}
```

**then 方法回调函数中的错误**

应该在下一个 then 方法的回调函数中捕获错误, 三种状态的错误都要进行处理

```javascript
then (successCallback, failCallback) {
  let promise2 = new Promise((resolve, reject) => {   // 返回一个全新的Promise
    if(this.status === FULFILLED) {
      // 处理then方法中的错误
      setTimeout(() => {
        try {
          let x = successCallback(this.value)   // 前面then方法回调返回的值传递给下一个then方法回调中
          parsePromise(promise2, x, resolve, reject)
        } catch (error) {
          reject(error)   // 传递给下一个Promise对象回调中
        }
      }, 0);
    } else if(this.status === REJECTED) {
        setTimeout(() => {
        try {
          let x = failCallback(this.error)  // 前面then方法回调返回的值传递给下一个then方法回调中
          parsePromise(promise2, x, resolve, reject)
        } catch (error) {
          reject(error)   // 传递给下一个Promise对象回调中
        }
      }, 0);
    } else {      // 等待状态,需要存储回调函数后续执行
      // this.successCallback.push(successCallback)      // 此时无法对错误情况进行处理,,需要改造
      // this.failCallback.push(failCallback)
      this.successCallback.push(() => {
        setTimeout(() => {
          try {
            let x = successCallback(this.value)   // 前面then方法回调返回的值传递给下一个then方法回调中
            parsePromise(promise2, x, resolve, reject)
          } catch (error) {
            reject(error)   // 传递给下一个Promise对象回调中
          }
        }, 0);
      })
      this.failCallback.push(() => {
        setTimeout(() => {
          try {
            let x = failCallback(this.error)  // 前面then方法回调返回的值传递给下一个then方法回调中
            parsePromise(promise2, x, resolve, reject)
          } catch (error) {
            reject(error)   // 传递给下一个Promise对象回调中
          }
        }, 0);
      })
    }
  })
  return Promise2
}
```

### 将then方法参数变成可选参数

promise.then()等价于promise.then(value => value)

```javascript
then (successCallback, failCallback) {
  successCallback ? successCallback : value => value
  failCallback ? failCallback : error => {throw error}
  // ...
}

// 调用
promise.then().then().then(value => console.log(value))
```

### Promise.all

```javascript
function p1 () {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      resolve('p1')
    }, 2000)
  })
}

function p2 () {
  return new Promise(function (resolve, reject) {
    resolve('p2')
  })
}

```
当p1,p2按顺序执行时,先得到p2结果.但是, 如果按顺序使用Promise.all方法中,一定是先得到p1的结果

`Promise.all(['a', 'b', p1(), p2()]).then()`

```javascript
static all (array) {
  let result = []
  let index = 0
  return new Promise((resolve, reject) => {
    function addData (key, value) {
      result[key] = value
      index++
    }
    for (let i = 0;i < array.length;i++) {
      let current = array[i]
      if(current instanceof Promise) {
        current.then((value) => {
          addData(i, value)
        }, (error) => {
          reject(error)
        })
      } else {
        addData(i, current)
      }
    }
    if(index === array.length) {
      resolve(result)
    }
  })
}
```

### Promise.resolve

Promise.resolve()可以传入普通值或者Promise对象,都会创建一个Promise对象;如果传入的是一个Promise对象,那么该方法会原封不动的把这个Promise对象作为Promise.resolve()的返回值

```javascript
static resolve (value) {
  if(value instanceof Promsie) return value
  return new Promise(resolve => resolve(value))
}

//调用
function p1 () {
  return new Promise(function (resolve, reject) {
    setTimeout(() => {
      resolve('p1')
    }, 1000);
  })
}

function p2 () {
  return new Promise(function (resolve, reject) {
    resolve('p2')
  })
}

Promise.resolve(100).then(value => console.log(value))
Promise.resolve(p1()).then(value => console.log(value))
```

### Promise.prototype.finally

- 无论成功还是失败,最终都会执行一次
- 可以链式调用then方法拿到finally最终的结果,因此返回一个Promise对象

```javascript
finally (callback) {
  return this.then((value) => {
    callback()    // bug: 如果finally调用时返回了另外的Promise对象,此处没有等待其状态变更就立即返回了value,错误的
    return value
  }, (error) => {
    callback()
    throw error   // 将错误结果传递到下一个then的失败回调中
  })
}

p2().finally(() => {
  console.log('finally')
  return p1()   // 注意,这个地方后面的then虽然是为了p1()返回的Promise注册的回调,但是then回调中的值不是p1,仍然是p2自身
}).then(value => {
  console.log(value)    // p2
})

```
因此
```javascript
finally (callback) {
  return this.then((value) => {
    // 借用resolve静态方法包装为Promise对象
    return Promise.resolve((callback()).then(newValue => value)
  }, (error) => {
    return Promise.resolve(callback()).then(() => throw error)
  })
}
```

### Promise.prototype.catch

```javascript
catch (failCallback) {
  return this.then(undefined, callback)
}
```

<!--sec data-title="完整代码" data-id="showCode" data-show=true data-collapse=true ces-->
```javascript
const PENDING = 'pending'; // 等待
const FULFILLED = 'fulfilled'; // 成功
const REJECTED = 'rejected'; // 失败

class MyPromise {
  constructor (executor) {
    try {
      executor(this.resolve, this.reject)
    } catch (e) {
      this.reject(e);
    }
  }
  // promsie 状态 
  status = PENDING;
  // 成功之后的值
  value = undefined;
  // 失败后的原因
  reason = undefined;
  // 成功回调
  successCallback = [];
  // 失败回调
  failCallback = [];

  resolve = value => {
    // 如果状态不是等待 阻止程序向下执行
    if (this.status !== PENDING) return;
    // 将状态更改为成功
    this.status = FULFILLED;
    // 保存成功之后的值
    this.value = value;
    // 判断成功回调是否存在 如果存在 调用
    // this.successCallback && this.successCallback(this.value);
    while(this.successCallback.length) this.successCallback.shift()()
  }
  reject = reason => {
    // 如果状态不是等待 阻止程序向下执行
    if (this.status !== PENDING) return;
    // 将状态更改为失败
    this.status = REJECTED;
    // 保存失败后的原因
    this.reason = reason;
    // 判断失败回调是否存在 如果存在 调用
    // this.failCallback && this.failCallback(this.reason);
    while(this.failCallback.length) this.failCallback.shift()()
  }
  then (successCallback, failCallback) {
    // 参数可选
    successCallback = successCallback ? successCallback : value => value;
    // 参数可选
    failCallback = failCallback ? failCallback: reason => { throw reason };
    let promsie2 = new MyPromise((resolve, reject) => {
      // 判断状态
      if (this.status === FULFILLED) {
        setTimeout(() => {
          try {
            let x = successCallback(this.value);
            // 判断 x 的值是普通值还是promise对象
            // 如果是普通值 直接调用resolve 
            // 如果是promise对象 查看promsie对象返回的结果 
            // 再根据promise对象返回的结果 决定调用resolve 还是调用reject
            resolvePromise(promsie2, x, resolve, reject)
          }catch (e) {
            reject(e);
          }
        }, 0)
      }else if (this.status === REJECTED) {
        setTimeout(() => {
          try {
            let x = failCallback(this.reason);
            // 判断 x 的值是普通值还是promise对象
            // 如果是普通值 直接调用resolve 
            // 如果是promise对象 查看promsie对象返回的结果 
            // 再根据promise对象返回的结果 决定调用resolve 还是调用reject
            resolvePromise(promsie2, x, resolve, reject)
          }catch (e) {
            reject(e);
          }
        }, 0)
      } else {
        // 等待
        // 将成功回调和失败回调存储起来
        this.successCallback.push(() => {
          setTimeout(() => {
            try {
              let x = successCallback(this.value);
              // 判断 x 的值是普通值还是promise对象
              // 如果是普通值 直接调用resolve 
              // 如果是promise对象 查看promsie对象返回的结果 
              // 再根据promise对象返回的结果 决定调用resolve 还是调用reject
              resolvePromise(promsie2, x, resolve, reject)
            }catch (e) {
              reject(e);
            }
          }, 0)
        });
        this.failCallback.push(() => {
          setTimeout(() => {
            try {
              let x = failCallback(this.reason);
              // 判断 x 的值是普通值还是promise对象
              // 如果是普通值 直接调用resolve 
              // 如果是promise对象 查看promsie对象返回的结果 
              // 再根据promise对象返回的结果 决定调用resolve 还是调用reject
              resolvePromise(promsie2, x, resolve, reject)
            }catch (e) {
              reject(e);
            }
          }, 0)
        });
      }
    });
    return promsie2;
  }
  finally (callback) {
    return this.then(value => {
      return MyPromise.resolve(callback()).then(() => value);
    }, reason => {
      return MyPromise.resolve(callback()).then(() => { throw reason })
    })
  }
  catch (failCallback) {
    return this.then(undefined, failCallback)
  }
  static all (array) {
    let result = [];
    let index = 0;
    return new MyPromise((resolve, reject) => {
      function addData (key, value) {
        result[key] = value;
        index++;
        if (index === array.length) {
          resolve(result);
        }
      }
      for (let i = 0; i < array.length; i++) {
        let current = array[i];
        if (current instanceof MyPromise) {
          // promise 对象
          current.then(value => addData(i, value), reason => reject(reason))
        }else {
          // 普通值
          addData(i, array[i]);
        }
      }
    })
  }
  static resolve (value) {
    if (value instanceof MyPromise) return value;
    return new MyPromise(resolve => resolve(value));
  }
}

function resolvePromise (promsie2, x, resolve, reject) {
  if (promsie2 === x) {
    return reject(new TypeError('Chaining cycle detected for promise #<Promise>'))
  }
  if (x instanceof MyPromise) {
    // promise 对象
    // x.then(value => resolve(value), reason => reject(reason));
    x.then(resolve, reject);
  } else {
    // 普通值
    resolve(x);
  }
}

module.exports = MyPromise;
```
<!--endsec-->

