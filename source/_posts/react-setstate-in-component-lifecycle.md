---
title: setState方法的调用在React Component生命周期中表现
date: 2017-12-18 16:32:36
tags: [React]
categories: [前端攻城尸]
icon: fa-code
---
# setState方法的调用在React Component生命周期中表现

### 原由

在日常工作中修改ESLint问题的时候，我注意到ESLint的一些规则要求在React Component的某些特定的生命周期避免`setState`方法的调用，然而我并不确定在这些不推荐的生命周期方法中调用`setState`方法会发生什么。

### ESLint的规则

下面列出了ESLint和`setState()`方法相关的[规则](ESLint):

- [Prevent direct mutation of this.state (react/no-direct-mutation-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-direct-mutation-state.md)
	- NEVER mutate `this.state` directly, as calling `setState()` afterwards may replace the mutation you made. Treat `this.state` as if it were immutable.
	- 永远不要直接修改`this.state`，而是应该调用`setState()`来代替你的修改。把`this.state`当作是不可变的。
	- The only place that's acceptable to assign `this.state` is in a ES6 `class` component constructor.
	- 唯一允许直接赋值给`this.state`的地方是在ES6的 `class` component constructor。

- [Prevent using this.state within a this.setState (react/no-access-state-in-setstate)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-access-state-in-setstate.md)
	- This rule should prevent usage of `this.state` inside `setState` calls. Such usage of `this.state` might result in errors when two state calls are called in batch and thus referencing old state and not the current state.
	- 这条规则不允许在`setState`的方法內调用`this.state`。如果这样使用可能导致`this.state`处于错误的状态，因为两次state在合并处理中的修改调用只会引用之前的state，而不是当前的state。

- [Prevent usage of setState in componentDidMount (react/no-did-mount-set-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-did-mount-set-state.md)
	- Updating the state after a component mount will trigger a second `render()` call and can lead to property/layout thrashing.
	- 在component will mount方法中更新state会出发第二次的`render()`方法，并可能导致属性和布局的抖动。

- [Prevent usage of setState in componentDidUpdate (react/no-did-update-set-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-did-update-set-state.md)	
	- Updating the state after a component update will trigger a second `render()` call and can lead to property/layout thrashing.
	- 在component did update方法中更新state会出发第二次的`render()`方法，并可能导致属性和布局的抖动。

- [Prevent usage of setState in componentWillUpdate (react/no-will-update-set-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-will-update-set-state.md)
	- Updating the state during the component will update step can lead to indeterminate component state and is not allowed.
	- 在component will update方法中更新state会导致component的状态不明确，这样做是不可接受的。

### 如果我们真的这样做了...

PS: 下面使用的代码在 [set-state](https://github.com/Acgsior/setState) @GitHub

#### 在`componentWillMount()`中调用`setState`

````js
class StateComponent extends Component {
  constructor(props) {
    super(props);
    this.renderCount = 0;
    const constructorRecord = `constructor: ${this.renderCount}`;
    this.renderCounts = [constructorRecord];
    this.state = {
      content: [constructorRecord],
      times: []
    };
  }

  componentWillMount() {
    const { content } = this.state;
    const willMountRecord = `componentWillMount: ${this.renderCount}`;
    this.renderCounts.push(willMountRecord);
    this.setState({
      content: content.concat([willMountRecord])
    });
  }

  render() {
    this.renderCount += 1;
    this.renderCounts.push(`render: ${this.renderCount}`)
    const { times } = this.state;
    return (
      <div className='state-component normal-state-component'>
        <ul>
          {this.renderCounts.map((text, index) => (<li key={index}>{text}</li>))}
          <li>time: {times.map((time) => `${time}`)}</li>
        </ul>
      </div>
    );
  }
}

````

结果一切正常。

````
constructor: 0
componentWillMount: 0
render: 1
````

#### 在`componentWillReceiveProps()`中调用`setState`


````js
  componentWillReceiveProps(nextProps) {
    const { content, times } = this.state;
    const willReceiveRecord = `componentWillReceiveProps: ${this.renderCount}`;
    this.setState({
      content: content.concat([willReceiveRecord])
    });
    this.renderCounts.push(willReceiveRecord);
    (this.props.time !== nextProps.time) && this.setState({
      times: times.concat([nextProps.time])
    });
  }
  
````

得到的结果和我们期望的一致。

````
constructor: 0
componentWillMount: 0
render: 1
componentWillReceiveProps: 1
render: 2
````

多次的调用`setState()`会被合并处理：[React may batch multiple setState() calls into a single update for performance.](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)

#### 在`componentDidMount()`中调用`setState`

````js
  componentDidMount() {
    const { content } = this.state;
    const didMountRecord = `componentDidMount: ${this.renderCount}`;
    this.renderCounts.push(didMountRecord);
    this.setState({
      content: content.concat([didMountRecord])
    });
  }

````

得到的结果是`render()`方法被触发了两次，和ESLint的规则描述的一致。

#### 在`componentWillUpdate()`中调用`setState`

````js
  componentWillUpdate() {
    const compWillUpdateRecord = `componentWillUpdate: ${this.renderCount}`;
    this.renderCounts.push(compWillUpdateRecord);
    const { content } = this.state;
    this.setState({
      content: content.concat([`componentWillUpdate: ${this.renderCount}`])
    });
  }

````

在检查打印出来的结果的时候发现程序产生了infinite loops exception:

````
Maximum update depth exceeded. This can happen when a component repeatedly calls setState inside componentWillUpdate or componentDidUpdate. React limits the number of nested updates to prevent infinite loops.

▶ 6 stack frames were collapsed.

> 16 |   this.setState({
  17 |     content: content.concat([`componentWillUpdate: ${this.renderCount}`])
  18 |   });
  

````

看来我们需要限制下`componentWillUpdate()`的调用。

````js
  componentWillUpdate() {
    const compWillUpdateRecord = `componentWillUpdate: ${this.renderCount}`;
    this.renderCounts.push(compWillUpdateRecord);
    if (this.compDidUpdateCount < 2) {
      this.compDidUpdateCount += 1;
      const { content } = this.state;
      this.setState({
        content: content.concat([`componentWillUpdate: ${this.renderCount}`])
      });
    }
  }
  
````

得到的结果也比较奇怪，这种自循环大概就是ESLint规则中所说的**导致不确定的component state**(lead to indeterminate component state)

````
constructor: 0
render: 1
componentWillUpdate: 1
render: 2
componentWillUpdate: 2
render: 3
componentWillUpdate: 3
render: 4
count of componentWillUpdate: 2
````

### 总结

在React component的生命周期方法调用`setState()`需要一定的考虑，避免在`componentDidMount()`/`componentWillUpdate()`/`componentDidUpdate()`调用。

#### 你可能需要

- React Component的生命周期方法顺序：

````
// mount
constructor => componentWillMount => render => componentDidMount => render(maybe)

// update
componentWillReceiveProps => componentShouldUpdate(if true) => componentWillUpdate => render => componentDidUpdate

// setState
setState(with callback) => render => callback

````

- React Component的实例(instance)上的属性发生了改变并不会触发`render()`，即使是这个属性在`render()`方法中有使用。
