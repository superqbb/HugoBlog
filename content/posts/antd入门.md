---
title: "《Ant Design 实战教程》学习笔记"
date: 2019-12-28T10:19:48+08:00
draft: true
tags: ["React"]
series: ["antd"]
categories: ["入门教程笔记"]
---

#《Ant Design 实战教程》学习笔记
> 近几年，前端框架发生了很大的变化，最近比较火的有react，vue等等。
作为后端开发人员，有时候前端资源未到位时，可能需要自己兼顾前端开发，或者自己想搞一些小demo，小系统，需要通过一个好看的前端来提高系统颜值。阿里开源的antd，就是基于react的开源前端组件库，能够帮助我们快速构建漂亮的，符合现代化的前端。于是通过学习官方实战教程，并将一些基础的笔记记录在此。

《Ant Design 实战教程》：https://www.yuque.com/ant-design/course/intro

## 学习目的
* 通过antd官网入门教程，按照官方教程步骤，学习antd + react项目结构，搭建过程。
* 学习react入门基础知识
* 学习antd项目开发基础、调试方式、部署方式

##  学习笔记

### 一、初始化项目

#### 组件

React 的核心概念就是组件。

组件的好处有很多，下面是其中几点。

- 有利于细化 UI 逻辑，不同的组件负责不同的功能点。

- 有利于代码复用，多个页面可以使用同样的组件。

- 有利于人员分工，不同的工程师负责不同的组件。

#### JSX 语法特点

React 自创的 JSX 语法，JSX 可以被 Babel 转码器转为正常的 JavaScript 语法

```jsx
//转码前
export default () => {
  return <div> hello world </div>;
};

//转码后
exports.default = function () {
  return React.createElement(
    "div",
    null,
    "hello world"
  );
};
```



JSX 更易写也更易读，因此React开发者均使用JSX语法。

JSX 语法的特点就是，凡是使用 JavaScript 的值的地方，都可以插入这种类似 HTML 的语法。

```jsx
const element = <h1>Hello, world!</h1>;
```

注意点：

* 所有 HTML 标签必须是闭合的，没有闭合语法的标签，必须在标签尾部加上斜杠

  ```jsx
  // 报错
  <h1>Hello
  
  // 不报错
  <img src="" />
  ```

  

* 任何 JSX 表达式，顶层只能有一个标签，也就是说只能有一个根元素

  ```jsx
  // 报错
  const element = <h1>hello</h1><h1>world</h1>;
  
  // 不报错
  const element = <div><h1>hello</h1><h1>world</h1></div>;
  ```




> 规范：HTML 原生标签都使用小写，开发者自定义的组件标签首字母大写，比如`<MyComponent/>`



**JSX 语法允许 HTML 与 JS 代码混写**

```jsx
//JSX 语法的值的部分，只要使用了大括号{}，就表示进入 JS 的上下文，可以写入 JS 代码。
const element = (
  <h1>
    Hello, {formatName(user)}!
  </h1>
);
```

//todo 学习更多的jsx规范并将特点整理在此

更多jsx规范: https://reactjs.org/docs/introducing-jsx.html



#### React 组件语法

`ShoppingList.js` 示例

```jsx
import React from 'react';

class ShoppingList extends React.Component {
  render() {
    return (
      <div className="shopping-list">
        <h1>Shopping List for {this.props.name}</h1>
        <ul>
          <li>Instagram</li>
          <li>WhatsApp</li>
          <li>Oculus</li>
        </ul>
      </div>
    );
  }
}

export default ShoppingList;
```

* 自定义的组件必须继承`React.Component`这个基类
* 必须有一个`render`方法，给出组件的输出。

使用`shoppinglist.js`组件

```jsx
import React from 'React';
import ShoppingList from './shoppinglist.js';

class Content extends React.Component {
  render() {
    return (
      <ShoppingList name="张三" />
      
      /*
      写法2:
      <ShoppingList name="张三">
        {/* 插入的其他内容 */}
      </ShoppingList>
      */
    );
  }
}

export default Content;
```

* 新建了一个`Content`组件，里面使用了`ShoppingList`组件，并传入了`name`参数值
* `ShoppingList`是组件名，`name="张三"`表示这个组件有一个参数`name`

#### 组件的参数

* 组件内部，所有参数都放在`this.props`属性上面。参数机制，React 组件可以接受外部消息

* `this.props`对象有一个非常特殊的参数`this.props.children`，表示当前组件“包裹”的所有内容。

  > 这个属性在 React 里面有很大的作用，它意味着组件内部可以拿到，用户在组件里面放置的内容。

Show me the code

`Picture`：内部使用`props.children`，获取用户传入的内容

```jsx
const Picture = (props) => {
  return (
    <div>
      <img src={props.src} />
      {props.children}
    </div>
  )
}
```

**向`props.children`传入内容**

```jsx
render () {
  const picture = {
    src: 'https://cdn.nlark.com/yuque/0/2018/jpeg/84141/1536207007004-59352a41-4ad8-409b-a416-a4f324eb6d0b.jpeg',
  };
  return (
    <div className='container'>
      <Picture src={picture.src}>
        // 这里放置的内容就是 props.children
      </Picture>
    </div>
  )
}
```

#### 组件的状态 state

除了接受外部参数，组件内部也有不同的状态

React 规定，组件的内部状态记录在`this.state`这个对象

```jsx
//这个例子里面，使用内部状态来区分用户是否点击了按钮
class Square extends React.Component {
  //构造方法
  constructor(props) {
    super(props);
    //state定义了状态对象，这个对象里面有个value字段
    //设定value字段初始值为null
    this.state = {
      value: null,
    };
  }

  render() {
    return (
      //onClick监听函数执行this.setState()方法
      //React 使用这个方法，更新this.state对象
      <button
        className="square"
        onClick={() => this.setState({value: 'X'})}
      >
        {this.state.value}
      </button>
    );
  }
}
```

`this.setState`这个方法有一个特点，就是每次执行以后，它会自动调用`render`方法，导致 UI 更新。UI 里面使用`this.state.value`，输出状态值。随着用户点击按钮，页面就会显示`X`。



#### 生命周期方法

生命周期方法（lifecycle methods）: 组件的运行过程中，存在不同的阶段，React 为这些阶段提供了钩子方法

```jsx
class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

  componentDidMount() {
		//组件挂载后自动调用
  }

  componentWillUnmount() {
		//组件卸载前自动调用
  }

  componentDidUpdate() {
  	//UI 每次更新后调用（即组件挂载成功以后，每次调用 render 方法，都会触发这个方法）
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
```



### 二、使用 Ant Design 组件

##### 引入antd

在 umi 中，umi-plugin-react 中配置 antd 打开 antd 插件，antd 插件会帮你引入 antd 并实现按需编译。

```shell
# 安装antd依赖
cnpm install --save antd
```

```javascript
export default {
  plugins: [
    ['umi-plugin-react', {
      antd: true
    }],
  ],
  // ...
}
```

##### 使用antd组件

* `import`对应的组件

  ```jsx
  import { Card } from 'antd';
  ```

* 根据API 调整组件参数，实现不同的效果

  > [官方文档](https://ant.design/components/card/)

#### 受控组件与非受控组件

“受控”与“非受控”两个概念，区别在于这个组件的状态是否可以被外部修改







