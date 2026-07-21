# 🔥 XSS Day 3：存储型 XSS + Cookie安全 + 攻击链分析

这一节进入 XSS 最重要的部分。

前两节你学习：

```
Day 1：
反射型XSS原理

输入
 ↓
服务器
 ↓
响应页面
 ↓
浏览器执行


Day 2：
过滤绕过

为什么<script>过滤无效
如何分析过滤逻辑
```

今天进入：

> **为什么企业更重视存储型XSS？**

因为它不是“一次性攻击”，而可能影响大量用户。

---

# 一、什么是存储型 XSS（Stored XSS）

存储型XSS：

也叫：

```
Persistent XSS
持久化XSS
```

核心：

> 攻击代码被服务器保存，之后访问页面的用户都会触发。

---

## 反射型 XSS 回顾

流程：

```
攻击链接
    |
    ↓
用户点击
    |
    ↓
服务器返回
    |
    ↓
执行JS
```

特点：

需要诱导用户访问恶意链接。

---

## 存储型 XSS

流程：

```
攻击者提交恶意内容

        ↓

服务器保存

        ↓

数据库

        ↓

其他用户访问页面

        ↓

浏览器执行
```

---

区别：

|类型|是否存储|影响范围|
|---|---|---|
|反射型|否|单个用户|
|存储型|是|所有访问用户|

---

# 二、DVWA Stored XSS 实战

进入：

```
DVWA

↓

XSS (Stored)
```

设置：

```
Security Level: Low
```

---

页面通常：

```
Message:

Name:
[       ]

Message:
[       ]

Sign Guestbook
```

---

正常输入：

Name:

```
test
```

Message:

```
hello
```

提交。

页面显示：

```
test says:

hello
```

---

说明：

你的输入：

被保存。

---

# 三、第一次验证存储型XSS

输入：

Name:

```
test
```

Message:

```html
<script>alert(1)</script>
```

提交。

如果弹窗：

说明：

漏洞存在。

---

但是注意：

存储型XSS的关键不是弹窗。

重点：

看数据流。

---

# 四、分析漏洞数据流

打开源码：

路径：

```
DVWA/vulnerabilities/xss_s/source/
```

找到：

```
low.php
```

类似：

```php
$message = $_POST['txtMessage'];
$name = $_POST['txtName'];


$sql="
INSERT INTO guestbook
VALUES('$name','$message')
";
```

---

这里：

第一阶段：

输入：

```
txtMessage
```

↓

数据库

---

第二阶段：

页面读取：

例如：

```php
SELECT message FROM guestbook
```

↓

输出：

```php
echo $message;
```

---

完整链：

```
用户输入

↓

数据库

↓

页面输出

↓

浏览器解析

↓

JavaScript执行
```

---

# 五、为什么存储型XSS危害更高？

假设：

一个企业：

```
论坛
```

评论功能。

攻击者提交：

```
恶意JS
```

保存。

---

之后：

10000个用户打开帖子：

```
用户A
 ↓
执行

用户B
 ↓
执行

用户C
 ↓
执行
```

---

影响：

可能包括：

- 用户身份信息泄露
    
- 页面篡改
    
- 钓鱼
    
- 权限操作
    
- 业务逻辑攻击
    

---

# 六、XSS攻击链（重点）

真实攻击不是：

```
alert(1)
```

而是：

```
发现输入点

↓

构造JavaScript

↓

影响用户浏览器

↓

执行敏感操作
```

---

例如：

攻击链：

```
存储型XSS

↓

管理员访问后台

↓

浏览器执行JS

↓

执行管理员权限操作

↓

造成业务影响
```

---

# 七、Cookie安全问题

你之前学习Session：

知道：

```
Cookie:

PHPSESSID=xxxx
```

如果Cookie可以被JavaScript读取：

例如：

```javascript
document.cookie
```

可能获取：

```
PHPSESSID=xxxx
```

---

然后攻击者可能：

利用用户身份。

---

例如：

用户：

```
admin
```

Cookie：

```
PHPSESSID=abc123
```

攻击者：

得到：

```
PHPSESSID=abc123
```

可能：

冒充身份。

---

# 八、为什么HttpOnly可以防？

Cookie属性：

例如：

普通Cookie：

```
PHPSESSID=abc123
```

JS：

```javascript
document.cookie
```

可能读取。

---

增加：

```
HttpOnly
```

变：

```
PHPSESSID=abc123;
HttpOnly
```

效果：

浏览器：

允许：

```
HTTP请求自动携带
```

禁止：

```
JavaScript读取
```

---

例如：

XSS执行：

```javascript
alert(document.cookie)
```

结果：

以前：

```
PHPSESSID=abc123
```

现在：

```
空
```

---

# 九、HttpOnly是不是万能？

不是。

重点：

HttpOnly只能保护：

```
Cookie读取
```

不能防止：

XSS执行。

---

例如：

XSS仍然可以：

```
修改页面内容
```

```
钓鱼
```

```
发送请求
```

---

所以：

核心防护：

还是：

```
输出编码
```

---

# 十、BeEF是什么？

你提到：

BeEF。

全称：

```
Browser Exploitation Framework
```

浏览器攻击框架。

它用于：

研究浏览器安全。

---

核心思想：

如果：

用户浏览器执行：

```javascript
hook.js
```

那么：

浏览器成为：

```
Zombie Browser
```

---

攻击者可以研究：

- 浏览器信息
    
- 环境信息
    
- DOM操作
    
- 浏览器安全问题
    

---

学习阶段：

你需要理解：

```
XSS
    |
    ↓
JavaScript执行
    |
    ↓
浏览器控制
```

---

但是企业防护重点：

不是BeEF。

而是：

```
为什么JS能执行？
```

---

# 十一、Burp分析Stored XSS

打开：

```
Burp

Proxy

HTTP history
```

找到：

```
POST /DVWA/vulnerabilities/xss_s/
```

类似：

```http
POST /DVWA/vulnerabilities/xss_s/

txtName=test

mtxMessage=hello
```

---

修改：

```
mtxMessage=<script>alert(1)</script>
```

发送。

观察：

响应。

---

重点：

不是看请求。

而是：

之后刷新页面：

```
GET guestbook页面
```

是否仍然存在：

```html
<script>alert(1)</script>
```

---

这就是：

存储。

---

# 十二、企业真实修复方案

## 1. 输出编码（第一原则）

错误：

```php
echo $comment;
```

正确：

```php
echo htmlspecialchars(
$comment,
ENT_QUOTES,
'UTF-8'
);
```

---

## 2. 输入验证

例如：

用户名：

允许：

```
[a-zA-Z0-9]
```

拒绝：

```
<script>
```

---

## 3. Cookie安全

设置：

```
HttpOnly
Secure
SameSite
```

例如：

```
Set-Cookie:

PHPSESSID=xxx;

HttpOnly;

Secure;

SameSite=Strict
```

---

## 4. CSP

限制脚本来源：

例如：

```
Content-Security-Policy:
default-src 'self'
```

---

# 十三、今天练习任务

完成：

## 任务1

DVWA Stored XSS Low：

提交：

```
<script>alert(1)</script>
```

确认保存。

---

## 任务2

观察：

刷新页面后：

是否仍然执行。

理解：

```
数据库
 ↓
页面
 ↓
浏览器
```

---

## 任务3

查看源码：

找到：

```
INSERT
SELECT
echo
```

三个位置。

---

## 任务4

回答：

为什么：

```
HttpOnly
```

不能完全解决XSS？

---

# 今天知识闭环

你应该掌握：

```
XSS

反射型:
输入
 ↓
响应
 ↓
执行


存储型:
输入
 ↓
数据库
 ↓
页面
 ↓
执行


防护:
输出编码
+
Cookie安全
+
CSP
```

---

下一节进入：

# 🔥 XSS Day 4：DOM XSS + 前端代码审计

内容：

1. DOM XSS原理
    
2. JavaScript危险函数
    
3. innerHTML漏洞分析
    
4. location.hash利用
    
5. 前端代码审计方法
    
6. 企业SPA/Vue/React中的XSS风险
    

这一节会开始接近真实企业Web安全测试。