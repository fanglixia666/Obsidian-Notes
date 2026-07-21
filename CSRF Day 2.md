# 🔥 CSRF Day 2：Token防护分析 + 绕过思路 + 企业修复

今天进入 CSRF 深入阶段。

上一节你已经完成：

```text
CSRF Low

抓请求
 ↓
发现无Token
 ↓
构造HTML请求
 ↓
浏览器自动携带Cookie
 ↓
业务操作执行
```

今天提升到企业漏洞分析思路：

```text
为什么Low存在漏洞？

↓

开发增加防护

↓

Token如何工作？

↓

Token为什么有时仍然失效？

↓

企业如何修复？
```

---

# 一、DVWA Medium CSRF分析

首先切换：

```text
DVWA

↓

Security

↓

Medium
```

进入：

```text
CSRF
```

---

抓正常修改密码请求。

Burp：

```text
Proxy

↓

HTTP history
```

你会看到类似：

```http
GET /DVWA/vulnerabilities/csrf/?password_new=111&password_conf=111&Change=Change HTTP/1.1

Host: lvh.me

Cookie:
PHPSESSID=xxxx;
security=medium

Referer:
http://lvh.me/DVWA/vulnerabilities/csrf/
```

---

注意：

Medium增加了：

```http
Referer
```

检查。

---

# 二、Referer防护原理

Referer：

表示：

> 当前请求从哪个页面跳转过来。

例如：

正常：

你从：

```
http://lvh.me/DVWA/vulnerabilities/csrf/
```

点击提交。

请求：

```http
Referer:
http://lvh.me/DVWA/vulnerabilities/csrf/
```

---

服务器判断：

```php
if(
strpos($_SERVER['HTTP_REFERER'],
"lvh.me")
)
{
执行修改
}
```

---

如果攻击页面：

```text
evil.com
```

发起：

请求：

```http
Referer:
http://evil.com
```

服务器拒绝。

---

# 三、为什么Referer不是最佳方案？

原因：

## 1. Referer可以为空

例如：

浏览器隐私策略：

可能发送：

```http
Referer:
```

---

## 2. 可以被环境影响

代理：

安全软件：

浏览器插件：

都可能改变。

---

## 3. 只能判断来源

不能证明请求真实性。

---

例如：

攻击者页面：

如果在同域：

```text
lvh.me/evil.html
```

那么：

Referer：

也是：

```text
lvh.me
```

---

所以：

Referer属于：

辅助防御。

---

# 四、进入High CSRF

切换：

```text
Security:

High
```

---

抓请求：

你会发现：

请求变化。

例如：

```http
GET /DVWA/vulnerabilities/csrf/

?password_new=111

&password_conf=111

&Change=Change

&user_token=xxxxxxxx
```

---

出现：

```text
user_token
```

这就是：

CSRF Token。

---

# 五、CSRF Token是什么？

一句话：

> 服务端生成一个随机值，绑定用户Session，请求时必须携带正确Token。

---

流程：

## 第一步：打开页面

服务器：

生成：

```text
user_token:

abc123xyz
```

保存：

Session：

```json
{
 PHPSESSID:
 xxx,

 token:
 abc123xyz
}
```

---

## 第二步：返回页面

HTML：

```html
<input
name="user_token"
value="abc123xyz">
```

---

## 第三步：提交请求

请求：

```http
GET /csrf/

password_new=111

user_token=abc123xyz
```

---

## 第四步：服务器验证

服务器：

比较：

请求Token：

```text
abc123xyz
```

Session：

```text
abc123xyz
```

一致：

执行。

---

# 六、为什么Token能防CSRF？

攻击者的问题：

他可以伪造：

```text
password_new=hacker
```

但是：

不知道：

```text
user_token
```

---

因为：

Token通常：

随机生成：

例如：

```text
f8d92ksl39d0a8sd7f
```

---

攻击页面：

无法知道：

用户Session中的Token。

---

所以：

攻击失败。

---

# 七、Burp分析Token

重点来了。

真实测试不是：

看到Token就结束。

而是分析：

Token实现是否正确。

---

抓请求：

例如：

```http
GET /csrf/

password_new=111

user_token=abc123
```

发送：

```text
Send to Repeater
```

---

测试：

## 情况1：删除Token

修改：

删除：

```text
user_token
```

发送。

观察：

响应。

---

如果：

成功修改密码。

说明：

Token验证失效。

---

## 情况2：修改Token

例如：

```text
user_token=test123
```

发送。

正常：

应该失败。

---

# 八、常见Token设计缺陷

企业真实漏洞经常出现这些。

---

# 缺陷1：Token固定

例如：

所有用户：

```text
token=123456
```

问题：

攻击者可以猜。

---

# 缺陷2：Token没有绑定Session

错误：

数据库：

保存：

```text
token=abc
```

所有用户通用。

---

正确：

应该：

```text
Session A:

token=111


Session B:

token=222
```

---

# 缺陷3：Token泄露

例如：

URL：

```
/change?id=1&token=abc123
```

可能进入：

- 日志
    
- 浏览器历史
    
- Referer
    

---

# 缺陷4：只验证存在

错误：

```php
if(isset($_POST['token']))
{
执行
}
```

---

正确：

应该：

```php
if(
$_POST['token']
==
$_SESSION['token']
)
{
执行
}
```

---

# 九、SameSite Cookie深入

现代Web：

大量使用：

SameSite。

Cookie：

例如：

```http
Set-Cookie:

PHPSESSID=abc;

SameSite=Strict
```

---

## Strict

最严格。

跨站：

```text
evil.com

↓

bank.com
```

Cookie：

不发送。

---

## Lax

目前很多浏览器默认。

允许部分安全GET。

---

## None

允许跨站。

必须：

```text
Secure
```

---

# 十、CSRF企业防御组合

真实企业不会只依赖一个。

推荐：

## 1. CSRF Token

核心。

---

## 2. SameSite Cookie

辅助。

---

## 3. Referer/Origin检查

辅助。

---

## 4. 重要操作二次确认

例如：

修改密码：

要求：

- 原密码
    
- 短信
    
- MFA
    

---

# 十一、CSRF漏洞报告模板

下面按照企业报告格式。

---

## 漏洞名称

```
Cross-Site Request Forgery (CSRF)
跨站请求伪造
```

---

## 风险等级

```
Medium / High
```

根据业务影响。

---

## 漏洞位置

例如：

```
URL:

/user/changePassword


参数:

password_new


请求方式:

GET
```

---

## 漏洞描述

> 应用程序敏感操作接口未进行有效CSRF防护，攻击者可诱导已登录用户访问恶意页面，使浏览器自动携带身份认证信息向服务器提交请求，从而在用户不知情情况下完成密码修改等敏感操作。

---

## 复现步骤

用户登录系统。

访问攻击页面：

```html
<form>
...
</form>
```

请求自动发送。

业务操作成功。

---

## 修复建议

### 推荐：

增加CSRF Token。

示例：

请求：

```
password=123

csrf_token=random
```

服务端：

校验：

```
token == session token
```

---

辅助：

- SameSite Cookie
    
- Origin校验
    
- Referer校验
    
- 敏感操作二次确认
    

---

# 十二、CSRF模块完成情况

你现在掌握：

```text
CSRF

基础:
├── Cookie机制
├── Session认证
├── 自动携带Cookie

攻击:
├── GET CSRF
├── POST CSRF
├── Burp验证

防御:
├── Token
├── SameSite
├── Referer
├── Origin

企业:
├── 漏洞报告
├── 修复验证
```

---

# 下一阶段进入 OWASP Top 10 第四个核心漏洞：

# 🔥 文件上传漏洞（Unrestricted File Upload）

学习路线：

```text
文件上传原理

↓

DVWA File Upload Low

↓

MIME绕过

↓

后缀绕过

↓

Content-Type绕过

↓

黑名单绕过

↓

WebShell风险理解

↓

文件上传安全修复

↓

企业漏洞报告
```

这个漏洞会正式进入你前面规划里的：

> 信息收集 → Web打点 → Shell → 内网横向

中的 **Web拿权限入口**。