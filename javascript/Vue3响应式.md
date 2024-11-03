### 响应式 ###

响应式机制的主要功能就是，可以把普通的 **JavaScript 对象封装成为响应式对象，拦截数据的获取和修改操作，实现依赖数据的自动化更新。**

一个最简单的响应式模型，我们可以通过 **reactive 或者 ref 函数，把数据包裹成响应式对象**，**并且通过 effect 函数注册回调函数，然后在数据修改之后，响应式地通知 effect 去执行回调函数即可。**

使用下面的代码在 node 环境中使用 Vue 响应。以 reactive 为例，我们使用 reactive 包裹 JavaScript 对象之后**，每一次对响应式对象 counter 的修改，都会执行 effect 内部注册的函数：**

```js
const {effect, reactive} = require('@vue/reactivity')

let dummy
const counter = reactive({ num1: 1, num2: 2 })
effect(() => {
  dummy = counter.num1 + counter.num2
  console.log(dummy)// 每次counter.num1修改都会打印日志
})
setInterval(()=>{
  counter.num1++
},1000)
```

执行 node test.js 之后，你就可以看到 effect 内部的函数会一直调用，每次 count.value 修改之后都会执行。

**在 effect 中获取 counter.num1 和 counter.num2 的时候，就会触发 counter 的 get 拦截函数**；get 函数，**会把当前的 effect 函数注册到一个全局的依赖地图中去**。这样 counter.num1 在修改的时候，**就会触发 set 拦截函数，去依赖地图中找到注册的 effect 函数，然后执行。**

### reactive ###

进入到 src/reactivity 目录中，新建 reactive.spec.js，使用下面代码测试 reactive 的功能，能够在响应式数据 ret 更新之后，执行 effect 中注册的函数：

```js
import { effect } from '../effect'
import { reactive } from '../reactive'

describe('测试响应式', () => {
  test('reactive基本使用', () => {
    const ret = reactive({ num: 0 })
    let val
    effect(() => {
      val = ret.num
    })
    expect(val).toBe(0)
    ret.num++
    expect(val).toBe(1)
    ret.num = 10
    expect(val).toBe(10)
  })
})
```

在 Vue3 中，**reactive 是通过 ES6 中的 Proxy 特性实现的属性拦截**，所以，在 reactive 函数中我们直接返回 newProxy 即可：

```js
export function reactive(target) {
  if (typeof target!=='object') {
    console.warn(`reactive  ${target} 必须是一个对象`);
    return target
  }

  return new Proxy(target, mutableHandlers);
}
```

下一步我们需要实现的就是 Proxy 中的处理方法 mutableHandles。

### mutableHandles ###

它要做的事就是**配置 Proxy 的拦截函数，这里我们只拦截 get 和 set 操作**，进入到 baseHandlers.js 文件中

使用 createGetter 和 createSetters 来创建 set 和 get 函数，**mutableHandles 就是配置了 set 和 get 的对象返回。**

* get 中直接返回读取的数据，这里的 Reflect.get 和 target[key]实现的结果是一致的；并且返回值是对象的话，还会嵌套执行 reactive，并且调用 track 函数收集依赖
* set 中调用 trigger 函数，执行 track 收集的依赖。

```js
const get = createGetter();
const set = createSetter();

function createGetter(shallow = false) {
  return function get(target, key, receiver) {
    const res = Reflect.get(target, key, receiver)
    track(target, "get", key)
    if (isObject(res)) {
      // 值也是对象的话，需要嵌套调用reactive
      // res就是target[key]
      // 浅层代理，不需要嵌套
      return shallow ? res : reactive(res)
    }
    return res
  }
}

function createSetter() {
  return function set(target, key, value, receiver) {
    const result = Reflect.set(target, key, value, receiver)
    // 在触发 set 的时候进行触发依赖
    trigger(target, "set", key)
    return result
  }
}
export const mutableHandles = {
  get,
  set,
};
```

track 函数是怎么完成依赖收集的。

### track ###

在 track 函数中，我们可以使用一个巨大的 **tragetMap** 去存储依赖关系

**map 的 key 是我们要代理的 target 对象**，值还是一个 **depsMap，存储这每一个 key 依赖的函数**，每一个 key 都可以依赖多个 effect。上面的代码执行完成，depsMap 中就有了 num1 和 num2 两个依赖。

而依赖地图的格式，用代码描述如下:

```js
targetMap = {
 target： {
   key1: [回调函数1，回调函数2],
   key2: [回调函数3，回调函数4],
 }  ,
  target1： {
   key3: [回调函数5]
 }  

}
```

在 reactive 下新建 effect.js

由于 target 是对象，所以必须得用 map 才可以把 target 作为 key 来管理数据，每次操作之前需要做非空的判断。最终把 activeEffect 存储在集合之中

```js
const targetMap = new WeakMap()

export function track(target, type, key) {

  // console.log(`触发 track -> target: ${target} type:${type} key:${key}`)

  // 1. 先基于 target 找到对应的 dep
  // 如果是第一次的话，那么就需要初始化
  // {
  //   target1: {//depsmap
  //     key:[effect1,effect2]
  //   }
  // }
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    // 初始化 depsMap 的逻辑
    // depsMap = new Map()
    // targetMap.set(target, depsMap)
    // 上面两行可以简写成下面的
    targetMap.set(target, (depsMap = new Map()))
  }
  let deps = depsMap.get(key)
  if (!deps) {
    deps = new Set()
  }
  if (!deps.has(activeEffect) && activeEffect) {
    // 防止重复注册
    deps.add(activeEffect)
  }
  depsMap.set(key, deps)
}
```

###  trigger ###

**trigger 函数实现的思路就是从 targetMap 中，根据 target 和 key 找到对应的依赖函数集合 deps，然后遍历 deps 执行依赖函数。**

```js
export function trigger(target, type, key) {
  // console.log(`触发 trigger -> target:  type:${type} key:${key}`)
  // 从targetMap中找到触发的函数，执行他
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    // 没找到依赖
    return
  }
  const deps = depsMap.get(key)
  if (!deps) {
    return
  }
  deps.forEach((effectFn) => {

    if (effectFn.scheduler) {
      effectFn.scheduler()
    } else {
      effectFn()
    }
  })
  
}
```

执行的是 effect 的 scheduler 或者 run 函数，这是因为我们**需要在 effect 函数中把依赖函数进行包装**，并对依赖函数的执行时机进行控制

### effect ###

```js
export function effect(fn, options = {}) {
  // effect嵌套，通过队列管理
  const effectFn = () => {
    try {
      activeEffect = effectFn
      //fn执行的时候，内部读取响应式数据的时候，就能在get配置里读取到activeEffect
      return fn()
    } finally {
      activeEffect = null
    }
  }
  if (!options.lazy) {
    //没有配置lazy 直接执行
    effectFn()
  }
  effectFn.scheduler = options.scheduler // 调度时机 watchEffect会用到
  return effectFn
  
}
```

* 我们把传递进来的 fn 函数通过 effectFn 函数包裹执行
* 在 effectFn 函数内部，把函数赋值给全局变量 activeEffect
* 然后执行 fn() 的时候，就会触发响应式对象的 get 函数
* get 函数内部就会把 activeEffect 存储到依赖地图中，完成依赖的收集

effect 传递的函数，比如可以通过传递 lazy 和 scheduler 来控制函数执行的时机，默认是同步执行。

scheduler 存在的意义就是我们可以手动控制函数执行的时机，方便应对一些性能优化的场景，比如数据在一次交互中可能会被修改很多次，我们不想每次修改都重新执行依次 effect 函数，而是合并最终的状态之后，最后统一修改一次。

scheduler 怎么用你可以看下面的代码，我们**使用数组管理传递的执行任务，最后使用 Promise.resolve 只执行最后一次，这也是 Vue 中 watchEffect 函数的大致原理。**

```js
const obj = reactive({ count: 1 })
effect(() => {
  console.log(obj.count)
}, {
  // 指定调度器为 queueJob
  scheduler: queueJob
})
// 调度器实现
const queue: Function[] = []
let isFlushing = false
function queueJob(job: () => void) {
  if (!isFlushing) {
    isFlushing = true
    Promise.resolve().then(() => {
      let fn
      while(fn = queue.shift()) {
        fn()
      }
    })
  }
}
```

### 另一个选择 ref 函数 ###

ref 的执行逻辑要比 reactive 要简单一些，不需要使用 Proxy 代理语法，直接使用对象语法的 getter 和 setter 配置，监听 value 属性即可。

在 ref 函数返回的对象中，对象的 get value 方法，使用 track 函数去收集依赖，set value 方法中使用 trigger 函数去触发函数的执行。

```js
export function ref(val) {
  if (isRef(val)) {
    return val
  }
  return new RefImpl(val)
}
export function isRef(val) {
  return !!(val && val.__isRef)
}

// ref就是利用面向对象的getter和setters进行track和trigget
class RefImpl {
  constructor(val) {
    this.__isRef = true
    this._val = convert(val)
  }
  get value() {
    track(this, 'value')
    return this._val
  }

  set value(val) {
    if (val !== this._val) {
      this._val = convert(val)
      trigger(this, 'value')
    }
  }
}

// ref也可以支持复杂数据结构
function convert(val) {
  return isObject(val) ? reactive(val) : val
}
```

ref 函数实现的相对简单很多，只是利用面向对象的 getter 和 setter 拦截了 value 属性的读写，这也是为什么我们需要操作 ref 对象的 value 属性的原因。

### computed ###













