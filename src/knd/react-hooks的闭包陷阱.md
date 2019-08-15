# React State Hooks的闭包陷阱，在使用Hooks之前必须掌握

先看以下代码

```jsx
import React from 'react'
import ReactDOM from 'react-dom'

const buttonStyles = {
  border: '1px solid #ccc',
  background: '#fff',
  fontSize: '2em',
  padding: 15,
  margin: 5,
  width: 200,
}
const labelStyles = {
  fontSize: '5em',
  display: 'block',
}

function Stopwatch() {
  const [lapse, setLapse] = React.useState(0)
  const [running, setRunning] = React.useState(false)

  React.useEffect(
    () => {
      if (running) {
        const startTime = Date.now() - lapse
        const intervalId = setInterval(() => {
          setLapse(Date.now() - startTime)
        }, 0)
        return () => clearInterval(intervalId)
      }
    },
    [running],
  )

  function handleRunClick() {
    setRunning(r => !r)
  }

  function handleClearClick() {
    setRunning(false)
    setLapse(0)
  }

  return (
    <div>
      <label style={labelStyles}>{lapse}ms</label>
      <button onClick={handleRunClick} style={buttonStyles}>
        {running ? 'Stop' : 'Start'}
      </button>
      <button onClick={handleClearClick} style={buttonStyles}>
        Clear
      </button>
    </div>
  )
}

function App() {
  const [show, setShow] = React.useState(true)
  return (
    <div style={{textAlign: 'center'}}>
      <label>
        <input
          checked={show}
          type="checkbox"
          onChange={e => setShow(e.target.checked)}
        />{' '}
        Show stopwatch
      </label>
      {show ? <Stopwatch /> : null}
    </div>
  )
}

ReactDOM.render(<App />, document.getElementById('root'))
```

这里的代码想要实现的功能如下：

1 点击 Start 开始执行 interval，并且一旦有可能就往 lapse 上加一

2 点击 Stop 后取消 interval

3 点击 Clear 会取消 interval，并且设置 lapse 为 0

但是这个例子在实际执行过程中会出现一个问题，那就是在 interval 开启的情况下，直接执行 clear，会停止 interval，但是显示的 lapse 却不是 0，那么这是为什么呢？

出现这样的情况主要原因是：useEffect 是异步的，也就是说我们执行 useEffect 中绑定的函数或者是解绑的函数，**都不是在一次 setState 产生的更新中被同步执行的。**啥意思呢？我们来模拟一下代码的执行顺序：

在我们点击来 clear 之后，我们调用了 setLapse 和 setRunning，这两个方法是用来更新 state 的，所以他们会标记组件更新，然后通知 React 我们需要重新渲染来。

然后 React 开始来重新渲染的流程，并很快执行到了 Stopwatch 组件。

注意以上都是同步执行的过程，所以不会存在在这个过程中 setInterval 又触发的情况，所以在更新 Stopwatch 的时候，如果我们能同步得执行 useEffect 的解绑函数，那么就可以在这次 JavaScript 的调用栈中清除这个 interval，而不会出现这种情况。

是恰恰因为 useEffect 是异步执行的，他要在 React 走完本次更新之后才会执行解绑以及重新绑定的函数。那么这就给 interval 再次触发的机会，这也就导致来，我们设置 lapse 为 0 之后，他又在 interval 中被更新成了一个计算后的值，之后才被真正的解绑。

那么我们如何解决这个问题呢？

## 使用 useLayoutEffect

useLayoutEffect 可以看作是 useEffect 的同步版本。使用 useLayoutEffect 就可以达到我们上面说的，在同一次更新流程中解绑 interval 的目的。

那么同学们肯定要问了，既然 useLayoutEffect 可以避免这个问题，那么为什么还要用 useEffect 呢，直接所有地方都用 useLayoutEffect 不就好了。

这个呢主要是因为 useLayoutEffect 是同步的，如果我们要在 useLayoutEffect 调用状态更新，或者执行一些非常耗时的计算，可能会导致 React 运行时间过长，阻塞了浏览器的渲染，导致一些卡顿的问题。这块呢我们有机会再单独写一篇文章来分析，这里就不再赘述。

## 不使用 useLayoutEffect

当然我们不能因为 useLayoutEffect 非常方便得解决了问题所以就直接抛弃 useEffect，毕竟这是 React 更推荐的用法。那么我们该如何解决这个问题呢？

在解决问题之前，我们需要弄清楚问题的根本。在这个问题上，我们之前已经分析过，就是因为在我们设置了 lapse 之后，因为 interval 的再次触发，但是又设置了一次 lapse。那么要解决这个问题，就可以通过避免最新的那次触发，或者在触发的时候判断如果没有 running，就不再设置。

用 useLayoutEffect 显然属于第一种方法来解决问题，那么我们接下去来讲讲第二种方法。

按照这种思路，我们第一个反应应该就是在 setInterval 的回调中加入判断：

```js
const intervalId = setInterval(() => {
  if (running) {
    setLapse(Date.now() - startTime)
  }
}, 0)
```

但是很遗憾，这样做是不行的，因为这个回调方法保存了他的闭包，而在他的闭包里面，running 永远都是true。那么我们是否可以通过在 useEffect 外部声明方法来逃过闭包呢？比如下面这样：

```js
function updateLapse(time) {
  if (runing) {
    setLapse(time)
  }
}

React.useEffect(() => {
  //...
  setInterval(() => {
    updateLapse(/* ... */)
  })
})
```

看上去 updateLapse 使用的是直接外部的 running，所以不是 setInterval 回调保存的闭包来。但是可惜的是，这也是不行的。因为 updateLapse 也是 setInterval 闭包中的一部分，在这个闭包当中，running 永远都是一开始的值。

可能看到这里大家会有点迷糊，主要就是对于闭包的层次的不太理解，这里我就专门提出来讲解一下。

在这里我们的组件是一个函数组件，他是一个纯粹的函数，没有 this，同理也就没有 this.render 这样的在 ClassComponent 中特有的函数，所以每次我们渲染函数组件的时候，我们都是要执行这个方法的，在这里我们执行 Stopwatch。

那么在开始执行的时候，我们就为 Stopwatch 创建来一个作用域，在这个作用域里面我们会声明方法，比如 updateLapse，他是在这次执行 Stopwatch 的时候才声明的，每一次执行 Stopwatch 的时候都会声明 updateLapse。同样的，lapse 和 running 也是每个作用域里单独声明的，**同一次声明的变量会出于同一个闭包，不同的声明在不同的闭包。**而 useEffect 只有在第一次渲染，或者后续 running 变化之后才会执行他的回调，所以对应的回调里面使用的闭包，也是每次执行的那次保存下来的

这就导致了，在一个 useEffect 内部是无法获知 running 的变化的，这也是 useEffct 提供第二个参数的原因。

那么是不是这里就无解了呢？明显不是的，这时候我们需要考虑使用 useReducer 来管理 state

### 逃出闭包

我们先来看一下使用 useReducer 实现的代码：

```js
import React from 'react'
import ReactDOM from 'react-dom'

const buttonStyles = {
  border: '1px solid #ccc',
  background: '#fff',
  fontSize: '2em',
  padding: 15,
  margin: 5,
  width: 200,
}
const labelStyles = {
  fontSize: '5em',
  display: 'block',
}

const TICK = 'TICK'
const CLEAR = 'CLEAR'
const TOGGLE = 'TOGGLE'

function stateReducer(state, action) {
  switch (action.type) {
    case TOGGLE:
      return {...state, running: !state.running}
    case TICK:
      if (state.running) {
        return {...state, lapse: action.lapse}
      }
      return state
    case CLEAR:
      return {running: false, lapse: 0}
    default:
      return state
  }
}

function Stopwatch() {
  // const [lapse, setLapse] = React.useState(0)
  // const [running, setRunning] = React.useState(false)

  const [state, dispatch] = React.useReducer(stateReducer, {
    lapse: 0,
    running: false,
  })

  React.useEffect(
    () => {
      if (state.running) {
        const startTime = Date.now() - state.lapse
        const intervalId = setInterval(() => {
          dispatch({
            type: TICK,
            lapse: Date.now() - startTime,
          })
        }, 0)
        return () => clearInterval(intervalId)
      }
    },
    [state.running],
  )

  function handleRunClick() {
    dispatch({
      type: TOGGLE,
    })
  }

  function handleClearClick() {
    // setRunning(false)
    // setLapse(0)
    dispatch({
      type: CLEAR,
    })
  }

  return (
    <div>
      <label style={labelStyles}>{state.lapse}ms</label>
      <button onClick={handleRunClick} style={buttonStyles}>
        {state.running ? 'Stop' : 'Start'}
      </button>
      <button onClick={handleClearClick} style={buttonStyles}>
        Clear
      </button>
    </div>
  )
}

function App() {
  const [show, setShow] = React.useState(true)
  return (
    <div style={{textAlign: 'center'}}>
      <label>
        <input
          checked={show}
          type="checkbox"
          onChange={e => setShow(e.target.checked)}
        />{' '}
        Show stopwatch
      </label>
      {show ? <Stopwatch /> : null}
    </div>
  )
}

ReactDOM.render(<App />, document.getElementById('root'))
```

在这里我们把 lapse 和 running 放在一起，变成了一个 state 对象，有点类似 Redux 的用法。在这里我们给 TICK action 上加了一个是否 running 的判断，以此来避开了在 running 被设置为 false 之后多余的 lapse 改变。

那么这个实现跟我们使用 updateLapse 的方式有什么区别呢？最大的区别是我们的 state 不来自于闭包，在之前的代码中，我们在任何方法中获取 lapse 和 running 都是通过闭包，而在这里，state 是作为参数传入到 Reducer 中的，也就是不论何时我们调用了 dispatch，在 Reducer 中得到的 State 都是最新的，这就帮助我们避开了闭包的问题。

其实我们也可以通过 useState 来实现，原理是一样的，我们可以通过把 lapse 和 running 放在一个对象中，然后使用

```js
updateState(newState) {
  setState((state) => ({ ...state, newState }))
}
```

这样的方式来更新状态。这里最重要的就是给 setState 传入的是回调，这个回调会接受最新的状态，所以不需要使用闭包中的状态来进行判断。具体的代码我这边就不为大家实现来，大家可以去试一下，最终的代码应该类似下面的（没有测试过）：

```js
const [state, dispatch] = React.useState(stateReducer, {
  lapse: 0,
  running: false,
})

function updateState(action) {
  setState(state => {
    switch (action.type) {
      case TOGGLE:
        return { ...state, running: !state.running }
      case TICK:
        if (state.running) {
          return { ...state, lapse: action.lapse }
        }
        return state
      case CLEAR:
        return { running: false, lapse: 0 }
      default:
        return state
    }
  })
}
```

## 总结

相信看到这里大家应该已经有一些自己的心得了，关于 Hooks 使用上存在的一些问题，最主要的其实就是因为函数组件的特性带来的作用域和闭包问题，一旦你能够理清楚那么你就可以理解很多了。

