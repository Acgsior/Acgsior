---
title: Strange Usage of Promise
date: 2017-12-28 15:27:41
tags: [React, Frontend]
categories: [前端攻城尸]
icon: fa-code
---
I found an "interesting" usage of Promise at work a few days ago. It looks like the author might be having misunderstood(or abused) the Promise.

----

See the pseudocode below:

````js
// in redux action
let _resolve;
let _reject;

const getPromise = () => new Promise((resolve, reject) => {
  _resolve = resolve;
  _reject = reject;
  
  const data = {};
  console.log('do something...');
  
  return data;
});

const action0 = () => {
  getPromise().then((data) => {
    console.log(`then do something: ${data}`);
    // in business success
    return data;
  }).catch((error) => {
    // in business fail
    console.log(`catch do something: ${error}`)
  });
};

const action1 = () => {
  // mock data
  const data = 'mock data';
  // in some conditions
  _resolve(data);
};

const action2 = () => {
  // mock error
  const error = 'mock error';
  // in some conditions
  _reject(error);
};


```` 

Firstly, he defines a method to return a Promise object and makes the `resolve` and `reject` handler as global variable references in redux action.

Then he defines the process of resolve state and rejected state for Promise by calling the previous method.

At last, he will trigger the `resolve` or `reject` handler in different actions(which means success or fail condition in business) to invoke the asynchronous operation in Promise.

Actually, it's the first time I've seen the usage of Promise which took a long time to understand the meaning. 

### Resolve is not the opposite of reject

The issue is making the state of Promise dependents on extrinsic objects directly.

The Promise should determine its state by itself which means all asynchronous operation need be promise wrapped. 

It's impossible to control the state completely from the outside even because the `resolve` is not the **opposite** of `reject`. In short, the state `resolve` and state `reject` is not the mutex relationship.

For example, an exception met in `then` block changes the state to `reject`, but the business fail logic in `catch` block should not be able to handle it.

Although we could use the solution above if we CAN make sure the code is safety, however, the code is still hard to read.

So we should really split the business success and fail to different the promise chain which means let the action1/action2 to link the `then` block after the Promise which `getPromise` method returned.

It's that simple.

##### References

- [MDN Promise docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#Creating_a_Promise)
- [Promises: resolve is not the opposite of reject](https://jakearchibald.com/2014/resolve-not-opposite-of-reject/)
