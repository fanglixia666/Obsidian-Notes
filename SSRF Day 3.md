# 🔥 SSRF Day 3：综合实战（Pikachu + PortSwigger + SRC复盘）

今天是 SSRF 模块最后一天。

目标：

从：

> “知道 SSRF 是什么”

提升到：

> “看到一个 URL 请求点，知道怎么判断、怎么验证、怎么写报告”。

---

# 一、Pikachu SSRF 两个模块完整复盘

Pikachu 设计两个 SSRF：

1. SSRF(curl)
    
2. SSRF(file_get_contents)
    

核心区别：

|模块|底层函数|特点|
|---|---|---|
|SSRF(curl)|curl_exec|偏HTTP请求|
|SSRF(file_get_contents)|file_get_contents|支持更多资源读取方式|

---

# 二、Pikachu SSRF(curl)复盘

## 1. 页面分析

进入：

```
Pikachu
 ↓
SSRF
 ↓
SSRF(curl)
```

页面通常：

```
请输入URL:

[____________]

提交
```

---

## 2. 正常测试

输入：

```
http://www.baidu.com
```

现象：

页面返回百度内容。

说明：

服务器执行：

```php
curl_exec($url)
```

请求链：

```
浏览器
 |
 |
 v
Pikachu服务器
 |
 |
 v
百度
```

---

# 3. 测试本机访问

输入：

```
http://127.0.0.1
```

注意：

这里最容易理解错误。

你的电脑：

访问：

```
127.0.0.1
```

代表：

你的电脑。

但是：

Pikachu服务器访问：

```
127.0.0.1
```

代表：

服务器自己。

实际：

```
攻击者

↓

Pikachu

↓

Pikachu服务器本机
```

---

如果返回：

- Apache页面
    
- 本地网站
    
- 管理页面
    

说明：

SSRF成立。

---

# 三、Pikachu SSRF(file_get_contents)复盘

进入：

```
SSRF(file_get_contents)
```

---

测试：

```
http://www.baidu.com
```

正常。

说明：

后端类似：

```php
$url=$_GET['url'];

echo file_get_contents($url);
```

---

# 四、file协议理解

重点：

file_get_contents不仅能读：

```
http://
```

还可能支持：

```
file://
```

---

例如：

```
file:///etc/passwd
```

含义：

读取服务器文件：

```
/etc/passwd
```

---

如果成功：

返回类似：

```
root:x:0:0
www-data:x
```

说明：

存在：

```
SSRF
+
本地文件读取
```

---

# 五、Burp抓包分析

真实测试不是靠页面。

核心：

Burp。

---

## 1. 抓请求

例如：

```http
GET /pikachu/vul/ssrf/ssrf_curl.php?url=http://www.baidu.com HTTP/1.1

Host: pikachu
Cookie: PHPSESSID=xxxx
```

重点：

参数：

```
url=
```

---

# 2. 发送Repeater

右键：

```
Send to Repeater
```

---

原：

```
url=http://www.baidu.com
```

修改：

```
url=http://127.0.0.1
```

发送。

---

观察：

## Response

例如：

返回：

```
Apache2 Ubuntu Default Page
```

说明：

服务器访问成功。

---

# 六、如何判断SSRF是否存在？

很多新人会犯一个错误：

> 输入127.0.0.1，没有返回，就认为没有SSRF。

错误。

判断SSR需要多个角度。

---

# 方法1：响应判断

服务器返回：

目标内容。

例如：

输入：

```
http://127.0.0.1/admin
```

返回：

后台页面。

成立。

---

# 方法2：状态变化

例如：

访问：

```
http://不存在地址
```

返回：

```
timeout
```

访问：

```
http://127.0.0.1
```

返回：

```
200
```

说明：

服务器行为变化。

---

# 方法3：时间判断

例如：

端口开放：

响应快。

关闭：

等待超时。

---

# 方法4：外带验证

真实SRC常用。

场景：

服务器请求，但是页面不显示。

例如：

输入：

```
http://你的测试域名
```

收到：

服务器访问记录。

说明：

存在SSR。

---

# 七、URL过滤绕过思路

真实项目：

一般不会直接：

```
url=http://127.0.0.1
```

因为开发会过滤。

---

## 1. 黑名单过滤问题

错误代码：

```php
if(strstr($url,"127.0.0.1"))
{
    die("forbidden");
}
```

问题：

只是检查字符串。

不是检查：

最终访问IP。

---

安全：

应该：

```
URL解析

↓

DNS解析

↓

IP判断

↓

禁止内网
```

---

# 八、常见绕过思想（理解，不背Payload）

## 1. IP解析差异

同一个目标：

可能存在不同表示方式。

原因：

网络库解析方式不同。

---

## 2. DNS解析差异

流程：

```
检查URL

↓

解析IP

↓

请求
```

中间存在时间窗口。

---

## 3. 重定向

例如：

服务器允许：

```
http://safe.com
```

但是：

safe.com跳转：

```
127.0.0.1
```

如果没有二次检查：

可能产生问题。

---

# 九、PortSwigger SSRF Labs学习路线

推荐顺序：

进入：

PortSwigger Web Security Academy

SSRF章节。

---

## Lab 1：

### Basic SSRF against the local server

目标：

理解：

```
服务器访问自己
```

---

学习点：

```
url参数

↓

localhost

↓

内部访问
```

---

## Lab 2：

### Basic SSRF against another back-end system

目标：

理解：

```
内网服务器探测
```

---

思路：

发现：

```
192.168.x.x
```

服务。

---

## Lab 3：

### SSRF with blacklist-based input filter

目标：

理解：

黑名单过滤问题。

---

学习：

为什么：

```
禁止字符串
```

不是安全方案。

---

## Lab 4：

### SSRF with whitelist-based input filter

目标：

理解：

白名单绕过风险。

---

# 十、真实SRC测试流程

假设发现：

接口：

```
GET /api/image?url=
```

---

## Step 1：确认URL功能

测试：

```
http://example.com
```

观察：

服务器是否请求。

---

## Step 2：外带确认

输入：

测试域名。

确认：

收到请求。

---

## Step 3：测试内部资源

测试方向：

```
localhost

127.0.0.1

内网地址
```

---

## Step 4：评估影响

判断：

### 低：

只能访问公网。

### 中：

访问内部系统。

### 高：

访问：

- 管理后台
    
- 云Metadata
    
- 内部服务
    

---

# 十一、SSRF漏洞报告实战模板

## 标题

```
服务器端请求伪造漏洞（SSRF）
导致内部资源访问
```

---

## 漏洞描述

```
系统图片获取接口允许用户控制请求地址，
服务器未限制访问目标范围，
攻击者可以诱导服务器访问内部资源。
```

---

## 请求

```http
GET /api/image?url=http://127.0.0.1
```

---

## 响应

服务器返回：

```
内部服务页面
```

---

## 影响

攻击者可能：

- 探测内网资产
    
- 访问内部管理服务
    
- 获取敏感信息
    
- 扩大攻击范围
    

---

## 修复

### 1.

限制URL来源：

```
白名单
```

### 2.

禁止：

```
127.0.0.1
localhost
私有网段
```

### 3.

解析后校验：

```
最终IP
```

### 4.

网络隔离

限制服务器访问范围。

---

# 十二、SSRF模块最终总结

现在你的知识链：

```
用户输入

↓

URL参数

↓

服务器请求

↓

访问目标

↓

权限扩大
```

---

完成 SSRF 后，你已经完成：

```
OWASP核心漏洞

SQL Injection       ✅
XSS                 ✅
CSRF                ✅
File Upload         ✅
IDOR                ✅
SSRF                ✅
```

---

# 下一阶段进入路线：

## 🔥 Day 37–45 综合Web漏洞训练

不再单独学漏洞。

进入真实渗透流程：

```
信息收集
   ↓
发现入口
   ↓
Burp分析
   ↓
漏洞组合
   ↓
验证影响
   ↓
报告输出
```

下一节：

# 🔥 Web综合实战 Day 37：PortSwigger Web Academy训练方法 + Juice Shop完整渗透流程

这一阶段开始训练“像渗透工程师一样思考”。