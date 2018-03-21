---
title: React Async Component HOC
date: 2018-02-03 17:55:54
tags: [React]
categories: [前端攻城尸]
icon: fa-code
---
# React Async Component HOC
### 原由

这次的起因也是因为在工作项目看到了同事在我看来有些不太合理的实现。

### 一般来说

众所周知，如果我们需要从远程异步的获取数据来渲染某个React Component，官方都会推荐在`componentDidiMount`这个生命周期方法中来发起这个异步请求。

大概是这样的：(typescript)

````typescript
interface AsyncComponentProps {
  loader: () => Promise<string>;
}

interface AsyncComponentState {
  data: string;
}

class AsyncComponent extends React.Component<AsyncComponentProps, AsyncComponentState> {
  constructor(props: AsyncComponentProps) {
    super(props);

    this.state = { data: '' };
  }
  
  componentDidMount() {
  	const { loader } = this.props;
    loader().then(data => this.setState({ data }));
  }
  
  render() {
    const { data } = this.state;
    return (
      <div>{data}</div>
    );
  }
}

````

如果期望component只会被初次渲染，之后再也不更新，我们可以重写`shouldComponentUpdate`方法来忽略之后传入的`props`的更新。

````typescript
shouldComponentUpdate() {
  return false;
}

````

一个可能的改进是让component在发起异步请求到获得远程返回的响应之间展示一个加载中的状态。比如：显示请求中的动画或是进度条来避免对应的区域保持空白，在完成了异步请求后在通过远程数据正确的渲染页面。

简单的实现是在`render`方法中添加判断。

```` typescript
  render() {
    const { data } = this.state;
    return (
      <div>{data ? data : 'loading...'}</div>
    );
  }

````

功能是实现了，但是如果每个需要异步获取数据component都需要写上这样一大段代码是宁人乏味 ，我们需要一种更加便捷的方法完成这种异步获取数据的component。

### 继承 vs 高阶组件（High-Order Component, HOC for short)

首先引用facebook官方的文档。

> A higher-order component (HOC) is an advanced technique in React for reusing component logic. HOCs are not part of the React API, per se. They are a pattern that emerges from React’s compositional nature. 
> 高阶组件是React重用component逻辑的一种高级技巧。本质上来说，HOC并不是React API的一部分，而是React的组合特性的模式体现。
> Concretely, **a higher-order component is a function that takes a component and returns a new component.**
> 具体而言，**高阶组件是输入一个component并返回一个新的component的方法**。

在我们需要逻辑重用的时候，通常需要考虑的是选择继承还是组合：

- 继承代表了一种**IS-A**的关系，通常父类型和子类型会高度耦合。
- 组合代表了一种**HAS-A**的关系，能保持调用者与被调用者的相对独立。

HOC作为组合的一种实现，有下列的优点：

- Isolation／隔离性
- Interoperability／互用性
- Maintainability ／ 可维护性

### Async Component HOC不太合理的实现

其实发现这个问题，首先是因为我发现了一个奇怪的redux action。

```` javascript
// actions/xxx.js
let cachedPromise;

const fetchxxxAsyncAction = () => dispatch => {
  if (!cachedPromise) {
    cachedPromise = dispatch(fetchxxx());
  }
  return cachedPromise;
};

````

如果你熟悉设计模式，很容易就能明白这是个延迟加载的单例模式。他在调用`fetchxxx`之前会先去判断`cachedPromise`变量是否已经有值，如果没有才会去加载；如果已经有值了就直接返回。

顺藤摸瓜找到了问题的所在。

```` javascipt
// AsyncContent.jsx
import createAsyncContent from './createAsyncContent';

class AsyncContent extends React.PureComponent {
  render() {
    const {
      loader,
      component,
      ...
    } = this.props;
    const newComponent = createAsyncContent(() => loader().then(() => component), { ... });
    return React.createElement(newComponent);
  }
}

// createAsyncContent.jsx
export default (loader, { ... }) =>
  class asyncContent extends React.Component {
    ...
    
    componentDidMount() {
      const component = isFunction(loader) ? loader() : loader;
      
      return component
        .then(module => this.mounting && this.setState({ component: module }))
        .catch(err => console.error(err));
    }
    
    render() {
      const { component } = this.state;
      return (
        <div>
          { component || this.renderLoader() }
        </div>);
    }
  };

````

稍微仔细的看看，可以发现很多的问题：

- `AsyncContent`和`createAsyncContent`是一个组合的关系，但是好像并没有什么理由需要拆分他们。
- 传给`createAsyncContent`的第一个参数被明确定义成为了返回一个promise的方法，所以在`createAsyncContent`的`componentDidMount`中再去检查`isFunction(loader) ? loader() : loader;`并没有什么意义。
- `createAsyncContent`中的state value`mounting`可以通过判断`component`来代替。
- `AsyncContent`的`render`方法调用了`React.createElement()`，所以只要`AsyncContent`重新渲染，总是会返回新生成的component instance, 这应该就是作者对action使用单例模式的原因。

### Async Component HOC正确的实现

参考文章开始部分正确的方式来实现HOC。(typescript)

PS: 下面使用的代码在 [AsyncComponent](https://github.com/Acgsior/AcgsiorSamples/tree/master/src/AsyncComponent)@GitHub

```` typescript
interface AsyncComponentState {
  component: React.ComponentClass | null;
  componentProps: {};
}

const asyncComponent = (
  importComponent: () => React.ComponentClass,
  loader: () => Promise<{}>,
  forceSteady: boolean = true) =>
  class extends React.Component<{}, AsyncComponentState> {
    constructor(props: {}) {
      super(props);

      this.state = {
        component: null,
        componentProps: {}
      };
    }

    shouldComponentUpdate(nextProps: {}, nextState: AsyncComponentState) {
      return nextState.component !== this.state.component || !forceSteady;
    }

    componentDidMount() {
      const { component } = this.state;
      
      if (component === null) {
        loader().then((componentProps) => {
          this.setState({
            component: importComponent(),
            componentProps
          });
        });
      }
    }

    render() {
      const { component: C, componentProps } = this.state;
      return (
        <div>
          {C !== null ? <C {...componentProps} /> : 'Loading...'}
        </div>
      );
    }
  };
  
````

- 通过Promise来异步的请求远程数据，并且作为`props`传给被包裹的`component`。
- 重写了`componentShouldUpdate`，通过指定参数`forceSteady`来保证component不会被再次渲染。

使用方法：

```` typescript
const AsyncNameComp1 = asyncComponent(() => NameComponent, () => mockFetch());
````

### 总结

Async Component HOC的好处在于： 低耦合的实现，HOC和被包裹的component可以各自独立工作，而并不相互依赖。

顺便一提，在实现功能的的时候还是应该先思考，保证思路的清晰而不是陷入X-Y的问题。

#### 你可能需要

- [Higher-Order Components](https://reactjs.org/docs/higher-order-components.html)
- [React Top-Level API#createElement()](https://reactjs.org/docs/react-api.html#createelement)
