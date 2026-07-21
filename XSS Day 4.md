# 🔥 XSS Day 4：DOM XSS + 前端代码审计

这一节开始进入更接近真实企业项目的阶段。

前面：

```text
反射型XSS
输入
 ↓
服务器
 ↓
HTML响应
 ↓
浏览器执行


存储型XSS
输入
 ↓
数据库
 ↓
页面输出
 ↓
浏览器执行
```

今天学习：

> **服务器可能完全没有漏洞，但是前端JavaScript自己把漏洞制造出来。**

这就是：

# DOM XSS

---

# 一、DOM XSS原理

## 1. 什么是DOM？

DOM：

```text
Document Object Model
文档对象模型
```

浏览器加载HTML后，会形成一个对象树：

例如：

HTML:

```html
<html>
<body>

<h1>Hello</h1>

</body>
</html>
```

浏览器内部：

```
Document
 |
 html
 |
 body
 |
 h1
 |
 "Hello"
```

JavaScript可以操作这个树。

例如：

```javascript
document.getElementById()
```

修改页面。

---

# 二、什么是DOM XSS？

核心：

> JavaScript读取了用户可控数据，然后把它放入危险位置。

数据流：

```
用户输入
   |
   ↓
JavaScript读取
   |
   ↓
危险函数
   |
   ↓
HTML解析
   |
   ↓
XSS执行
```

---

注意：

DOM XSS：

可能：

```
浏览器
   |
   |
   ↓

根本不经过服务器
```

---

# 三、反射型XSS vs DOM XSS

对比：

|类型|漏洞位置|
|---|---|
|反射型XSS|后端代码|
|存储型XSS|数据库+后端|
|DOM XSS|前端JavaScript|

---

例如：

## 反射型：

PHP：

```php
echo $_GET['name'];
```

漏洞。

---

## DOM：

HTML：

```html
<script>

let name = location.hash;

document.write(name);

</script>
```

漏洞。

---

# 四、DOM XSS核心：Source 和 Sink

这是企业代码审计重点。

---

## Source（输入源）

表示：

用户可以控制的数据来源。

常见：

---

### 1. URL参数

```javascript
location.search
```

例如：

URL：

```
http://test.com/?name=admin
```

获取：

```javascript
location.search
```

结果：

```
?name=admin
```

---

### 2. URL片段

```javascript
location.hash
```

例如：

```
http://test.com/#hello
```

JS：

```javascript
location.hash
```

得到：

```
#hello
```

---

### 3. Cookie

```javascript
document.cookie
```

---

### 4. LocalStorage

```javascript
localStorage.getItem()
```

---

# 五、Sink（危险输出点）

Sink：

表示：

数据进入危险位置。

---

## 1. innerHTML（重点）

危险：

```javascript
element.innerHTML=data;
```

---

例如：

HTML:

```html
<div id="box"></div>
```

JS:

```javascript
let input=location.hash;

box.innerHTML=input;
```

---

访问：

```
http://test.com/#<img src=x onerror=alert(1)>
```

流程：

```
location.hash

↓

<img src=x onerror=alert(1)>

↓

innerHTML

↓

浏览器解析

↓

执行
```

---

# 六、innerHTML为什么危险？

因为：

innerHTML会让浏览器重新解析HTML。

例如：

赋值：

```javascript
div.innerHTML="<h1>Hello</h1>";
```

浏览器：

创建：

```html
<h1>Hello</h1>
```

---

如果输入：

```html
<script>alert(1)</script>
```

浏览器可能解析。

---

所以：

危险：

```javascript
innerHTML
```

---

安全：

```javascript
innerText
```

或者：

```javascript
textContent
```

---

区别：

## innerHTML

输入：

```html
<b>Hello</b>
```

结果：

显示：

**Hello**

---

## textContent

结果：

显示：

```
<b>Hello</b>
```

只是文本。

---

# 七、JavaScript危险函数列表

企业代码审计重点关注：

---

## HTML解析类

危险：

```javascript
innerHTML
outerHTML
document.write()
```

---

## DOM操作

危险：

```javascript
insertAdjacentHTML()
```

---

## 动态执行

危险：

```javascript
eval()
```

例如：

```javascript
eval(userInput)
```

用户输入：

直接执行。

---

## 定时执行

危险：

```javascript
setTimeout()
setInterval()
```

例如：

```javascript
setTimeout(userInput,1000)
```

---

# 八、DVWA DOM XSS实验

进入：

```
DVWA

↓

XSS (DOM)
```

设置：

```
Security Low
```

---

页面通常：

有：

```
Choose your language:

English
French
German
```

---

观察URL变化。

例如：

选择：

English

URL：

```
http://lvh.me/DVWA/vulnerabilities/xss_d/?default=English
```

---

这里：

重点看：

```
default
```

---

# 九、Burp分析DOM XSS

打开：

Burp：

```
Proxy
HTTP history
```

观察：

请求：

```http
GET /DVWA/vulnerabilities/xss_d/?default=English
```

---

发送到：

```
Repeater
```

修改：

```
default=test
```

---

看响应。

重点：

搜索：

```
test
```

---

如果：

响应里面没有：

```
test
```

但是页面出现：

说明：

可能是：

JavaScript处理。

---

# 十、查看前端源码

浏览器：

F12

进入：

```
Sources
```

搜索：

```
default
```

或者：

```
location
```

---

可能看到：

类似：

```javascript
var lang = location.search.substring(1);

document.write(lang);
```

---

这里：

数据流：

```
location.search

↓

document.write()

↓

HTML解析

↓

XSS
```

---

# 十一、前端代码审计方法

真实项目：

不是看页面。

看：

JS文件。

流程：

---

## 第一步：寻找Source

搜索：

```
location
```

```
cookie
```

```
localStorage
```

```
URLSearchParams
```

---

例如：

```javascript
let id =
location.search;
```

记录：

输入点。

---

## 第二步：追踪数据流

例如：

```javascript
let a=location.hash;

let b=a;

div.innerHTML=b;
```

数据流：

```
hash
 |
 ↓
a
 |
 ↓
b
 |
 ↓
innerHTML
```

---

## 第三步：判断Sink危险程度

危险：

```
innerHTML
document.write
eval
```

安全：

```
textContent
innerText
```

---

# 十二、企业SPA框架中的XSS风险

现在企业大量：

- Vue
    
- React
    
- Angular
    

---

## Vue

正常：

```html
<div>
{{username}}
</div>
```

Vue默认：

转义。

安全。

---

危险：

```html
<div v-html="content">
</div>
```

类似：

```javascript
innerHTML
```

---

如果：

content来自用户：

存在XSS。

---

## React

正常：

```jsx
<div>
{name}
</div>
```

默认转义。

---

危险：

```jsx
<div
dangerouslySetInnerHTML={{
__html:data
}}
/>
```

类似：

innerHTML。

---

# 十三、企业修复DOM XSS

## 1. 避免危险函数

不要：

```javascript
innerHTML
```

改：

```javascript
textContent
```

---

## 2. 输出编码

例如：

HTML：

```
<  → &lt;
>  → &gt;
```

---

## 3. 使用安全模板

例如：

Vue：

默认插值：

```html
{{data}}
```

不要：

```html
v-html
```

---

## 4. CSP

限制：

```
script执行来源
```

---

# 十四、今天实验任务

完成：

## 任务1

DVWA DOM XSS Low：

观察：

URL是否出现：

```
default=
```

---

## 任务2

F12源码搜索：

寻找：

```
location
```

```
document.write
```

---

## 任务3

理解数据流：

画：

```
Source:

location.hash


↓

Sink:

innerHTML
```

---

## 任务4

总结三个XSS区别：

```
反射型：

?

存储型：

数据库

DOM型：

JavaScript
```

---

# 今天完成后，你的XSS知识树：

```
XSS

├── Reflected XSS
│      ├── 参数输入
│      ├── Burp验证
│      └── 过滤绕过
│
├── Stored XSS
│      ├── 数据库存储
│      ├── 影响范围
│      └── Cookie风险
│
└── DOM XSS
       ├── Source
       ├── Sink
       ├── JavaScript审计
       └── SPA框架风险
```

---

下一节进入：

# 🔥 XSS Day 5：XSS综合实战 + 绕过 + 漏洞报告

内容：

1. DVWA High XSS分析
    
2. XSS过滤绕过总结
    
3. Burp自动化检测思路
    
4. XSS漏洞报告编写
    
5. 企业修复验证流程
    

完成这一节后，XSS模块就达到初级Web渗透岗位要求。