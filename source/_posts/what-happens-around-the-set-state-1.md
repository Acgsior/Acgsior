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

然后跑去翻官方文档找到了这样的描述：

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

#### setState()真的是异步的?