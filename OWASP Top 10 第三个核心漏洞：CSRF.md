# 🔥 OWASP Top 10 第三个核心漏洞：CSRF（Cross-Site Request Forgery）

今天进入 **CSRF完整实战模块 Day 1：原理 + DVWA验证**。

这一章会把你之前学的：

```text
HTTP请求
    +
Cookie
    +
Session
    +
Burp抓包
    +
Web业务逻辑
```

全部串起来。

---

# 一、什么是CSRF？

CSRF：

全称：

```text
Cross-Site Request Forgery

跨站请求伪造
```

核心一句话：

> **攻击者诱导已经登录的用户，在不知情的情况下发送一个合法请求。**

注意：

CSRF不是窃取Cookie。

而是：

**利用浏览器自动携带Cookie的特性。**

---

# 二、CSRF和XSS区别（重点）

很多初学者容易混淆。

||XSS|CSRF|
|---|---|---|
|攻击目标|浏览器执行JS|服务器业务操作|
|核心问题|输入未过滤|请求没有验证|
|是否需要JS|通常需要|不一定|
|是否需要用户登录|不一定|通常需要|
|利用|执行代码|冒用身份请求|

---

举例：

## XSS：

攻击者：

```html
<script>
alert(document.cookie)
</script>
```

目标：

浏览器。

---

## CSRF：

攻击者：

诱导：

```text
修改密码请求
```

目标：

服务器业务。

---

# 三、为什么Cookie导致CSRF？

回顾Cookie。

登录：

服务器：

返回：

```http
Set-Cookie:

PHPSESSID=abc123
```

浏览器保存。

以后访问：

```http
GET /profile
```

自动：

```http
Cookie:

PHPSESSID=abc123
```

---

重点：

浏览器不会判断：

> 这个请求是不是用户主动发起。

只要：

域名匹配。

Cookie自动发送。

---

# 四、CSRF攻击流程

假设：

银行网站：

```text
bank.com
```

用户已经登录。

Cookie：

```text
PHPSESSID=xxxx
```

---

正常修改密码：

请求：

```http
POST /change_password

password=123456
```

服务器：

检查：

Cookie有效。

修改密码。

---

攻击者：

创建：

```html
<form action="https://bank.com/change_password"
method="POST">

<input name="password"
value="hacker">

</form>

<script>
document.forms[0].submit();
</script>
```

---

用户访问：

```text
evil.com
```

浏览器：

自动发送：

```http
POST bank.com/change_password

Cookie:
PHPSESSID=xxxx


password=hacker
```

---

银行服务器认为：

这是用户本人请求。

密码修改成功。

---

# 五、CSRF成立需要三个条件

非常重要。

---

## 条件1：存在敏感操作

例如：

修改：

- 密码
    
- 邮箱
    
- 手机号
    
- 权限
    
- 转账
    

普通：

搜索。

没有意义。

---

## 条件2：请求依赖Cookie认证

例如：

```http
Cookie:

PHPSESSID=xxx
```

服务器：

靠Cookie识别用户。

---

## 条件3：没有额外验证

例如：

没有：

```text
CSRF Token
```

---

三个条件：

```text
敏感操作

+

Cookie认证

+

无请求校验

=

CSRF
```

---

# 六、进入DVWA CSRF实验

环境：

打开：

```
DVWA
```

设置：

```
Security Level:

Low
```

---

进入：

```
DVWA

↓

CSRF
```

---

页面：

通常：

```
Change your password

New password:
[        ]

Confirm password:
[        ]

Change
```

---

# 七、正常修改密码抓包

打开：

Burp：

```
Proxy

↓

Intercept On
```

---

输入：

```
123456
```

提交。

---

Burp捕获：

类似：

```http
POST /DVWA/vulnerabilities/csrf/

HTTP/1.1

Host: lvh.me

Cookie:
PHPSESSID=xxxx;
security=low


password_new=123456

password_conf=123456

Change=Change
```

---

重点看：

请求：

```text
有没有随机Token？
```

现在：

没有。

---

# 八、分析为什么存在漏洞

请求：

```http
POST /csrf/
```

参数：

```text
password_new

password_conf
```

服务器判断：

类似：

```php
if(
password_new == password_conf
){

修改密码

}
```

---

但是：

没有：

```text
这个请求是不是用户真实页面发出的？
```

---

攻击者可以构造：

```html
<form>
password_new=hacker

password_conf=hacker
</form>
```

---

# 九、模拟CSRF攻击页面

创建：

csrf.html

内容：

```html
<html>

<body>

<form 
action="http://lvh.me/DVWA/vulnerabilities/csrf/"
method="POST">


<input 
name="password_new"
value="hacker123">


<input
name="password_conf"
value="hacker123">


<input
type="submit"
value="submit">

</form>


</body>

</html>
```

---

运行：

用户登录DVWA状态下。

打开：

```
csrf.html
```

点击提交。

---

观察：

密码是否改变。

---

# 十、Burp如何验证CSRF？

Burp核心：

## 1. 抓正常请求

例如：

```http
POST /csrf/
```

---

## 2. 右键

```
Engagement tools

↓

Generate CSRF PoC
```

---

Burp自动生成：

HTML：

```html
<form>

<input>

</form>
```

---

这就是：

企业测试CSRF常用方法。

---

# 十一、为什么CSRF Token可以防？

修复前：

请求：

```http
POST /change_password


password=123
```

服务器：

不知道：

请求来源。

---

修复后：

页面：

生成：

```html
<input 
name="csrf_token"
value="a8sd92jd">
```

---

请求：

```http
POST /change_password


password=123

csrf_token=a8sd92jd
```

---

服务器：

检查：

```text
token是否正确
```

---

攻击者：

不知道token。

所以：

请求失败。

---

# 十二、CSRF Token原理

流程：

登录：

↓

服务器生成：

```text
token=random
```

↓

绑定：

```text
用户Session
```

↓

表单携带：

```text
token
```

↓

提交：

服务器验证：

```text
token == session中的token
```

---

关系：

```text
Cookie

识别用户


CSRF Token

验证请求真实性
```

---

# 十三、SameSite Cookie防护

现代浏览器：

Cookie增加：

```http
SameSite
```

---

例如：

```http
Set-Cookie:

PHPSESSID=abc;

SameSite=Strict
```

---

作用：

限制：

跨站请求携带Cookie。

---

三个模式：

## Strict

最严格：

跨站不发送。

---

## Lax

默认：

部分GET允许。

---

## None

允许跨站：

需要：

```text
Secure
```

---

# 十四、今天任务

完成：

## 任务1

DVWA CSRF Low：

抓修改密码请求。

记录：

```text
URL

POST参数

Cookie
```

---

## 任务2

回答：

为什么攻击者不知道Cookie也能CSRF？

---

## 任务3

使用Burp：

生成：

```
CSRF PoC
```

---

## 任务4

画流程：

```text
用户登录

↓

Cookie保存

↓

访问恶意页面

↓

浏览器自动带Cookie

↓

服务器执行敏感操作
```

---

# 今天必须掌握

完成后你应该理解：

✅ CSRF不是盗Cookie  
✅ CSRF利用浏览器自动携带Cookie  
✅ CSRF攻击的是业务逻辑  
✅ Token为什么有效  
✅ SameSite作用  
✅ Burp如何生成CSRF PoC

---

下一节：

# 🔥 CSRF Day 2：Token防护分析 + 绕过思路 + 企业修复

内容：

1. DVWA Medium/High CSRF分析
    
2. CSRF Token生成机制
    
3. Token验证缺陷
    
4. Token绕过案例分析
    
5. Referer验证
    
6. SameSite深入
    
7. 企业CSRF漏洞报告编写
    

这一节会把你的 **Cookie / Session / HTTP请求分析能力**进一步提升。