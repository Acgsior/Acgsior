---
title: React Async Component HOC
date: 2018-02-03 17:55:54
tags: [React]
categories: [前端攻城尸]
icon: fa-code
---

As we known that fetching the remote data in the `componentDidMount` method if the react component depeonds on it.

So a basicly implemention is:

````javascript
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

Overwrite the `shouldComponentUpdate` if we keep the steady of component if it is only used for display.

````javascript
shouldComponentUpdate() {
  return false;
}
````

An additional enhancement is displaying different views of fetching/fetched condition, e.g. display a loading bar during fetching data to avoid screen stays on the white page, then display the page correctly once the data fetched. 

Just add a condition in `render` method.

```` javascript
  render() {
    const { data } = this.state;
    return (
      <div>{data ? data : 'loading...'}</div>
    );
  }
````

It's tedious to write every async component like this. These things should have a more convenient solution.

#### Inheritance vs High-Order Component(HOC)

> A higher-order component (HOC) is an advanced technique in React for reusing component logic. HOCs are not part of the React API, per se. They are a pattern that emerges from React’s compositional nature. 
>
> Concretely, **a higher-order component is a function that takes a component and returns a new component.**

In most simple way to explain the difference of inheritance and HOC:

- Inheritance means the **IS-A** relationship, it's a super type object.
- HOC actually is Composition, which means the **HAS-A** relationship, the behavior belongs to the other object.

The advantages of HOC:

- Isolation
- Interoperability
- Maintainability

#### Unsound implemention of Async Component HOC

This topic starts with finding a strange implemention of Redux action.

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

It's a typical singleton pattern, however, why he need to do such a thing?

So I follow the vine to get the melon:

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
Obviously it's not a good implementation with a number of defects:

- `AsyncContent` is **strongly coupled** with `asyncContent` method wrapper, which means separating them makes no sense.
- The first parameter of method `asyncContent` actually is defined as the structure a function which return a promise with a component as return value, so the check that `isFunction(loader) ? loader() : loader` is meaningless. (btw the naming of `component` may not appropriate.)
- The `mounting` state value could be replaced by checking component.
- The `React.createElement(newComponent)` of `AsyncContent` will always create a new wrapped async component once its parent component re-rendered, and it's the reason that the author creates a singleton reducx action.

    - [React Top-Level API#createElement()](https://reactjs.org/docs/react-api.html#createelement)

#### Proper Implemention of Async Component HOC (by TS)

```` javascript
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
- Use promise loader function to load remote data and pass to component as props
- Use force steady to make sure the component will not be re-render
- How to use:

```` javascript
const AsyncNameComp1 = asyncComponent(() => NameComponent, () => mockFetch());
````

- Find code at github project: [AsyncComponent](https://github.com/Acgsior/AcgsiorSamples/tree/master/src/AsyncComponent)
