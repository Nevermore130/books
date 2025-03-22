### useState实现

如何使用

```js
 
function Foo() {
  console.log("re foo")
  const [count, setCount] = React.useState(10);
  const [bar, setBar] = React.useState("bar");
  function handleClick() {
    setCount((c) => c + 1);
    setBar((s) => s + "bar");
  }
 
  return (
    <div>
      <h1>foo</h1>
      {count}
      <div>{bar}</div>
      <button onClick={handleClick}>click</button>
    </div>
  );
}
 
function App() {
  return (
    <div>
      hi-mini-react
      <Foo></Foo>
    </div>
  );
}
 
export default App;
```

* button 点击后会执行handleClick 
* handleClick 会用 useState返回的setState函数来设置新的值并触发dom刷新

先实现useState函数

```js
let stateHooks;
let stateHookIndex;

function useState(initial) {
  let currentFiber = wipFiber; //暂存当前fiber
  //尝试获取旧vdom节点的stateHooks对象是否能拿到对应的hook对象
  //旧vdom 节点即currentFiber.alternate
  const oldHook = currentFiber.alternate?.stateHooks[stateHookIndex];
  
  //如果旧节点没有stateHooks对象(第一次执行)，则当前stateHook用initial的值
  const stateHook = {
    state: oldHook ? oldHook.state : initial,
    queue: oldHook ? oldHook.queue : [],
  };
 
  stateHook.queue.forEach((action) => {
    stateHook.state = action(stateHook.state);
  });
 
  stateHook.queue = [];
 
  //更新当前stateHook索引
  stateHookIndex++;
  stateHooks.push(stateHook);
 
  //stateHooks存到当前fiber节点上
  currentFiber.stateHooks = stateHooks;
 
  //外层执行setState((state)=>state + 1)
  //之前旧的stateHook放入 当前action
  //给wipRoot 赋值，同时赋值给nextWorkOfUnit，然后开始update vdom树。开始刷新
  function setState(action) {
    stateHook.queue.push(action);
 
    wipRoot = {
      ...currentFiber,
      alternate: currentFiber,
    };
 
    nextWorkOfUnit = wipRoot;
  }
 
  return [stateHook.state, setState];
}
```

* update vdom树后，由于会重新执行updateHostComponent ,所以会重新执行Foo() 
* 重新执行Foo()后又会执行useState()，这个时候就能拿到oldHook了，而oldHook存有之前setState赋值的新值
* 这个时候useState()返回的就是更新后的值了