### 如何更新props

从处理fiber节点开始

```js
function performWorkOfUnit(fiber) {
  const isFunctionComponent = typeof fiber.type === "function";
 
  if (isFunctionComponent) {
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }
 
  // 4. 返回下一个要执行的任务
  if (fiber.child) {
    return fiber.child;
  }
 
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) return nextFiber.sibling;
    nextFiber = nextFiber.parent;
  }
}

function updateFunctionComponent(fiber) {
  const children = [fiber.type(fiber.props)];
 
  reconcileChildren(fiber, children);
}
 
function updateHostComponent(fiber) {
  if (!fiber.dom) {
    const dom = (fiber.dom = createDom(fiber.type));
 
    updateProps(dom, fiber.props, {});
  }
 
  const children = fiber.props.children;
  reconcileChildren(fiber, children);
}



```

### reconcileChildren逻辑

如何让新的vdom树的fiber节点快速找到旧vdom树对应的fiber节点呢： 

**Update操作**

* update函数构建根节点wipRoot
* 开始workLoop，通过遍历fiber循环构建vdom树链表，即performWorkOfUnit
* 遍历fiber的children 开始 reconcileChildren
* reconcileChildren内部会将新的vdom树下的fiber节点遍历children, 并对比fiber.alternate.child 和当前fiber的child是否是同一个类型
* 如果是同一个类型则执行updateProps操作，否则则是placement，放置一个新的fiber节点，后续会根据新的vdom树fiber节点的effectTag来决定 在dom上执行什么操作
* 当执行到最后一个nextWorkOfUnit的时候，统一从根节点wipRoot提交dom渲染的commitRoot

```js
function update() {
  wipRoot = {
    dom: currentRoot.dom,
    props: currentRoot.props,
    alternate: currentRoot,
  };
 
  nextWorkOfUnit = wipRoot;
}


function render(el, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [el],
    },
  };
 
  nextWorkOfUnit = wipRoot;
}


// render方法调用时候的根节点
let wipRoot = null;
let currentRoot = null;
let nextWorkOfUnit = null;
function workLoop(deadline) {
  let shouldYield = false;
  while (!shouldYield && nextWorkOfUnit) {
    nextWorkOfUnit = performWorkOfUnit(nextWorkOfUnit);
 
    shouldYield = deadline.timeRemaining() < 1;
  }
 
  if (!nextWorkOfUnit && wipRoot) {
    commitRoot();
  }
 
  requestIdleCallback(workLoop);
}
 

```



```js
 
//fiber: 当前处理的fiber节点
//children: 当前fiber的children(来自于fiber.props.children)
function reconcileChildren(fiber, children) {
  //旧vdom树遍历的时候alternate是没有值的
  //新vdom树是因为在update函数中给wipRoot这个fiber赋值了alternate，即旧vdom树的根节点
  let oldFiber = fiber.alternate?.child; 
  let prevChild = null;
  children.forEach((child, index) => {
    const isSameType = oldFiber && oldFiber.type === child.type;
 
    let newFiber;
    //如果是外部是通过执行update函数 来更新vdom树，就会边建立新vdom树的链表，同时也在建立
    //每一个新的fiber的alternate属性，即oldFiber赋值给新fiber的alternate
    if (isSameType) {
      // update
      newFiber = {
        type: child.type,
        props: child.props,
        child: null,
        parent: fiber,
        sibling: null,
        dom: oldFiber.dom,
        effectTag: "update",
        alternate: oldFiber,
      };
    } else {
      newFiber = {
        type: child.type,
        props: child.props,
        child: null,
        parent: fiber,
        sibling: null,
        dom: null,
        effectTag: "placement",
      };
    }
 
    if (oldFiber) {
      //如果oldFiber有兄弟节点的话，需要把兄弟节点赋值给下一个需要对比的oldFiber
      oldFiber = oldFiber.sibling;
    }
 
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      prevChild.sibling = newFiber;
    }
    prevChild = newFiber;
  });
}
```

#### 在commitRoot中如何渲染dom

```js
function commitRoot() {
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) return;
 
  let fiberParent = fiber.parent;
  while (!fiberParent.dom) {
    fiberParent = fiberParent.parent;
  }
 
  if (fiber.effectTag === "update") {
    updateProps(fiber.dom, fiber.props, fiber.alternate?.props);
  } else if (fiber.effectTag === "placement") {
    if (fiber.dom) {
      fiberParent.dom.append(fiber.dom);
    }
  }
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```

*  如果之前在reconcileChildren处理中新fiber和旧fiber为同一类型，则标记effectTag为update
*  如果不是同一类型，比如新fiber和旧fiber 中的type不一样，或者是render函数进来的，此时还没有旧的vdom树，即肯定没有alternate属性，所以也是标记effectTag为placement

```js
function updateProps(dom, nextProps, prevProps) {
  // 1. old 有  new 没有 删除
  Object.keys(prevProps).forEach((key) => {
    if (key !== "children") {
      if (!(key in nextProps)) {
        dom.removeAttribute(key);
      }
    }
  });
  // 2. new 有 old 没有 添加
  // 3. new 有 old 有 修改
  Object.keys(nextProps).forEach((key) => {
    if (key !== "children") {
      if (nextProps[key] !== prevProps[key]) {
        if (key.startsWith("on")) {
          const eventType = key.slice(2).toLowerCase();
 
          dom.removeEventListener(eventType, prevProps[key]);
 
          dom.addEventListener(eventType, nextProps[key]);
        } else {
          dom[key] = nextProps[key];
        }
      }
    }
  });
}
```











