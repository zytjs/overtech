---
title: useState 的全部
date: '2021-07-21'
spoiler: 一文搞懂 useState
cta: 'react'
---

## 初始化
```jsx
const [state, setState] = useState()
```

> React 保证 setState 函数在重新渲染中是不变的，所以可以在 useEffect 或 userCallback 的依赖列表中省略

### 懒惰初始化
```jsx
function Table(props) {
  // ⚠️ createRows() is called on every render
  const [rows, setRows] = useState(createRows(props.count));
  // ...
}
```
```jsx
function Table(props) {
  // ✅ createRows() is only called once
  const [rows, setRows] = useState(() => createRows(props.count));
  // ...
}
```
这俩的区别是，第一个是函数执行，相当于
```jsx
const value = createRows(props.count);
const [rows, setRows] = useState(value);
```
所以每次都执行，但 rows 的值是不变的，除非调用 setRows ；第二个是函数定义，只会在初始化的时候执行一次。

## 更新
useState 没有 this.setState 那样的回调函数，可以在确保 state 改变后再做些什么。但是可以借助 useEffect 来实现。
```jsx
const [value, setValue] = useState(0)

function () {
  setValue(newValue)
}

useEffect(() => {
  // ...
}, [value])
```

如果更新返回的值和当前状态完全相同，那么将完全跳过后续的重新渲染。使用 Object.is 进行值的对比。

## setState 是异步还是同步？
严格来说，不是异步也不是同步。setState 有时候不会即时更新是因为 React 的优化机制，在事件处理器中批量处理更新。而在某些时候，setState 会同步更新。分别看一下这两种情况。

### batch
在事件处理器中，React 会批量处理更新，比如
```jsx
const onClick = () => {
  this.setState({a: 1})
  this.setState({a: 2})
}
```
组件只会更新一次

而在异步代码（promise、async/await、setTimeout/setInterval、fetch）中的更新，不会批量处理，
比如:
```jsx
const onClick = () => {
  callAPI().then(() => {
    this.setState({a: 1})
    this.setState({a: 2})
  })
}
```
组件会更新两次，我们称这样的更新是 outside of react event handlers, 发生在 react 事件处理器之外，此时的回调在 react 执行机制完成之后进行，react 没办法批量更新。

### 那么为什么 setState 要设计成“异步”的，或者说为什么要有 batch？

简单概括就是为了避免多次更新。
如果 setState 是同步的，多次调用 setState 时，react 就有多次重新渲染，而有些渲染是没有必要的。


> ？？：多次调用 setState 时，react 就有多次重新渲染。不会影响 setState 之后的逻辑执行吗？

> 不会。react 在class组件中异步调用 setState 时，会先进行状态更新、re-render，然后按生命周期有序执行，在生命周期完成之后再继续执行 setState 之后的逻辑。
> 但是在函数式组件中，react 在 setState、re-render 之后，会先执行 setState 之后的逻辑（直到遇到另一个 setState），然后再去按生命周期执行。
> 区别如下：
> class 式组件
> ```jsx
> state = {
>    a: 0,
>    b: 0
>  }
>
>  componentDidMount() {
>    setTimeout(() => {
>      this.setState({a: 1})
>      console.log('a', this.state.a)
>
>      this.setState({b: 1})
>      console.log('b', this.state.b)
>    }, 1000)
>  }
>
>  componentDidUpdate() {
>    console.log('updated', this.state.a, this.state.b)
>  }
>
>  render() {
>    const {a, b} = this.state
>    return <div>{a} {b}</div>
>  }
> ```
> 输出：updated 1 0 , a 1 , updated 1 1 , b 1
> 函数式组件
> ```jsx
>   const [a, setA] = useState(0)
>   const [b, setB] = useState(0)
>
>   useEffect(() => {
>      setTimeout(() => {
>        setA(1)
>        console.log('a')
>
>        setB(1)
>        console.log('b')
>      }, 1000)
>    }, [])
>
>    useEffect(() => {
>      console.log('effect', a, b)
>    }, [a, b])
>
>    return <div>{a} {b}</div>
> ```
> 输出：effect 0 0 , a , effect 1 0 , b , effect 1 1


## 那么如何优化无法批量更新的更新呢？
1. 把 state 整合进一个 object，这样多个更新就变成了一个更新
2. 可以使用 React 提供的一个 API，手动强制批量更新。
```jsx
promise.then(() => {
  ReactDom.unstable_batchedUpdates(() => {
    this.setState({a: true}); // Doesn't re-render yet
    this.setState({b: true}); // Doesn't re-render yet
    this.props.setParentState(); // Doesn't re-render yet
  })
  // When we exit unstable_batchedUpdates, re-renders once
})
```
React event handlers 默认被包含在 unstable_batchedUpdates，所以它们可以批量更新。在其他地方可以强制使用 unstable_batchedUpdates，当 unstable_batchedUpdates 结束时，更新会被刷新到界面。
3. useReducer
