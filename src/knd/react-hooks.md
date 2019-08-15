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

在这里我们需要注意的是this.memoizedState，这个key就是用来存储在上次渲染过程中最终获得的节点的state的，每次执行render方法之前，React会计算出当前组件最新的state然后赋值给class的实例，再调用render。

所以很多不是很清楚React原理的同学会对React的ClassComponent有误解，认为state和lifeCycle都是自己主动调用的，因为我们继承了React.Component，它里面肯定有很多相关逻辑。事实上如果有兴趣可以去看一下Component的源码，大概也就是100多行，非常简单。所以在React中，class仅仅是一个载体，让我们写组件的时候更容易理解一点，毕竟组件和class都是封闭性较强的

## 原理

在知道上面的基础之后，对于Hooks为什么能够保存无状态组件的原理就比较好理解了。

我们假设有这么一段代码：

```js
function FunctionalComponent () {
  const [state1, setState1] = useState(1)
  const [state2, setState2] = useState(2)
  const [state3, setState3] = useState(3)
}
```

在我们执行functionalComponent的时候，在第一次执行到useState的时候，他会对应Fiber对象上的memoizedState，这个属性原来设计来是用来存储ClassComponent的state的，因为在ClassComponent中state是一整个对象，所以可以和memoizedState一一对应。

但是在Hooks中，React并不知道我们调用了几次useState，所以在保存state这件事情上，React想出了一个比较有意思的方案，那就是调用useState后设置在memoizedState上的对象长这样：

```javaScript
{
  baseState,
  next,
  baseUpdate,
  queue,
  memoizedState
}
```

我们叫他Hook对象。这里面我们最需要关心的是memoizedState和next，memoizedState是用来记录这个useState应该返回的结果的，而next指向的是下一次useState对应的`Hook对象。

也就是说：

```js
hook1 => Fiber.memoizedState
state1 === hook1.memoizedState
hook1.next => hook2
state2 === hook2.memoizedState
hook2.next => hook3
state3 === hook3.memoizedState
```

每个在FunctionalComponent中调用的useState都会有一个对应的Hook对象，他们按照执行的顺序以类似链表的数据格式存放在Fiber.memoizedState上

重点来了：就是因为是以这种方式进行state的存储，所以useState（包括其他的Hooks）都必须在FunctionalComponent的根作用域中声明，也就是不能在if或者循环中声明，比如







