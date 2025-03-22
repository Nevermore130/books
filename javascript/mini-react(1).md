### React.js

如何解决在render函数中递归调用太多render渲染dom，从而导致js线程卡顿？

**requestIdleCallback**

这个函数传入一个deadline参数，表示当前事件循环中执行的task还剩余多少时间

可以基于requestIdleCallback中当前Task的剩余时间来渲染dom

比如

```js
//el 就是外部传入的vdom 即element对象，container就是  #root返回的dom对象
function render(el, container) {
  nextWorkOfUnit = {
    dom: container,
    props: {
      children: [el],
    },
  };
}
let nextWorkOfUnit = null;

function workLoop(deadline) {
  let shouldYield = false;
  while (!shouldYield && nextWorkOfUnit) {
    nextWorkOfUnit = performWorkOfUnit(nextWorkOfUnit);
 
    //只要剩余时间大于1ms  就继续执行performWorkOfUnit返回的下一个work节点
    shouldYield = deadline.timeRemaining() < 1;
  }
 
  requestIdleCallback(workLoop);
}


//首次注册workLoop 在task剩余时间开始执行dom渲染
requestIdleCallback(workLoop);
```

nextWorkOfUnit  就是**封装在requestIdleCallback 中执行的work**

那么如何在performWorkOfUnit返回下一个要执行的work节点呢

```js
//fiber是当前要执行渲染的work节点
function performWorkOfUnit(fiber) {
  //如果filber节点没有dom对象才需要创建dom对象
  if (!fiber.dom) {
    //根据fiber的type （比如type是"div"）来创建dom对象并赋值给fiber
    const dom = (fiber.dom = createDom(fiber.type));
 
    //用fiber父节点的dom对象 添加 创建好的dom 去渲染
    fiber.parent.dom.append(dom);
 
    //更新当前fiber节点dom上的props属性
    updateProps(dom, fiber.props);
  }
 
  ///初始化当前渲染fiber节点的children
  initChildren(fiber)
 
  // 4. 返回下一个要执行的任务
  if (fiber.child) {
    return fiber.child;
  }
 
  if (fiber.sibling) {
    return fiber.sibling;
  }
 
  return fiber.parent?.sibling;
}

 
function createDom(type) {
  return type === "TEXT_ELEMENT"
    ? document.createTextNode("")
    : document.createElement(type);
}

```

初始化当前渲染fiber节点的children

```js
 
function initChildren(fiber) {
  const children = fiber.props.children;
  let prevChild = null;
  /// 遍历当前fiber节点的children
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
      //赋值上一个节点的兄弟节点（即当前遍历的child节点）
      prevChild.sibling = newFiber;
    }
    //记录上一个child vdom节点
    prevChild = newFiber;
  });
}
```

#### 总结

* requestIdleCallback 中执行work任务渲染
* 将vdom 封装成一个fiber对象
* 通过fiber对象的type创建 dom ，将dom添加到fiber父节点的dom对象中完成渲染
* 构建fiber对象的children vdom，将children 封装成fiber并通过链表方式维护串联起来，并找到合适的fiber节点作为下一个要渲染的fiber对象
* 往复上述步骤

### 问题

如果执行渲染fiber节点的时候执行完两个，idle时间不够了，卡了一会有了时间再去渲染剩余的fiber节点，这样可能会造成视觉上dom节点渲染的卡顿



#### 解决

在所有fiber节点遍历完成之后（即fiber链表都遍历完成后），统一commit来完成渲染











