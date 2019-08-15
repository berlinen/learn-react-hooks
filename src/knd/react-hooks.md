# 来讲讲React Hooks是怎么实现的?

React 16.7-alpha 中新增了功能：Hooks 总结他的功能就是：让<mark>FunctionalComponent</mark>具有<mark>ClassComponent</mark>的功能。

```js
import React, { useState, useEffect } from 'react'

function FuunComp(props) {
  const [ data, setData ] = useState('initialState')

  function handleChange(e) {
      setData(e.target.value)
  }

  useEffect(() => {
      subscribeToSomething()

      return () => {
        unSubscribeToSomething()
      }
  })

  return (
    <input value={data} onChange={handleChange} />
  )
}
```

### 按照Dan的说法，设计Hooks主要是解决ClassComponent的几个问题：

1 很难复用逻辑（只能用HOC，或者render props），会导致组件树层级很深

2 产生巨大的组件（指很多代码必须写在类里面）

3 类组件很难理解，比如方法需要bind，this指向不明确

```js
export default withStyle(style)(connect(/*something*/)(withRouter(MyComponent)))
```

这就是一个4层嵌套的HOC组件

同时，如果你的组件内事件多，那么你的constructor里面可能会酱紫：

```js
class MyComponent extends React.Component {
    constructor () {
         // initiallize

    this.handler1 = this.handler1.bind(this)
    this.handler2 = this.handler2.bind(this)
    this.handler3 = this.handler3.bind(this)
    this.handler4 = this.handler4.bind(this)
    this.handler5 = this.handler5.bind(this)
    // ...more
}
```
虽然最新的class语法可以用handler = () => {}来快捷绑定，但也就解决了一个声明的问题，整体的复杂度还是在的。

然后还有在componentDidMount 和 componentDidUpdate 中内容订阅，还需要在componentWillUnMount 中取消订阅的代码，里面会存在很多重复性工作，最重要的是，在一个classCompoent 中的生命周期方法中的代码，是很难在其他组件中复用的，这就导致了了代码复用率低的问题。

还有就是class代码对于打包工具来说，很难被压缩，比如方法名称。

## 先来了解一些基础概念

首先useState是一个方法，它本身是无法存储状态的

其次，他运行在FunctionalComponent里面，本身也是无法保存状态的

useState只接收一个参数initial value，并看不出有什么特殊的地方。所以React在一次重新渲染的时候如何获取之前更新过的state呢？

在开始讲解源码之前，大家先要建立一些概念：

### React Element

JSX翻译过来之后是React.createElement，他最终返回的是一个ReactElement对象，他的数据解构如下：

```js
const element = {
  $$typeof: REACT_ELEMENT_TYPE, // 是否是普通Element_Type

  // Built-in properties that belong on the element
  type: type,  // 我们的组件，比如`class MyComponent`
  key: key,
  ref: ref,
  props: props,

  // Record the component responsible for creating this element.
  _owner: owner,
};
```

这其中需要注意的是type，在我们写<MyClassComponent {...props} />的时候，他的值就是MyClassComponent这个class，而不是他的实例，实例是在后续渲染的过程中创建的。

### Fiber

每个节点都会有一个对应的Fiber对象，他的数据解构如下：

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;  // 就是ReactElement的`$$typeof`
  this.type = null;         // 就是ReactElement的type
  this.stateNode = null;

  // Fiber
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.firstContextDependency = null;

  // ...others
}
```






