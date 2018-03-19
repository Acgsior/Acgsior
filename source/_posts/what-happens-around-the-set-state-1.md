---
title: setState调用前后发生了什么#1
date: 2018-03-16 10:14:27
tags: [React, Frontend]
categories: [前端攻城尸]
icon: fa-code
---
# setState调用前后发生了什么 #1

### 多个setState的中间状态，以及setState和callback的执行顺序

上几天在思考在这样的情况下：

```` javascript
  handleClick = () => {
    this.setState({ count: this.state.count + 1 }, () => {
      const { count } = this.state
      console.log('== callback count#1: ', count)
    })

    // ...

    this.setState({ count: this.state.count + 1 }, () => {
      const { count } = this.state
      console.log('== callback count#2: ', count)
    })
  }
  
````

做出这个尝试其实是在想知道在callback中是否能否拿到第一次`setState()`后的state的状态，事实上是不能：

````
== callback count#1:  1
== callback count#2:  1
````

然后去debug才发现，`setState()`的callback其实是在`componentDidUpdate()`(`render()`之后)才调用的，所以拿到的state当然是所有的setState执行后的结果。

````
setState(partialState, callback) * N => render() => componentDidUpdate() => callback()
````

然后跑去翻[官方文档](https://reactjs.org/docs/react-component.html#setstate)找到了这样的描述：

> `setState()` does not always immediately update the component. It may batch or defer the update until later. This makes reading this.state right after calling `setState()` a potential pitfall. **Instead, use `componentDidUpdate` or a `setState` callback (`setState(updater, callback)`), either of which are guaranteed to fire after the update has been applied.** If you need to set the state based on the previous state, read about the updater argument below.

````
(prevState, props) => stateChange
````

> prevState is a reference to the previous state. It should not be directly mutated. Instead, changes should be represented by building a new object based on the input from prevState and props.
> Both prevState and props received by the updater function are **guaranteed to be up-to-date**. The output of the updater is shallowly merged with prevState.

所以将代码进行修改为使用`setState((prevState, props) => stateChange)`的方式：

```` javascript
  handleClick = () => {
    this.setState((prevState) => {
      const { count } = prevState
      console.log('== callback count#1: ', count)
      return { count: count + 1 }
    })

    this.setState((prevState) => {
      const { count } = prevState
      console.log('== callback count#2: ', count)
      return { count: count + 1 }
    })
  }
````

结果正确，通过updater的方式是可以拿到位于多个`setState`中间状态的state的：

````
== callback count#1:  0
== callback count#2:  1
````

### setState()真的是异步的?

> `setState()` does not always immediately update the component. It may batch or defer the update until later.

`setState()`的实现也很简单，将每次的state的更改(partialState)放入一个队列里，然后再依次更新：

```` javascript 
ReactComponent.prototype.setState = function(partialState, callback) {
  // ...
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
}

````

#### setState()也有同步的时候

但是在实际工作中遇到过Promise中调用`setState`的时候会立即更新state导致component重新render，也就是说并不是前面所描述的异步批量更新。类似下面的情况：

```` javascript
  handleClick = () => {
    Promise.resolve().then(() => {
      const { count } = this.state
      console.log('== count#1: ', count)
      this.setState({ count: count + 1 })
      console.log('== count#2: ', this.state.count)
    })
  }
````

得到的结果是：

````
== count#1:  0
== count#2:  1
````

还有其他的方法也可以同样得到类似的结果：

```` javascript
  handleClick = () => {
    setTimeout(() => {
      const { count } = this.state
      console.log('== count#3: ', count)
      this.setState({ count: count + 1 })
      console.log('== count#4: ', this.state.count)
    })
  }
````

得到的结果是：

````
== count#3:  0
== count#4:  1
````

#### 什么时候setState会是同步的

在[Dan Abramov](https://github.com/gaearon)的twitter上找到了这样的回答：
> [`setState()` is currently synchronous if you're not inside an event handler.](https://twitter.com/dan_abramov/status/959507572951797761)
> 
> 如果没有在(React)事件处理中，目前`setState()`就是同步的。

换句话说，`setState`的并不具备异步的特性，说它是异步的则是指的被异步调用。

React认为通常情况下会有下面2种情况会需要调用`setState()`:

- Component挂载(mount)的期间
- Component挂载后的事件触发

所以当前只在这两种情况下保证`setState`会被异步的处理。

而上面同步的`setState`的例子刚好不在React的事件处理中，所以它们是同步的。