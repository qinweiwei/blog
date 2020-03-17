---
title: go template学习
date: 2018-03-12
updated: 2018-03-12 16:08:00
tags:
    - golang
---

# go template 模块学习

golang提供了对模板的支持，其是数据驱动模板，data-driven template，分别在"text/template"和"html/template"两个包下，这两个包的api基本一致，只是"html/template"会将一些html中的关键符号（类似'<', '>'之类的）做些转义处理再输出具体可以参见[template官网](https://golang.org/pkg/text/template/)

<!-- more -->
## template模块的一些主要func

```
1. Template.Must(t *Template , err error) *Template #template有效性检查
2. Template.New(name string) *Template #生成template对象
3. func (t *Template) Parse(text string) (*Template, error) #解析模板字符串text
4. func (t *Template) ParseFiles(filenames ...string) (*Template, error) #解析模板文件
5. func (t *Template) Execute(wr io.Writer, data interface{}) error #执行模板替换
```

## 基本语法
### Actions
在template中用分隔符`{{`和`}}`标识，用于执行go template的语法，在Actions之外的文本都直接输出。 Actions包括以下几种类型：

```
- 注释 {{/*a comment */}}
- {{ pipeline }} , pipeline返回的值直接输出
- {{if pipeline}} T1 {{end}} 用于判断，如果pipeline的值不为空，则执行T1
- {{if pipeline}} T1 {{else}} T0 {{end}} ，用于条件判断，如果pipeline的值不为空，则执行T1，反之则执行T0
- {{range pipeline}} T1 {{end}} 用于循环处理，其中pipeline必须是array，map，slice类型
- {{template "name" pipeline}} ，引用template，其中pipeline的值为子模板的`.`对象
- {{with pipeline}} T1 {{else}} T0 {{end}} 如果pipeline值不为空，则执行T1，同时将pipeline赋值给T1的`.`对象
```

### 变量
变量是为于action之中的定义，其范围一般在action内部，对于`if`,`with`和`range`的control structure中，变量的范围到`end`，定义变量和引用变量的语法如下：
> $variable := pipeline
> range $index, $element := pipeline

### 函数
go template的函数包括两种，一种是全局的预置的函数，一种用户自己定义的模板函数
1. 全局预置函数：具体可见 [Functions](https://golang.org/pkg/text/template/#Functions)
2. 自定义模板函数
- template包创建新的模板的时候，支持.Funcs方法来将自定义的函数集合导入到该模板中，后续通过该模板渲染的文件均支持直接调用这些函数。
- 通过template.FuncMap对象生成自定义的函数对象，其中key为函数的名字，value为具体的执行函数
- 自定义的函数包括两种，一种直接返回可用值，一种返回两个值，一个返回值，一个error，用于中指模板渲染
- 调用方法是直接在action里调用
> {{funcname .arg1 .arg2}}


