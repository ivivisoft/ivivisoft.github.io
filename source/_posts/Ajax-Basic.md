---
title: Ajax基础
date: 2022-01-25 22:23:07
category: 
- Javascript
tags:
- Ajax
---

### Ajax简介

ajax技术的核心是XMLHttpRequest对象(简称XHR),她使用户不需要刷新页面就能从
服务器获取数据.代码地址[github](https://github.com/ivivisoft/ajax)

### Ajax的使用方法

#### 获取XMLHttpRequest对象

正常情况下只需要 new XMLHttpRequest()即可,但是要兼容IE7以前的浏览器,则需
要做一些额外的操作,index.js文件中的createXHR()方法就是做了这些事情.

#### 发送请求

发送请求包括同步请求和异步请求.

同步请求一般步骤:

1. 使用xhr的open方法,open方法第一个参数表示调用方法的方式get,post等,第二个参
   数表示URL地址,第三个参数表示是否异步发送,false表示否,true表示异步发送.

   ```
   注意!此处的URL只能向同一个域中使用相同端口和协议的URL发送请求,
   否则存在跨域问题.
   ```

2. 使用send方法发送请求,send方法接受一个参数,既要作为请求主体发送的数据,如果不
   需要参数,一定要传null.

3. 返回结果:响应返回的结果会自动的填充到xhr对象上:

   - responseText:作为响应主体返回的文本
   - responseXML:如果响应的内容类型是”text/xml”或”application/xml”,这个
     属性中将保存包含着响应数据的XML DOM文档,对于非xml的响应,其为null
   - status:响应的HTTP状态,200表示成功,304表示服务端数据没有改变,可以使用浏览
     器缓存中的数据
   - statusText:HTTP状态说明

异步请求:
异步方式依赖于XHR对象的readyState属性,她表示请求/响应过程的当前活动阶段,取值如下:

- 0:未初始化.尚未调用open()方法

- 1:启动.已经调用open方法,但尚未调用send方法

- 2:发送.已经调用send方法,但尚未接收到响应

- 3:接收.已经接收到部分响应数据

- 4:完成.已经接收全部响应数据,而且已经可以在客户端使用了

  注意!onreadystatechange的函数没有使用this对象,
  原因是onreadystatechange事件处理的作用域问题.如果使用
  this对象在有的浏览器会执行失败,或者导致错误.所以使用实际
  的XHR对象实例变量是较为可靠的一种方式.另外在接收到响应之
  前,可以调用xhr.abort()来取消异步请求

#### HTTP头信息说明

在默认情况下,在发送XHR请求时,还会发送下列头部信息:

- Accept:浏览器能够处理的内容类型

- Accept-Charset:浏览器能够显示的字符集

- Accept-Encoding:浏览器能够处理的压缩编码

- Accept-Language:浏览器当前设置的语言

- Connection:浏览器与服务器之间连接的类型

- Cookie:当前页面设置的任何Cookie

- Host:发送请求的页面所在的域

- Referer:发送请求的页面URI

- User-Agent:浏览器的用户代理字符串

- 使用setRequestHeader可以设置自定义的请求头部,她接收两个参数,
  第一个是头部的名称,后面一个是头部字段的值.

  注意!要成功发送请求头部信息,必须在调用open方法之后且
  调用send方法之前,调用setRequestHeader方法.设置的头
  部名称的值不要和默认发送的一样,会导致错误.

#### GET请求说明

GET请求要把查询字符串添加到URL后面,以便传给服务器.对于XHR而言,位于传入open方法的URL末尾的查询字符串必须经过正确的编码才行.index.js中提供了一个简单的方法addURLParam()实现了这个功能

#### POST请求说明

由于XHR最初设计主要是为了处理XML的,因此可以传入XML DOM文档,传入的文档经序列化之后将作
为请求主体被提交到服务器.当然也可以传入任何想传给服务器的字符串
注意!默认情况下,服务器对POST请求和提交web表单的请求并不会一视同仁.因此服务器端必须有程序
来读取发送过来的原始数据,并解析出有用的部分.不过我们可以使用XHR来模仿表单提交:

1. 首先将Content-Type头部信息设置为application/x-www-form-urlencoded.
2. 以适当的格式创建一个字符串.也就是表单数据序列化.

针对于Ajax,POST请求带来的一个问题就是表单序列化.
先看一下,在表单提交期间,浏览器是怎样将数据发送给服务器的:

1. 对表单字段的名称和值进行URL编码,使用和号(&)分割.
2. 不发送禁用的表单字段
3. 只发送勾选的复选框和单选按钮
4. 不要发送type为”reset”和”button”的按钮
5. 多选选择框的每个选中的值单独一个条目.
6. 在单击提交按钮提交表单的情况下,也会发送提交按钮;否则不发送提交按钮.也包括type为”image”的input元素
7. select元素的值,就是选中的option元素的value特性的值.如果option元素没有value特性,则是option元素的文本值index.js的serialize方法提供了form表单的序列化功能.

#### 特别说明

form表单中的button如果不指定type类型,默认为submit类型,会触发表单提交,解决方法有两种:

1. 指定type为button
2. onclick事件上加return false;
