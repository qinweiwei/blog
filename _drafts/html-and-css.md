---
title: html and CSS
tags: html,CSS
notebook: 前端
---


## 响应式设计（Responsive design）

> "Responsive design” refers to the idea that your website should display equally well in everything from widescreen monitors to mobile phones.

### media queries
Responsive design都是通过媒体查询（media queries）来实现的，media queries格式如下图所示, 我们可以通过`@media`指令为不通的设备定义不同的**CSS**。

![Image.png](https://i.loli.net/2020/03/08/2S6BQyc7E5q4trX.png)@w=500

例如：通过media指令可以指定手机，平板和桌面等不同的CSS规则。

```

/* Mobile Styles */
@media only screen and (max-width: 400px) {
  body {
    background-color: #F09A9D; /* Red */
  }
}

/* Tablet Styles */
@media only screen and (min-width: 401px) and (max-width: 960px) {
  body {
    background-color: #F5CF8E; /* Yellow */
  }
}

/* Desktop Styles */
@media only screen and (min-width: 961px) {
  body {
    background-color: #B2D6FF; /* Blue */
  }
}
```

### 布局
* 流动布局（fluid layout）也称为自适应布局，宽度使用的百分比宽度来适应不通的屏幕分辨率，就想`flexible boxed`
    例如：

```
.footer-item {
  border: 1px solid #fff;
  background-color: #D6E9FE;
  height: 200px;
  flex: 1;
}
```
* 固定布局（fixed layout)：不管屏幕尺寸如何变化，布局宽度一直是固定大小的。
```
.page {
  width: 600px;
  margin: 0 auto;
}
```

### disabling viewport zooming
在响应式设计出现之前，移动端的布局仍然使用桌面布局，只是通过zoom，缩小至移动端的大小，在响应式设计出现以后，我们需要关闭viewport zooming，这样移动端的布局才会生效。
```

<met
name='viewport'
      content='width=device-width, initial-scale=1.0, maximum-scale=1.0' /

```




## Semantic HTML
>“Semantic HTML” refers to the idea that all your HTML markup should convey the underlying meaning of your content—not its appearance.

* 增加web网页元素的含义，使用搜索引擎，屏幕阅读器，以及其他的机器识别web的内容更加容器
* 方便web开发人员更好的维护网页内容
* 也被称为章节元素**sectioning elements**
![Image _2_.png](https://i.loli.net/2020/03/09/OnIHA5roFydla82.png)@w=500

### Outline
* 每个html文档都有一个提纲（outline），搜素引擎，屏幕阅读器通过outline查看web内容的层次关系
* heading elements组成了web的提纲
* 层次等级随着h1,h2,h3逐级降低，例如h2是h1元素的子章节
* 通过[HTML 5 Outliner](https://gsnedders.html5.org/outliner)

### semantic element
* **article**元素标识一个web网页的独立的文章，仅仅封装于哪些希望从网页摘要出来的内容。通过article元素告诉搜索引擎和其他的机器应用，标识的部分内容为整个网页的主要文章。同时一个web网页中支持多个**article**元素

* **section**元素跟article类似，但是外部的应用不会像使用article元素一样，提取出section元素中的内容。 在section中，使用heading element定义outline，h1,h2,h3..的层级关系是内嵌在section元素中的。
* **nav**元素标识标识网页的导航章节，该元素用来关联sidebar，表格，以及一组links
* **header**元素用来标识一个section，article和整个web网页的前言内容。
* **footers**元素用来标识整体网页，article，section的页脚部分，一般位于底部。
* **aside**元素不同于其他用来增加网页额外信息的元素，它一般用来告诉搜索引擎以及其他机器应用从整个网页是去掉的其封装的信息。其支持使用*class*属性通过CSS规则对其进行样式化。
* **time**元素用于标识时间和日期，其格式为:
![time_element.png](https://i.loli.net/2020/03/09/ULsVN9dRkBt6chQ.png)@w=500
* **address**元素同**time**一样，用于标识地址信息
* **figure**和**figcaption**元素，用于标识网页的图片信息和描述。

## Form(表单）
form元素一般是用来收集用户信息然后与后端服务进行交互的。其格式如下
```

<form action='' method='get' class='speaker-form'></form>
```
* 属性action定义了用于处理表单数据的URL地址，当action属性值为空时，表示提交给form所在的相同的URL地址
* 属性method的值为**GET**或者**POST**，定义如何提交表单数据给后端服务
* 样式化表单元素同`div`元素一样

### text input fields
```
## html

            <div class='form-row'>
                <label for='full-name'>Name</label>
                <input id='full-name' name='full-name' type='text' value="test"/>
            </div>



## CSS

            .form-row {
              margin-bottom: 40px;
              display: flex;
              justify-content: flex-start;
              flex-direction: column;
              flex-wrap: wrap;
            }

            .form-row input[type='text'] {
              background-color: #FFFFFF;
              border: 1px solid #D6D9DC;
              border-radius: 3px;
              width: 100%;
              padding: 7px;
              font-size: 14px;
            }

            .form-row label {
              margin-bottom: 15px;
            }

```

* label标签与语义化元素`header`类似，用于表单的label，其`for`属性的值应该与`input`属性的`id`值相同。此时用户在浏览器中点击label标签中的内容是，鼠标自动focus到input标签处。
* `input`标签通过`type`属性值为text定义该表达为文本输入类型
* `input`标签中的`value`属性定义了表单中的文本输入的默认值
* 通过class进行样式化，支持flex模型，另外可以通过属性选择器，对某一个特定类型的`input`进行样式化
>   .form-row input[type='text'] { /* ... */}

### Email input fields
```
## html

            <div class='form-row'>
                <label for='email'>Email</label>
                <input id='email' name='email' type='email' placeholder='joe@example.com' />
            </div>

## CSS

        .form-row input[type='text'],
        .form-row input[type='email'] {
          background-color: #FFFFFF;
          /* ... */
        }

        .form-row input[type='text']:invalid,
        .form-row input[type='email']:invalid {
          border: 1px solid #D55C5F;
          color: #D55C5F;
          box-shadow: none; /* Remove default red glow in Firefox */
        }
```
*  `input`标签通过`type`属性值为`email`定义该表单输入为邮箱输入类型
*  Email input feild会自动校验输入值是否带有`@`符号
* 可以通过伪类`invalid`对正确和错误的输入值进行不同的样式化
* 属性`placeholder`定义为占位符，用于提示用户
### radio buttons
```
## html
<fieldset class='legacy-form-row'>
  <legend>Type of Talk</legend>
  <input id='talk-type-1'
         name='talk-type'
         type='radio'
         value='main-stage' />
  <label for='talk-type-1' class='radio-label'>Main Stage</label>
  <input id='talk-type-2'
         name='talk-type'
         type='radio'
         value='workshop'
         checked />
  <label for='talk-type-2' class='radio-label'>Workshop</label></fieldset>

  ## CSS

.legacy-form-row {
  border: none;
  margin-bottom: 40px;
}

.legacy-form-row legend {
  margin-bottom: 15px;
}

.legacy-form-row .radio-label {
  display: block;
  font-size: 14px;
  padding: 0 20px 0 10px;
}

.legacy-form-row input[type='radio'] {
  margin-top: 2px;
}
```
* `type`值为*radio*定义为单选按钮
* 使用`fieldset`封装`radio`标签，且`field`标签使用`legend`用于label
* `fieldset`用于form内对元素进行分组，`legend`标签用于定义`fieldset`的标题
* 组内的每一个radio标签分配一个label元素
* 每个radio标签中的属性name值需要相同
* 每个radio标签使用不同的value属性
* checkd属性标识为默认选择项
* 注意fieldset标签不支持flex模型，只能使用float模型
### select element
```
## html
            <div class="form-row">
                <label for="t-shirt">T-shirt Size</label>
                <select name="t-shirt" id="t-shirt">
                    <option value="xs">Extra Small</option>
                    <option value="s">Small</option>
                    <option value="m">Medium</option>
                    <option value="l">Large</option>
                </select>
            </div>
## CSS
.form-row select {
  width: 100%;
  padding: 5px;
  font-size: 14px;
```
* *select*标签用于定义下拉菜单
* *option*标签定义下拉菜单中每一项内容，*value*属性定义选择该下拉项时的值。
* *select*标签的样式化是比较困难的，一般推荐使用**Bootstrap Dropdown**和**JQuery Selectmenu**等框架去进行样式化

### textarea

```
## html
<div class="form-row">
    <label for="abstract">Abstract</label>
    <textarea name="abstract" id="abstract" placeholder="请输入一段话"></textarea>
    <div class="instructions">Describe textarea</div>
</div>

##CSS
.form-row textarea {
  font-family: "Helvetica", "Arial", sans-serif;
  font-size: 14px;

  border: 1px solid #D6D9DC;
  border-radius: 3px;

  min-height: 200px;
  margin-bottom: 10px;
  padding: 7px;
  resize: none;
}
```

* *textarea*标签类似于*text input fields*，只是用来定义多行文本域
* 不同于*input*这种self-closing标签，textarea为双向标签
* 默认的浏览器允许用户缩放*textarea*域，通过属性*resize*值为none，则不允许用户就行缩放

### checkboxes
```
## html
<div class="form-row">
    <label for="available" class="checkbox-label">
        <input type="checkbox" id="available" name="available" value="is-available">
        <span>I am actually available the date of talk</span>
    </label>
</div>
## CSS
@media only screen and (min-width: 700px) {
  /* ... */
  .form-row .checkbox-label {
    margin-left: 120px;
    width: auto;
  }
}


```
* *input*标签中type值为*checkbox*时定义了多项选择
* 不同于*radio*,checkbox不需要fieldset标签分组
* 定义*width*值为*auto*后，可以使得全部的表单域都在单独的一行

### submit buttons

```
## html
<form action='' method='get' class='speaker-form' id="formname">
  <div class='form-row'>
    <button>Submit</button>
  </div>

  <div class="form-row">
      <input type="submit" value="Submit">
  </div>
</form>

<div class='form-row'>
    <button form="formname">Submit</button>
</div>

## CSS

.form-row input[type="submit"],
.form-row button {
  font-size: 16px;
  font-weight: bold;

  color: #FFFFFF;
  background-color: #5995DA;

  border: none;
  border-radius: 3px;

  padding: 10px 40px;
  cursor: pointer;
}
.form-row input[type="submit"]:hover,
.form-row button:hover {
  background-color: #76AEED;
}
```
* 表单的提交通过标签*button*和type为*submit*的input标签都可以
* 当button标签位于某个form中时，可以直接提交，否则需要将button的属性form值指为某个form的id
* type值为*submit*的input标签同上
* 建议使用button标签进行form提交
* 点击button时会验证form中的input的所有元素，然后提交表单内容至action的URL
* 提交form至URL的格式为:
  > speaker-submission.html?full-name=Rick&email=rick%40internetingishard.com&talk-type=workshop&t-shirt=l&abstract=Derp.&available=is-available
* 样式化button标签是一般会使用伪类如hover和active




