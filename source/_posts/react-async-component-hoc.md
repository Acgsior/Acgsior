---
title: React Async Component Hoc
date: 2018-02-03 17:55:54
tags: [React]
categories: [前端攻城尸]
icon: fa-code
---

As we known that fetching the data in the `componentDidMount` method if the react component depeonds on.

So a basicly implemention is:

````javascript
type dataType = {
  // ...
}

interface AsyncComponentProps {
  loader: () => dataType;
}

interface AsyncComponentState {
  data: dataType;
}

class AsyncComponent extends React.Component<AsyncComponentProps, AsyncComponentState> {
  constructor(props:AsyncComponentProps) {
    super(props);
    
    this.state = { data: {}};
  }
  
  componentDidMount() {
  	const { loader } = this.props;
    const data = loader();
	this.setState({ data });
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

