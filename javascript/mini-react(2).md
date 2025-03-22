#### functionComponent

vite在把jsx解析成类似`` React.createElement("div",{id: "123"}, "hi-react","hello world")``

来创建vdom

那如果jsx返回的是functionComponent呢？ 

```jsx
import React from "./core/React.js";
 
function Counter({ num }) {
  return <div>count: {num}</div>;
}

function App() {
  return (
    <div>
      hi-mini-react
      <Counter num={10}></Counter>
      <Counter num={20}></Counter>
    </div>
  );
}
export default App;
```

`` React.createElement(function App() ....., {}, React.createElement("div",NULL,"hi-mini-react"))``

``React.createElement(type,props,...children)``  这个函数就是创建vdom的函数

既然jsx中的functionComponent 会在createElement的时候把function对象传给type参数。

那么可以在这里判断是否是functionComponent

```js
//既然jsx中的functionComponent 会在createElement的时候把function对象传给type参数
function performWorkOfUnit(fiber) {
  //所以如果fiber.type 是function类型的话就是FunctionComponent组件了
  const isFunctionComponent = typeof fiber.type === "function";
 
  if(isFunctionComponent){
    updateFunctionComponent(fiber)
  }else{
    updateHostComponent(fiber)
  }
 
  // 4. 返回下一个要执行的任务
  if (fiber.child) {
    return fiber.child;
  }
 
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) return nextFiber.sibling;
    //向上查找，一直找到有兄弟节点的fiber作为下一个执行的fiber，否则万一是fiber是functionComponet则没有dom对象！
    //这样在fiber.parent.dom.append(dom)操作的时候就无法渲染dom元素
    nextFiber = nextFiber.parent;
  }
}


 
function updateFunctionComponent(fiber) {
  //执行fiber中的传入的functionComponent对象，即执行function App(){return <div></div>} 这个function
  //同时传入fiber的props
  const children = [fiber.type(fiber.props)];
 
  initChildren(fiber, children);
}

//遍历fiber的children
function initChildren(fiber, children) {
  let prevChild = null;
  children.forEach((child, index) => {
    const newFiber = {
      type: child.type,
      props: child.props,
      child: null,
      parent: fiber,
      sibling: null,
      dom: null,
    };
 
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      prevChild.sibling = newFiber;
    }
    prevChild = newFiber;
  });
}

 //如果fiber不是functionComponent
//执行创建dom的操作
//更新Props,遍历children等
function updateHostComponent(fiber) {
  if (!fiber.dom) {
    const dom = (fiber.dom = createDom(fiber.type));
 
    updateProps(dom, fiber.props);
  }
 
  const children = fiber.props.children;
  initChildren(fiber, children);
}
```











