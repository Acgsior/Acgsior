---
title: React 16的异常处理 - componentDidCatch
date: 2018-03-20 10:43:49
tags: [React, Frontend]
categories: [前端攻城尸]
icon: fa-code
---
# React 16的异常处理 - componentDidCatch

在16版本的React中引入了错误分界(Error Boundary)组件的概念，以及一个新的生命周期方法`componentDidCatch`来对运行时的异常进行更加方便而优雅的处理。

### 在React 16之前的异常处理

在React 16之前的版本，并没有什么办法对运行时的异常进行统一的处理。

在工作中也会需要有某些情况需要禁止异常的抛出，只能采用：

- try...catch
- Promise.reject／Promise.then.catch

写出来也是惨不忍睹，通常情况下使用Promise的特性来处理会比包裹在try...catch块更显清晰简单，但把同步的方法强制转化成为异步的也实在是不妥。

### 错误分界(Error Boundary)组件

> Error boundaries are React components that catch JavaScript errors anywhere in their **child component tree**, log those errors, and display a fallback UI instead of the component tree that crashed. Error boundaries catch errors during rendering, in lifecycle methods, and in constructors of the whole tree below them.
> 错误分界组件是指在**子组件树**中的任何位置捕获错误、记录错误，并通过回调显示对应的UI，而不是崩溃的组件树的组件。错误分界组件会捕获整个子组件树的`render`方法、生命周期方法和构造函数中的错误。

所以错误分界组件概念的引入主要是为了避免UI层的错误导致整个应用程序的崩溃，并且给予了回调方法来进行恢复和提示。

> Error boundaries only catch errors in the components below them in the tree. An error boundary can’t catch an error within itself.
> 错误分界组件只会捕获组件树下层组件里的错误。错误分界组件不会捕捉自身的错误。

所以在实现错误分界组件的时候需要令自身尽可能的简单，确保没有错误。

### componentDidCatch的使用

> A class component becomes an error boundary if it defines this lifecycle method(`componentDidCatch`).
> 如果一个component定义了生命周期方法`componentDidCatch`，他就是一个错误边界方法。

具体使用就是添加在子组件树中有错误抛出的时候会被调用的`componentDidCatch`方法，改变state去渲染异常情况下应该显示的视图。

```` typescript
class ErrorBoundary extends React.Component<{}, { error: Error | null, errorInfo: React.ErrorInfo | null }> {
  constructor(props: {}) {
    super(props);
    this.state = { error: null, errorInfo: null };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    this.setState({
      error: error,
      errorInfo: errorInfo
    });
  }

  render() {
    if (this.state.errorInfo) {
      return (
        <div>
          <h3>Something went wrong.</h3>
          <details>
            <summary>{this.state.error && this.state.error.toString()}</summary>
            <p>{this.state.errorInfo.componentStack}</p>
          </details>
        </div>
      );
    }
    return this.props.children;
  }
}

````

所以如果在传给`ErrorBoundary`的子组件中抛出了未处理的异常／错误，就会显示有错误的提示以及错误产生的component的堆栈，并且无论是子组件的构造函数(`constructor`)，生命周期函数(e.g.`componentDidMount`)，还是`render`方法都能够正常的捕获异常。

PS: 示例代码在 [ErrorHandling](https://github.com/Acgsior/AcgsiorSamples/tree/master/src/ErrorHandling) @GitHub

> Only use error boundaries for recovering from unexpected exceptions; don’t try to use them for control flow.
> 仅使用错误分界来应对未捕获的异常；不要尝试用它来控制流程。

需要注意的是，官方表示只应该用来处理异常情况，而不是来处理业务流程。

### 错误分界的实现

- 在fiber分发(`dipatch`)的时候检查component的instance上是否有`componentDidCatch`方法，如果有就通过`scheduleCapture`对后续fiber(用于mount／update子组件树)的错误进行捕获。

```` javascript
  // ReactFiberScheduler.js@dispatch
  
  if (
    typeof ctor.getDerivedStateFromCatch === 'function' ||
    (typeof instance.componentDidCatch === 'function' &&
      !isAlreadyFailedLegacyErrorBoundary(instance))
  ) {
    scheduleCapture(sourceFiber, fiber, value, expirationTime);
    return;
  }
````

- 在fiber执行时遇到没有捕获的异常错误，调用`throwException`方法并将错误信息(errorInfo)保存在`updateQueue.capturedValues`。

```` javascript
  // ReactFiberUnwindWork.js@throwException
  
  if (
    (workInProgress.effectTag & DidCapture) === NoEffect &&
    ((typeof ctor.getDerivedStateFromCatch === 'function' &&
      enableGetDerivedStateFromCatch) ||
      (instance !== null &&
        typeof instance.componentDidCatch === 'function' &&
        !isAlreadyFailedLegacyErrorBoundary(instance)))
  ) {
    ensureUpdateQueues(workInProgress);
    const updateQueue: UpdateQueue<
      any,
    > = (workInProgress.updateQueue: any);
    const capturedValues = updateQueue.capturedValues;
    if (capturedValues === null) {
      updateQueue.capturedValues = [value];
    } else {
      capturedValues.push(value);
    }
    workInProgress.effectTag |= ShouldCapture;
    return;
  }
````

- 在fiber执行完成(`completeWork`)时通过`effectTag`来表明记录了未捕获的错误。

```` javascript
  // ReactFiberCompleteWork.js@completeWork

  if (updateQueue !== null && updateQueue.capturedValues !== null) {
    workInProgress.effectTag &= ~DidCapture;
    if (typeof instance.componentDidCatch === 'function') {
      workInProgress.effectTag |= ErrLog;
    } else {
      // Normally we clear this in the commit phase, but since we did not
      // schedule an effect, we need to reset it here.
      updateQueue.capturedValues = null;
    }
  }
````

- 在fiber执行提交(`commitAllLifeCycles`)的时候通过`effectTag`判断是否遇到了未捕获的错误，并把错误信息传递给`componentDidCatch`。

```` javascript
  // ReactFiberScheduler.js@commitAllLifeCycles
  
  if (effectTag & ErrLog) {
    commitErrorLogging(nextEffect, onUncaughtError);
  }
````
```` javascript
  // ReactFiberCommitWork.js@commitErrorLogging
  
  const capturedErrors = updateQueue.capturedValues;
  updateQueue.capturedValues = null;

  if (typeof ctor.getDerivedStateFromCatch !== 'function') {
    // To preserve the preexisting retry behavior of error boundaries,
    // we keep track of which ones already failed during this batch.
    // This gets reset before we yield back to the browser.
    // TODO: Warn in strict mode if getDerivedStateFromCatch is
    // not defined.
    markLegacyErrorBoundaryAsFailed(instance);
  }

  instance.props = finishedWork.memoizedProps;
  instance.state = finishedWork.memoizedState;
  for (let i = 0; i < capturedErrors.length; i++) {
    const errorInfo = capturedErrors[i];
    const error = errorInfo.value;
    logError(finishedWork, errorInfo);
    instance.componentDidCatch(error);
  }
````

### 总结

React 16版本的错误分界较之前版本的使用者自己去各处捕获错误的优势：

- 声明式的生命周期实现，逻辑实现了原生的try...catch模式
	- try包裹的部分其实是错误分界component的子组件树
  	- catch部分的代码块是`componentDidCatch`本身的实现
- 模块化的错误捕获分割，保证页面剩余部分的正常可用，当然也更容易抽象和组合

#### 你可能需要

- [componentDidCatch](https://reactjs.org/docs/react-component.html#componentdidcatch)
- [Error Handling in React 16](https://reactjs.org/blog/2017/07/26/error-handling-in-react-16.html)
