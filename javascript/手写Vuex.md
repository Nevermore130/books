#### 如何使用

```js
// import { createStore } from "vuex";
import { createStore } from "../mini-vuex";

// 创建Store实例
const store = createStore({
  strict: true,
  state() {
    return {
      count: 1,
    };
  },
  mutations: {
    add(state) {
      state.count++;
    },
  },
  getters: {
    doubleCounter(state) {
      return state.count * 2;
    },
  },
  actions: {
    add({ commit }) {
      setTimeout(() => {
        commit("add");
      }, 1000);
    },
  },
});

export { store };
```

* state() 返回一个state对象，里面包含相关属性，如count
* mutations 类似reducer ，比如add函数就是找到key为add并处理，接受传入的state参数
* 在调用store.getter.xxxx 就是调用用户定义的getter里的属性
* actions ： dispatch(action)  就是通过传入的action找到actions定义的函数，并传入commit等参数，如果此action执行commit操作，则会在mutations里找到对应action的处理函数

```js
import { computed } from "@vue/reactivity";
import { reactive, watch } from "vue";

export function createStore(options) { 
	// Store实例
  const store = {
    _state: reactive(options.state()),
    get state() {
      return this._state;
    },
    set state(v) {
      console.error("please use replaceState() to reset state");
    },
    _mutations: options.mutations,
    _actions: options.actions,
    _commit: false, // 提交标识符，如果通过commit方式修改状态，则设置为true
  	_withCommit(fn) {
      // fn就是用户设置的mutaion执行函数
      this._commit = true;
      fn();
      this._commit = false;
    },
  };
  
  
  function commit(type, payload) {
    // 获取type对应的mutation,比如传入的是commit("add"),找到对应的add函数
    const entry = this._mutations[type];
    if (!entry) {
      console.error(`unknown mutation type: ${type}`);
      return;
    }
    // 要使用withCommit方式提交
    this._withCommit(() => {
      //执行add函数并传入state对象，第一个参数是绑定this对象，比如add函数中调用this就是指state
      entry.call(this.state, this.state, payload);
    });
  }
  
  function dispatch(type, payload) {
    // 获取用户编写的type对应的action
    const entry = this._actions[type];
    if (!entry) {
      console.error(`unknown action type: ${type}`);
      return;
    }
    /// 绑定this对象为store，比如actions里的定义的add中调用this就是store对象
    /// 同时传入store对象作为参数
    return entry.call(this, this, payload);
  }
  
  //确保commit 和dispatch中的this对象都为store
  store.commit = commit.bind(store);
  store.dispatch = dispatch.bind(store);
  
  // 定义store.getters
  store.getters = {};
  
  // 遍历用户定义getters
  Object.keys(options.getters).forEach((key) => {
    // 定义计算属性
    const result = computed(() => {
      const getter = options.getters[key];
      if (getter) {
        return getter.call(store, store.state);
      } else {
        console.error("unknown getter type:" + key);
        return "";
      }
    });

    // 当外部调用 store.getters.xxx 时
    // 会获取之前在这里动态定义在store.getter里的key
    //key 对应的value返回一个计算属性对象
    // 而computed对象getValue会 判断是否是dirty，如果是dirty就执行computed传入的回调函数fn
    //当响应式对象state改变了，会触发compute对象里定义的effect回调，此时会把dirty置为true
    //如果响应式对象state没有改变，则直接返回computed里的value即可，算是提高了性能
    Object.defineProperty(store.getters, key, {
      // 只读
      get() {
        return result;
      },
    });
  });
  
  
  // strict模式
  if (options.strict) {
    // 监听store.state变化
    watch(
      store.state,
      () => {
        if (!store._commit) {
          console.warn("please use commit to mutate state!");
        }
      },
      {
        deep: true,
        flush: "sync",
      }
    );
  }
  
   // 插件实现要求的install方法
  store.install = function (app) {
    const store = this;
    // 注册$router
    app.config.globalProperties.$store = store;
  };

  return store;
  
}
```







