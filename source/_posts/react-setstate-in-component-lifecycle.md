---
title: React setState in Component Lifecycle
date: 2017-12-18 16:32:36
tags: [React]
categries: [前端攻城尸]
icon: fa-code
---
I noticed that the rule of eslint recommends that no `setState()` method invocation in specific lifecycle methods during previously eslint issues. However, I'm not quite sure what would happen if I use `setState()` method in the unrecommended methods.

---

### [eslint-plugin-react rules](https://github.com/yannickcr/eslint-plugin-react/tree/master/docs/rules) with `setState()`

- [Prevent direct mutation of this.state (react/no-direct-mutation-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-direct-mutation-state.md)
	- NEVER mutate `this.state` directly, as calling `setState()` afterwards may replace the mutation you made. Treat `this.state` as if it were immutable.
	- The only place that's acceptable to assign this.state is in a ES6 `class` component constructor.

- [Prevent using this.state within a this.setState (react/no-access-state-in-setstate)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-access-state-in-setstate.md)
	- This rule should prevent usage of `this.state` inside `setState` calls. Such usage of `this.state` might result in errors when two state calls are called in batch and thus referencing old state and not the current state.

- [Prevent usage of setState in componentDidMount (react/no-did-mount-set-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-did-mount-set-state.md)
	- Updating the state after a component mount will trigger a second `render()` call and can lead to property/layout thrashing.

- [Prevent usage of setState in componentDidUpdate (react/no-did-update-set-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-did-update-set-state.md)	- Updating the state after a component update will trigger a second `render()` call and can lead to property/layout thrashing.

- [Prevent usage of setState in componentWillUpdate (react/no-will-update-set-state)](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-will-update-set-state.md)
	- Updating the state during the component will update step can lead to indeterminate component state and is not allowed.

---

### `setState()` in `componentWillMount()`

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
Get the result below:

````
constructor: 0
componentWillMount: 0
render: 1
````

### `setState()` in `componentWillReceiveProps()`

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

Get the result below after updated props:

````
constructor: 0
componentWillMount: 0
render: 1
componentWillReceiveProps: 1
render: 2
````

All result met our expectations.

Multiple `setState()` calls are batched. [React may batch multiple setState() calls into a single update for performance.](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)

### `setState()` in `componentDidMount()`

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

Get the result below:

````
constructor: 0
render: 1
componentDidMount: 1
render: 2
````

The `render()` is trigger twice just like the rule told.

### `setState()` in `componentWillUpdate()`

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

Check the result after updated props but get the infinite loops exception:

````
Maximum update depth exceeded. This can happen when a component repeatedly calls setState inside componentWillUpdate or componentDidUpdate. React limits the number of nested updates to prevent infinite loops.

▶ 6 stack frames were collapsed.

> 16 |   this.setState({
  17 |     content: content.concat([`componentWillUpdate: ${this.renderCount}`])
  18 |   });
  

````

Let's add a max count for `componentWillUpdate()`.

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

Get the result below after updated props:

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

Multiple calls of `componentWillUpdate()` and `render()` which should be the rule tells that **lead to indeterminate component state**.

### `setState()` in `componentDidUpdate()`

````js
  componentDidUpdate() {
    const compDidUpdateRecord = `componentDidUpdate: ${this.renderCount}`;
    this.renderCounts.push(compDidUpdateRecord);
    if (this.compDidUpdateCount < 2) {
      this.compDidUpdateCount += 1;
      const { content } = this.state;
      this.setState({
        content: content.concat([`componentDidUpdate: ${this.renderCount }`])
      });
    }
  }
  
````

Add the condition to avoid infinite calls and get the result below after updated props:

````
constructor: 0
render: 1
render: 2
componentDidUpdate: 2
render: 3
componentDidUpdate: 3
render: 4
count of compDidUpdateCount: 2
````

It looks like the same as the result of call `setState()` in `componentWillUpdate()` that triggers the second render logic and easy to be indeterminate.

---

### Summary

- Follow the eslint react rules to avoid calling `setState()` in
	- `componentDidMount()`
	- `componentWillUpdate()`
	- `componentDidUpdate()`

- Think about updating the structure of state/props to avoid `setState()` in incorrect lifecycle steps.
- Find code at github project: [setState](https://github.com/Acgsior/setState)

---

### Some note during testing

- Component instance property`(this.xxx)` update will not trigger the `render()` even if the instance property is used in `render()`.

	- [props and state are both just JavaScript objects that trigger a re-render when changed.](https://reactjs.org/docs/faq-state.html#what-is-the-difference-between-state-and-props)