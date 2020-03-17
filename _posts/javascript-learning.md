---
title: js学习教程
date: 2018-04-24
updated: 2018-04-24
tags:
  - js
---

JavaScript学习教程
本文只是对js学习的一些记录，内容非常简单，对js有了解的人可以直接略过

# JavaScript简介
## JS是脚本语言
- js是轻量级的脚本语言
- 可插入html页面的编码代码
- 由浏览器执行
- 简单容易学习
<!-- more -->
## JS能干什么
- 直接写入HTML输出流

```
 document.write("<h1>test</h1>");
```

- 对事件的反应

```
<button type="button" onclick="alert('欢迎!')">点我!</button>
```

- 改变HTML内容

```
x=document.getElementById("demo")  //查找元素
x.innerHTML="Hello JavaScript";    //改变内容
```

- 改变 HTML 图像
- 改变 HTML 样式

```
x=document.getElementById("demo")  //找到元素 
x.style.color="#ff0000";           //改变样式
```

- 验证输入

```
if isNaN(x) {alert("不是数字")};
```

# JavaScript的用法
JS脚本必须位于<script>和</script>标签之间，脚本可以放在HTML页面的<head>和<body>中
## JavaScript函数和事件
一般js会在页面加载时执行。通常，我们需要在某个事件发生时执行代码，比如当用户点击按钮时。如果我们把 JavaScript 代码放入函数中，就可以在事件发生时调用该函数
## JavaScript脚本存放位置
- <head>中的js函数
- <body>中的js函数
- 外部的js函数，通过src属性指定js文件

# JavaScript的输出
- 使用windows.alert()弹出告警框
- 使用document.write()方法将内容写入到HTML中
- 使用innerHTML写入到HTML元素中
- 使用console.log()写入到控制台中

# JavaScript的语法
## JavaScript的字面量
- 数字，整数，小数，科学计数(e)
- 字符串，以""或者''
- 表达式字面量，3 + 4
- 数组，[1,2,3,4]
- 对象(Object)，类似python的字典，{firtName:"John", lastName:"Qin", ages:50}
- 函数(Function),function(a,b) {return a+b}
## JavaScript变量
- 通过关键字var定义
- 变量通过变量名访问，字面量是恒定的值
## JavaScript的操作符
- 赋值，算术和位运算符，=  +  -  *  /
- 条件，比较及逻辑运算符， ==  != <  > 
## JavaScript注释
- 单行注释，使用//后的内容为注释
- 多行注释，使用`/*`和`*/`
## 代码行拆分
只能在文本字符串中使用反斜杠对代码行进行换行

```
document.write("你好 \
世界!");
```
不过不能这样拆分

```
document.write \
("你好 世界!");
```

## JavaScript的数据类型
- 数字，字符串，Boolean，数组，对象，Null，未定义(Undefine)
- Undefined 这个值表示变量不含有值。
- 可以通过将变量的值设置为 null 来清空变量。
## JavaScript的对象
- {firtName:"John", lastName:"Qin", ages:50}
- 拥有属性和方法的数据
- 方法的定义，methodName: function() { code line}

```
var person = {
    firstName: "John",
    lastName : "Doe",
    id : 5566,
    fullName : function() 
    {
       return this.firstName + " " + this.lastName;
    }
};
```

- 方法的访问，object.methodName()
## JavaScript的作用域
- 作用域为可访问变量，对象，函数的集合。
- 局部变量只能在函数内访问
- 全局变量具有全局作用域，在网页所有脚本和函数中均可以使用
- 如果变量在函数中没有声明，则为全局变量
- 在HTML中，全局变量是window对象，所有的数据变量都属于window对象

```
//此处可使用 window.carName
 
function myFunction() {
    carName = "Volvo";
}
```

## JavaScript事件 
HTML 事件是发生在 HTML 元素上的事情,当在 HTML 页面中使用 JavaScript 时， JavaScript 可以触发这些事件。
### 常见的HTML事件
- onchange ：HTML 元素改变
- onclick： 用户点击 HTML 元素
- onmouseover：用户在一个HTML元素上移动鼠标
- onmouseout：用户从一个HTML元素上移开鼠标
- onkeydown： 用户按下键盘按键
- onload： 浏览器已完成页面的加载

## JavaScript的循环
- for循环： 循环代码块一定的次数
- for/in: 循环遍历对象的属性

```
var person={fname:"John",lname:"Doe",age:25}; 
 
for (x in person)  // x 为属性名
{
    txt=txt + person[x];
}
```
- while: 当指定的条件为 true 时循环指定的代码块
- do/while: 同样当指定的条件为 true 时循环指定的代码块,但至少执行一次

## JavaScript的标签与Break，Continue的使用
- continue 语句（带有或不带标签引用）只能用在循环中。
- break 语句（不带标签引用），只能用在循环或 switch 中
- 通过标签引用，break 语句可用于跳出任何 JavaScript 代码块

## null和undefined的区别
值相同，类型不同
```
typeof undefined             // undefined
typeof null                  // object
null === undefined           // false
null == undefined            // true
```

## JavaScript数据类型 
- 5种不同的数据类型，string ，number，object，boolean，function
- 3种不同的对象类型，Date，Object，Aarry
- 不含任何值的数据类型，null，undefined

### 使用typeof操作符查看数据类型
- NaN的数据类型是number
- 数组(Aarry)的数据类型是object
- 日期(Date)的数据类型是object
- null的数据类型是object
- 未定义的变量的数据类型是undefined

```
typeof "John"                 // 返回 string 
typeof 3.14                   // 返回 number
typeof NaN                    // 返回 number
typeof false                  // 返回 boolean
typeof [1,2,3,4]              // 返回 object
typeof {name:'John', age:34}  // 返回 object
typeof new Date()             // 返回 object
typeof function () {}         // 返回 function
typeof myCar                  // 返回 undefined (如果 myCar 没有声明)
typeof null                   // 返回 object
```

### constructor 属性
constructor 属性返回所有 JavaScript 变量的构造函数

```
"John".constructor                 // 返回函数 String()  { [native code] }
(3.14).constructor                 // 返回函数 Number()  { [native code] }
false.constructor                  // 返回函数 Boolean() { [native code] }
[1,2,3,4].constructor              // 返回函数 Array()   { [native code] }
{name:'John', age:34}.constructor  // 返回函数 Object()  { [native code] }
new Date().constructor             // 返回函数 Date()    { [native code] }
function () {}.constructor         // 返回函数 Function(){ [native code] }
```

你可以使用 constructor 属性来查看对象是否为数组 (包含字符串 "Array"):

```
function isArray(myArray) {
    return myArray.constructor.toString().indexOf("Array") > -1;
}
```

## JavaScript Json 
- JSON.parse()： 用于将一个 JSON 字符串转换为 JavaScript 对象。
- JSON.stringify()： 用于将 JavaScript 值转换为 JSON 字符串。

## JavaScript验证API
### 约束验证 DOM 方法 

| Property | Description
| :--|:--|
| checkValidity() | 如果 input 元素中的数据是合法的返回 true，否则返回 false。|
| setCustomValidity() | 设置 input 元素的 validationMessage 属性，用于自定义错误提示信息的方法。使用 setCustomValidity 设置了自定义提示后，validity.customError 就会变成true，则 checkValidity 总是会返回false。如果要重新判断需要取消自定义提示 | 

### 约束验证 DOM 属性 

| Property | Description | 
| :--|:--|
| validity | 布尔属性值，返回 input 输入值是否合法 |
| validationMessage | 浏览器错误提示信息 |
| willValidate | 指定 input 是否需要验证 |

### Validity 属性 
input 元素的 validity 属性包含一系列关于 validity 数据属性:

| Property | Description | 
| :--|:--|
| customError | 设置为 true, 如果设置了自定义的 validity 信息。|
| patternMismatch | 设置为 true, 如果元素的值不匹配它的模式属性。|
| rangeOverflow | 设置为 true, 如果元素的值大于设置的最大值。|
| rangeUnderflow | 设置为 true, 如果元素的值小于它的最小值。|
| stepMismatch | 设置为 true, 如果元素的值不是按照规定的 step 属性设置。|
| tooLong | 设置为 true, 如果元素的值超过了 maxLength 属性设置的长度。|
| typeMismatch | 设置为 true, 如果元素的值不是预期相匹配的类型。 |
| valueMissing | 设置为 true，如果元素 (required 属性) 没有值 |
| valid | 设置为 true，如果元素的值是合法的。 |

## JavaScript函数 
### 函数定义 

```
function x(a, b) {
    return a * b 
}
```

### Function()构造函数 
函数同样可以通过内置的 JavaScript 函数构造器（Function()）定义。

```
var myFunction = new Function("a", "b", "return a * b ")

var x = myFunction(2,3)
```

### 函数提升(Hoisting) 
- 提升（Hoisting）是 JavaScript 默认将当前作用域提升到前面去的的行为。
- 提升（Hoisting）应用在变量的声明与函数的声明。

```
myFunction(5);

function myFunction(y) {
    return y * y;
}
```

### 自调用函数

```
var x = (function (a, b) { return a * b})(1, 2) // 1 * 2
```

### 函数也是对象

- 使用typeof操作判断函数类型将返回`function`
- 函数更想是一个对象，有属性和方法(arguments.length, toString())

### Arguments对象

JavaScript函数有个内置的对象Arguments对象， arguments对象包含函数调用的参数数组。

```
x = findMax(1, 123, 500, 115, 44, 88);
 
function findMax() {
    var i, max = arguments[0];
    
    if(arguments.length < 2) return max;
 
    for (i = 0; i < arguments.length; i++) {
        if (arguments[i] > max) {
            max = arguments[i];
        }
    }
    return max;
}
```

```
x = sumAll(1, 123, 500, 115, 44, 88);
 
function sumAll() {
    var i, sum = 0;
    for (i = 0; i < arguments.length; i++) {
        sum += arguments[i];
    }
    return sum;
}
```

### 参数传递

- 通过值传递函数参数，不会改变原来的值
- 通过对象传递函数参数，在函数中修改对象的属性值，在函数外会修改对象的原来的初始值。

### 函数调用

*this* 关键字： 在Javascript中，this指向函数执行时的当前对象。

#### 作为一个函数直接调用

```
function myFunction(a , b ){
    return a * b 
}
var x = myFunction(1,2)
```

#### 全局对象
当函数没有被自身的对象调用时 this 的值就会变成全局对象。

```
function myFunction(a , b ){
    return a * b 
}
var x = window.myFunction(1,2)
```

#### 函数作为方法调用

在 JavaScript 中你可以将函数定义为对象的方法。
以下实例创建了一个对象 (myObject), 对象有两个属性 (firstName 和 lastName), 及一个方法 (fullName):

```
var myObject = {
    firstName:"John",
    lastName: "Doe",
    fullName: function () {
        return this.firstName + " " + this.lastName;
    }
}
myObject.fullName();         // 返回 "John Doe"
```

#### 使用构造函数调用函数
如果函数调用前使用了 new 关键字, 则是调用了构造函数。
这看起来就像创建了新的函数，但实际上 JavaScript 函数是重新创建的对象：

```
// 构造函数:
function myFunction(arg1, arg2) {
    this.firstName = arg1;
    this.lastName  = arg2;
}
 
// This    creates a new object
var x = new myFunction("John","Doe");
x.firstName;                             // 返回 "John"
```

#### 作为函数方法调用函数
在 JavaScript 中, 函数是对象。JavaScript 函数有它的属性和方法。call() 和 apply() 是预定义的函数方法。 两个方法可用于调用函数，两个方法的第一个参数必须是对象本身。

```
function myFunction(a, b) {
    return a * b;
}
myObject = myFunction.call(myObject, 10, 2);     // 返回 20
```

```
function myFunction(a, b) {
    return a * b;
}
myArray = [10, 2];
myObject = myFunction.apply(myObject, myArray);  // 返回 20
```

> 两个方法都使用了对象本身作为第一个参数。 两者的区别在于第二个参数： apply传入的是一个参数数组，也就是将多个参数组合成为一个数组传入，而call则作为call的参数传入（从第二个参数开始）。
> 在 JavaScript 严格模式(strict mode)下, 在调用函数时第一个参数会成为 this 的值， 即使该参数不是一个对象。
> 在 JavaScript 非严格模式(non-strict mode)下, 如果第一个参数的值是 null 或 undefined, 它将使用全局对象替代。

# JavaScript HTML DOM
当网页加载时，浏览器一般会创建一个DOM（Documents Object Model）对象，通过JavaScript脚本能够改变HTML页面的元素内容，元素属性，CSS及对页面中所有事情做出反应。

## 查找HTML元素 
- 通过id查找： documents.getElementById("id")
- 通过类名去查找： documents.getElementByClassName("className")
- 通过标签名查找： documents.getElementByTagName("p")

```
<div id="main">
<p> DOM 是非常有用的。</p>
<p>该实例展示了  <b>getElementsByTagName</b> 方法</p>
</div>
<script>
var x=document.getElementById("main");
var y=x.getElementsByTagName("p");
document.write('id="main"元素中的第一个段落为：' + y[0].innerHTML);
</script>
```

## 修改HTML元素内容
- 改变HTML输出流： document.write("你好")
> 在文档加载完成使用该方法，会修改整个页面

- 改变HTML元素的内容： documents.getElementById("id").innerHTML= 新的内容
- 改变HTML元素的属性： documents.getElementById("id").*attribute* = 新的属性

## 改变HTML的样式

- 改变HTML元素的样式: documents.getElementById("id").style.*property* = *新样式*

## HTML DOM事件 
对事件的反应时执行JavaScript脚本
常见的HTML事件例子：
- 当用户点击鼠标时
- 当网页已加载时
- 当图像已加载时
- 当鼠标移动到元素上时
- 当输入字段被改变时
- 当提交 HTML 表单时
- 当用户触发按键时

```
<!DOCTYPE html>
<html>
<body>
<h1 onclick="this.innerHTML='Ooops!'">点击文本!</h1>
</body>
</html>
```

## HTML元素（节点）
### 创建新的HTML元素

```
<div id="div1">
<p id="p1">这是一个段落。</p>
<p id="p2">这是另一个段落。</p>
</div>

<script>
var para=document.createElement("p");
var node=document.createTextNode("这是一个新段落。");
para.appendChild(node);

var element=document.getElementById("div1");
element.appendChild(para);
</script>
```

### 删除已有的HTML元素

```
<div id="div1">
<p id="p1">这是一个段落。</p>
<p id="p2">这是另一个段落。</p>
</div>
<script>
var parent=document.getElementById("div1");
var child=document.getElementById("p1");
parent.removeChild(child);
</script>
```

# JavaScript对象 
## JavaScript对象的创建
- 直接创建实例

```
person=new Object();
person.firstname="John";
person.lastname="Doe";
person.age=50;
person.eyecolor="blue";
```

```
person={firstname:"John",lastname:"Doe",age:50,eyecolor:"blue"};
```

- 使用对象构造器

```
定义对象构造器
function person(firstname,lastname,age,eyecolor)
{
    this.firstname=firstname;
    this.lastname=lastname;
    this.age=age;
    this.eyecolor=eyecolor;
}

创建新的对象实例
var myFather=new person("John","Doe",50,"blue");
var myMother=new person("Sally","Rally",48,"green");
```

## JavaScript的for/in循环
for...in 循环中的代码块将针对每个属性执行一次。

```
var person={fname:"John",lname:"Doe",age:25}; 
 
for (x in person)
{
    txt=txt + person[x];
}
```



