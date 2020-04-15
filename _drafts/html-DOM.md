---
title: html-DOM
tags: html, DOM
---
# DOM介绍

## 修改DOM中的Node元素

### create new nodes

* createElement()
  ```
  const p = document.createElement('p')


  ```
* createTextNode()
  ```
  document.createTextNode('i am a new text node')
  ```
* node.textContent 推荐使用该方式
  ```
  p.textContent = 'i am a new text node'
  ```
* node.innerHTML
  ```
  p.innerHTML = "I'm a paragraph with <strong>bold</strong> text."
  ```
### insert node to DOM

* node.appChild()
* node.insertBefore()
* node.replaceChild()

```
const todoList = document.querySelector('ul')
const newTodo = document.createElement('li')
newTodo.textContent = "Do homework"

todoList.appendChild(newTodo)

todoList.insertBefore(newTodo, todoList.firstElementChild)

todoList.replaceChild(newTodo, todoList.children[2])

```

### remove Nodes from DOM

* node.removeChild() ## remove child node
* node.remove() ## remove node
* node.innerHTML("") ## 不推荐

```
todoList.removeChild(todoList.lastElementChild)
todoList.firstElementChild.remove()

```
## DOM中的事件处理

### event Handlers and event Listeners

* event handler 处理事件的js方法
* event listeners 附加在某个element上的监听事件的监听器

### 如何监听某element的事件

* Inline Event Handler Attributes
```
### html
<button onclick="changeText()">Click me</button>

<p>Try to change me.</p>

### event.js

const changeText = () => {
  const p = document.querySelector('p')

  p.textContent = 'I changed because of an inline event handler.'
}
```
* Event Handler Properties
```
##不需要在html里修改
## event.js

// Function to modify the text content of the paragraph
const changeText = () => {
  const p = document.querySelector('p')

  p.textContent = 'I changed because of an event handler property.'
}

// Add event handler as a property of the button element
const button = document.querySelector('button')
button.onclick = changeText
```

* Event Listeners
  1. 使用addEventListener方法
  2. 支持多个对同一个事件的多个监听方法
  3. 通过removeEventListener()，移除监听
```
  // Function to modify the text content of the paragraph
const changeText = () => {
  const p = document.querySelector('p')

  p.textContent = 'I changed because of an event listener.'
}

// Listen for click event
const button = document.querySelector('button')
button.addEventListener('click', changeText)
```
## 常用的事件类型

* Mouse Event
![](https://i.loli.net/2020/04/15/e5Nr3TIcMUjDVlJ.png)

* form Event
![](https://i.loli.net/2020/04/15/qnG5ArjwRlFbtY7.png)

* Keyboard Events
![](https://i.loli.net/2020/04/15/7LW1CM2fk9wGmRl.png)

对于keyboard事件来说，event对象有明确的属性，使用event对象，我们可以访问以下属性
例如：

![](https://i.loli.net/2020/04/15/t6AXYHgRUGMTamc.png)

```
// Test the keyCode, key, and code properties
document.addEventListener('keydown', event => {
  console.log('key: ' + event.keyCode)
  console.log('key: ' + event.key)
  console.log('code: ' + event.code)
})

keyCode: 65
key: a
code: KeyA
```

## Event Object

Event事件对象可以作为监听函数的参数，其包括一系列的属性和方法，作为一个通用的event对象，每一种事件都有自己的event对象扩展，比如KeyboardEvent 和MouseEvent

* event obj作为监听函数的参数
```
// Pass an event through to a listener
document.addEventListener('keydown', event => {
  var element = document.querySelector('p')

  // Set variables for keydown codes
  var a = 'KeyA'
  var s = 'KeyS'
  var d = 'KeyD'
  var w = 'KeyW'

  // Set a direction for each code
  switch (event.code) {
    case a:
      element.textContent = 'Left'
      break
    case s:
      element.textContent = 'Down'
      break
    case d:
      element.textContent = 'Right'
      break
    case w:
      element.textContent = 'Up'
      break
  }
})
```

* target属性
  event对象的target属性是常用的event属性，event.target属性值表示为event对应的元素。这对于内嵌元素的event监听非常有用。当我们对一个element进行监听时，同时也会对该element下的内嵌element也会启动监听作用，event.target值为event发生时的具体元素。
```
<section>
  <div id="one">One</div>
  <div id="two">Two</div>
  <div id="three">Three</div>
</section>

const section = document.querySelector('section')

// Print the selected target
section.addEventListener('click', event => {
  console.log(event.target)
}) ## console输出的值为click具体点击的element，包括section或者某个div
```

