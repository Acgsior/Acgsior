---
title: 奇怪的Promise用法
date: 2017-12-28 15:27:41
tags: [React, Frontend]
categories: [前端攻城尸]
icon: fa-code
---
# 奇怪的Promise用法

### 原由

前几天我在工作项目中的代码里一个“有趣”的javascript Promise的用例，左思右想也不太明白为什么会需要这样使用Promise。

### 这个有趣的Promise用例

稍微整理了下代码，删掉了不重要的和业务逻辑部分。

#### 伪代码：（React + Redux）

````javascript
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

#### 分析

1. 定义了一个方法返回Promise对象，并在当前redux action的全局作用域下拿到了Promise的`resolve`和`reject`两个状态控制方法。（很奇怪是吧？）
2. 定义了刚才的Promise分别在被`resolve`和被`reject`需要处理的逻辑。
3. 在其他的action方法里通过触发Promise的`resolve`和`reject`来进行对应的异步操作，换句话说通过promise的状态来表示业务逻辑的成功或者失败。

说实话我花了很长时间才明白他为什么要这样写，也是我第一次见到这样使用Promise。

### 总结

#### Resolve并不是Reject的相反面（Resolve is not the opposite of Reject）

事实上，Promise的`resolve`和`reject`状态的关系与逻辑上的true和false的关系并不相同，他们并不是完全的互斥关系。

因为在Promise链中任何的异常／错误都会导致promise直接进入`reject`状态，调用链后最近的`catch`块的逻辑；但是用promise的`reject`来处理业务上的失败逻辑的时候，真的能正确的处理系统异常的情况吗？

#### 这样做不好的其他理由

- 使Promise直接依赖外部对象的状态，而不是通常情况下的直接由自己决定。与此同时，Promise的异步调用链的代码被拆散，散落在不同的地方，结构凌乱。
- 这样的代码难以被理解和调试，同时也很难对结果进行预期和控制。

#### 可以这样做

想要稍微改进下这段代码其实很简单：拆分`resolve`和`reject`代码分别给各自的Promise。

对于上面的伪代码来说，直接让action1/action2各自拿到Promise对象然后通过`then`串联Promise链就可以了。

#### 你可能需要

- [MDN Promise docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#Creating_a_Promise)
- [Promises: resolve is not the opposite of reject](https://jakearchibald.com/2014/resolve-not-opposite-of-reject/)
