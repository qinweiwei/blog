---
title: React 简介
date: 2020-04-25 09:03:22
tags: React, HTML
---

# React 简介

## What is React？
> React 是一个用于构建用户界面的 JavaScript 库

```
ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('root')
);
```
以上将在页面上展示一个"Hello world"的标题。


## 安装React并使用它

* 在html中直接添加React

  1. 直接添加一个DOM容器到HTML
   ```
   <!-- ... 其它 HTML ... -->

<div id="like_button_container"></div>

<!-- ... 其它 HTML ... -->
   ```
  2. 添加标签**script**引入React和自身的组件代码
   ```
     <!-- 加载 React。-->
  <!-- 注意: 部署时，将 "development.js" 替换为 "production.min.js"。-->
  <script src="https://unpkg.com/react@16/umd/react.development.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@16/umd/react-dom.development.js" crossorigin></script>

  <!-- 加载我们的 React 组件。-->
  <script src="like_button.js"></script>
   ```
  3. 编写自身的组件代码,例如如下
    ```
    'use strict';

const e = React.createElement;

class LikeButton extends React.Component {
  constructor(props) {
    super(props);
    this.state = { liked: false };
  }

  render() {
    if (this.state.liked) {
      return 'You liked this.';
    }

    return e(
      'button',
      { onClick: () => this.setState({ liked: true }) },
      'Like'
    );
  }
}
    ```
* 使用**Create React App**创建单页应用
  1.使用以下命令创建一个React应用.
  ```
  npx create-react-app my-app
  cd my-app
  npm start

  ```
  2. 删除src目录下的文件，编写自己的组件和css
* 使用其他工具链,后续研究
  1. Next.js
  2. Gatsby
  3. Parcel

## React核心概念

### JSX
```
 const element = <h1>Hello, world!</h1>;
```
JSX是一个JavaScript的扩展，可以很好地描述UI应该呈现出的它应有的交还形式。

* JSX中支持嵌入的表达式
```
const name = 'Josh Perez';
const element = <h1>Hello, {name}</h1>;

ReactDOM.render(
  element,
  document.getElementById('root')
);
```
* JSX中属性值需要使用表达式
```
const element = <img src={user.avatarUrl}></img>;
```
* React使用JSX，需要Babel将JSX转义为一个React.createElement()函数调用
```
const element = (
  <h1 className="greeting">
    Hello, world!
  </h1>
);
//等同于一下函数调用

const element = React.createElement(
  'h1',
  {className: 'greeting'},
  'Hello, world!'
);
```
### 组件

> 组件允许你将 UI 拆分为独立可复用的代码片段，并对每个片段进行独立构思。

组件自定义了一个标签，从概念类似一个JavaScript函数，它接受任意的入参（即"props"），并返回用于描述页面展示的内容的React元素。

#### 组件的定义方式
* 函数组件
```
//箭头函数
const  Welcome = (props) => {
    return (
        <div>
            Welcome , {props.name}
        </div>
    )
}
// function
function Welcome(props) {
    return (
        <div> Welcome, {proms.name} </div>
    )
}

ReactDOM.render(<Welcome name="Sara" />, document.getElementById('root'))

```
*注意： 组件的命名必须首字母大写，用于与JS变量区分（小驼峰命名）*

* Class类组件
```
class Welcome extends React.Component {
    render() {
        return (
            <div>Welcome, {this.props.name}</div>
        )
    }
}

ReactDOM.render(<Welcome name="Sara"/>, document.getElementById('root'))
```

#### Props

*props*相当于组件的入参，其具有只读属性。组件的attribute和子元素都为组件的props

#### State

*State*相当于组件的私有变量，可读可写
* State值不要直接修改，应该通过setState函数
* 当调用setState时，React会重新渲染组件
* setState函数为异步操作，React会将几个组件的setState进行合并，因此更新state值时不要依赖this.state, 如果确实需要，让 setState() 接收一个函数而不是一个对象。这个函数用上一个 state 作为第一个参数，将此次更新被应用时的 props 做为第二个参数
```
// Correct
this.setState(function(state, props) {
  return {
    counter: state.counter + props.increment
  };
});

```


