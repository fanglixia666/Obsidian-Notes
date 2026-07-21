# 🔥 OWASP Top 10 第二个核心漏洞：XSS（Cross-Site Scripting）完整实战

我们开始进入 **XSS模块**。

这一章的学习目标：

你不仅要知道：

> `<script>alert(1)</script>` 是XSS

更要理解：

> 为什么浏览器会执行它？  
> 输入经过哪里？  
> 输出到哪里？  
> 为什么过滤失败？  
> 企业如何修复？

---

# Day 1：XSS基础 + DVWA反射型XSS实战

目标：

完成：

```
XSS原理理解
        ↓
浏览器执行机制
        ↓
DVWA Reflected XSS
        ↓
Burp抓包分析
        ↓
漏洞验证
```

---

# 一、什么是XSS？

XSS：

全称：

```
Cross-Site Scripting
跨站脚本攻击
```

本质：

> 攻击者控制网页输出内容，让浏览器执行恶意JavaScript。

注意：

攻击发生的位置：

不是服务器。

而是：

```
用户浏览器
```

---

# 二、SQL注入和XSS区别

你刚学完SQL Injection。

对比：

|漏洞|攻击目标|
|---|---|
|SQL Injection|数据库|
|XSS|浏览器|

---

SQL注入：

攻击链：

```
用户输入
 ↓
SQL语句
 ↓
数据库
```

---

XSS：

攻击链：

```
用户输入
 ↓
HTML页面
 ↓
浏览器解析
 ↓
JavaScript执行
```

---

# 三、浏览器为什么会执行？

浏览器看到：

```html
<script>
alert(1)
</script>
```

解析：

```
HTML解析器

发现script标签

↓

JavaScript引擎执行

↓

alert()
```

---

例如：

正常网页：

```html
<h1>
hello
</h1>
```

浏览器：

显示：

```
hello
```

---

如果：

```html
<h1>
<script>alert(1)</script>
</h1>
```

浏览器：

执行：

```
alert(1)
```

---

# 四、XSS产生条件（重点）

一个XSS漏洞需要三个条件：

---

## 条件1：用户输入可控

例如：

输入框：

```
name
comment
search
```

---

## 条件2：输入进入HTML

例如：

PHP：

```php
echo $_GET['name'];
```

---

## 条件3：没有正确编码

例如：

危险：

```php
echo $name;
```

安全：

```php
htmlspecialchars($name);
```

---

总结：

```
输入
 |
 ↓
服务器
 |
 ↓
HTML响应
 |
 ↓
浏览器执行
```

---

# 五、XSS分类

OWASP常见分类：

---

# 1. 反射型XSS（Reflected XSS）

特点：

```
一次性攻击
```

数据流程：

```
攻击链接
    |
    ↓
服务器
    |
    ↓
HTML响应
    |
    ↓
浏览器执行
```

例如：

URL:

```
search=test
```

服务器：

```php
echo $_GET['search'];
```

---

# 2. 存储型XSS（Stored XSS）

特点：

```
持久化
```

例如：

留言板：

攻击者提交：

```
<script>alert(1)</script>
```

保存数据库。

之后：

任何用户访问：

```
留言页面
```

都会执行。

流程：

```
输入
 ↓
数据库
 ↓
页面展示
 ↓
浏览器执行
```

---

# 3. DOM XSS

特点：

漏洞发生在：

```
JavaScript代码
```

例如：

```javascript
location.hash
```

直接写入：

```javascript
innerHTML
```

---

# 六、开始DVWA实验

环境：

```
DVWA
 ↓
XSS (Reflected)
```

---

先设置：

```
DVWA Security
 ↓
Low
```

原因：

先理解漏洞原理。

---

# 七、DVWA Reflected XSS

进入：

```
DVWA
 ↓
XSS (Reflected)
```

页面：

通常有：

```
What's your name?
[        ]

Submit
```

---

输入：

```
test
```

观察：

页面显示：

```
Hello test
```

---

说明：

你的输入：

进入页面。

---

# 八、第一次验证XSS

输入：

```html
<script>alert(1)</script>
```

点击：

Submit

如果弹窗：

```
1
```

说明：

XSS成功。

---

# 九、为什么成功？

后台类似：

```php
$name=$_GET['name'];

echo "Hello ".$name;
```

---

你的输入：

```html
<script>alert(1)</script>
```

进入：

```html
Hello <script>alert(1)</script>
```

---

浏览器：

解析HTML：

发现script：

执行。

---

# 十、Burp抓包分析

打开：

```
Burp
 ↓
Proxy
 ↓
HTTP history
```

找到：

请求：

类似：

```http
GET /DVWA/vulnerabilities/xss_r/?name=test
```

---

修改：

```
name=<script>alert(1)</script>
```

发送。

---

观察响应：

搜索：

```
script
```

如果返回：

```html
Hello <script>alert(1)</script>
```

说明：

服务器直接输出。

---

# 十一、今天任务

完成4个任务：

---

## 任务1：理解XSS数据流

画：

```
输入
 ↓
服务器
 ↓
HTML
 ↓
浏览器
 ↓
JS执行
```

---

## 任务2：DVWA Reflected XSS Low

输入：

```
<script>alert(1)</script>
```

确认弹窗。

---

## 任务3：Burp抓包

找到：

```
name参数
```

修改：

```
name=<script>alert(1)</script>
```

观察响应。

---

## 任务4：源码分析

打开：

```
DVWA/vulnerabilities/xss_r/source/low.php
```

寻找：

```php
echo
$_GET
```

---

# 十二、今天必须掌握

完成后你应该理解：

✅ XSS攻击目标是浏览器  
✅ XSS不是攻击数据库  
✅ 反射型XSS的数据流  
✅ 为什么script标签会执行  
✅ Burp如何验证XSS  
✅ 输入 → 输出 → 执行 的漏洞链

---

下一节：

# 🔥 XSS Day 2：反射型XSS绕过与过滤分析

内容：

1. DVWA Medium XSS
    
2. 为什么 `<script>` 被过滤
    
3. 大小写绕过
    
4. 标签绕过
    
5. 事件型XSS
    
6. XSS Payload设计思路
    
7. 企业修复（HTML实体编码）
    

这部分会和你前面的SQL注入一样，进入“为什么能绕过”的阶段。