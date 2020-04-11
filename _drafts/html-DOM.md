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
