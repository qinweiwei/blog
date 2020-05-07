---
title: React 简介
date: 2020-04-25 09:03:22
tags: React, HTML
---

React 简介

# What is React？
> React 是一个用于构建用户界面的 JavaScript 库

```
ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('root')
);
```
以上将在页面上展示一个"Hello world"的标题。


# 如何使用

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

# React核心概念

## JSX
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
## 组件

> 组件允许你将 UI 拆分为独立可复用的代码片段，并对每个片段进行独立构思。

组件自定义了一个标签，从概念类似一个JavaScript函数，它接受任意的入参（即"props"），并返回用于描述页面展示的内容的React元素。

### 组件的定义方式
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

### Props

*props*相当于组件的入参，其具有只读属性。组件的attribute和子元素都为组件的props

### State

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
### 生命周期方法

* componentDidMount() 当组件被挂载到DOM中时调用
* componentWillUnmount() 当组件被从DOM中移除时调用

### 事件处理

* React事件的命名采用小驼峰式，不是全小写
* 使用JSX语法时需要传入的一个函数作为处理函数，而不是字符串
* 在React中，阻止默认行为，不能通过返回false处理，必须显示调用e.preventDefault()
* class的方法不会默认绑定this，如果忘记绑定，在事件处理回调函数中使用this，则this值为undefined，解决方法如下：
  1.绑定this变量
  ```
  class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = {isToggleOn: true};

    // 为了在回调中使用 `this`，这个绑定是必不可少的
    this.handleClick = this.handleClick.bind(this);
  }
  ```
  2.  使用public class field语法
```
  handleClick = () => {
    conole.log("this is ", this)
  }
```
  3. 在回调中使用箭头函数
```
class LoggingButton extends React.Component {
  handleClick() {
    console.log('this is:', this);
  }

  render() {
    // 此语法确保 `handleClick` 内的 `this` 已被绑定。
    return (
      <button onClick={() => this.handleClick()}>
        Click me
      </button>
    );
  }
}
```
### 列表

前文说过JSX支持表达式的引用，表达式中可以直接引用列表，或者使用map()方法生成列表
```
const ListTest = (props) => {
  const numbers = [1, 2, 3, 4, 5];
  const listItems = numbers.map((number) =>
  (<li>{number}</li>)
  );
  return (
    <ul>{listItems}</ul>
  )
}

```
### 关键字key
对于列表，React建议在标签中增加一个属性key，作为该列表的唯一标识，Key帮助React识别哪些元素改变了
```
function ListItem(props) {
  // 正确！这里不需要指定 key：
  return <li>{props.value}</li>;
}

function NumberList(props) {
  const numbers = props.numbers;
  const listItems = numbers.map((number) =>
    // 正确！key 应该在数组的上下文中被指定
    <ListItem key={number.toString()}              value={number} />

  );
  return (
    <ul>
      {listItems}
    </ul>
  );
}
```
### 表单

* 受控组件

> 渲染表单的 React 组件还控制着用户输入过程中表单发生的操作。被 React 以这种方式控制取值的表单输入元素就叫做“受控组件”。

* text input

```
class NameForm extends React.Component {
  state = {value : ''}

  handleChange(event) {
    this.setState({value: event.target.value})
  }

  handleSubmit = (e) => {
    alert('提交的名字' + this.state.value)
    e.preventDefault()
  }
  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        <label htmlFor="name">
          名字:
        </ label>
        <input id="name" type="text" value={this.state.value} onChange={(e) => this.handleChange(e)} />
        <input type="submit" value="提交1" />
      </form>
    )
  }

}
```

* textarea标签，同input标签一样

* select标签, 在根标签上使用value属性

```
class FlavorForm extends React.Component {
  state = {value : ''}
  handleChange(e) {
    this.setState({value: e.target.value})
  }
  handleSubmit(e) {
    alert('select name:' + this.state.value )
    e.preventDefault()
  }
  render() {
    return (
      <form onSubmit={(e) => this.handleSubmit(e)} >
      <label htmlFor="name" >选择你喜欢的风味</label>
      <select defaultValue="lime" onChange={(e) => this.handleChange(e)}>
        <option value="grapefruit">葡萄柚</option>
        <option value="lime">酸橙</option>
        <option value="coconut">椰子</option>
        <option value="mango">芒果</option>
      </select>
      <input type="submit" value="提交" />
      </form>
    )
  }

}
```
* 表单中包括多个输入
当需要处理多个 input 元素时，我们可以给每个元素添加 name 属性，并让处理函数根据 event.target.name 的值选择要执行的操作。

```
class Reservation extends React.Component {
  state = {
    isGoing: true,
    numberOfGuests: 2
  }
  handleInputChange(event) {
    const value = event.target.name === 'isGoing' ? event.target.checked : event.target.value
    const name = event.target.name
    this.setState({
      [name] : value
    })
  }

  render() {
    return (
      <form>
        <label>
        参与：
        <input
          name="isGoing"
          type="checkbox"
          checked={this.state.isGoing}
          onChange={(e) => { this.handleInputChange(e)}} />
        </label>
        <br />
        <label>
        来宾人数：
        <input
          name="numberOfGuests"
          type="number"
          value={this.state.numberOfGuests}
          onChange={(e) => { this.handleInputChange(e)}}
          />
        </label>
      </form>
    )
  }

}
```
*注意： 这里用到了计算属性名称的用法更新对应的state值，即*
```
this.setState({
  [name]: value
});
等同于ES5
var partialState = {};
partialState[name] = value;
this.setState(partialState);
```

### 状态提升

当一个组件需要访问另一个组件的state时，按照数据流自上而下的原则，最后的方式是将组件的state提升为父组件的state，然后通过props传递给多个子组件。

### 组合与继承

在React中，


