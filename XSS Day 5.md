# 🔥 XSS Day 5：XSS综合实战 + 绕过 + 漏洞报告

今天是 XSS 模块收尾。

前面4天你已经掌握：

```text
Day1:
反射型XSS
输入 → 输出 → 浏览器执行

Day2:
过滤机制分析
为什么黑名单无法防御

Day3:
存储型XSS
数据库 → 页面 → 用户浏览器

Day4:
DOM XSS
JavaScript Source → Sink
```

今天站在**企业渗透测试工程师**角度：

目标：

```text
发现漏洞
 ↓
分析过滤
 ↓
验证影响
 ↓
编写报告
 ↓
指导修复
 ↓
复测确认
```

---

# 一、DVWA High XSS分析

进入：

```
DVWA

↓

Security Level

↓

High

↓

XSS Reflected
```

---

测试：

输入：

```html
<script>alert(1)</script>
```

观察。

结果：

大概率失败。

---

为什么？

查看源码：

路径：

```
DVWA/vulnerabilities/xss_r/source/high.php
```

类似：

```php
<?php

$name = $_GET['name'];

$name = preg_replace(
'/<script.*?>.*?<\/script>/i',
'',
$name
);

echo "Hello ".$name;

?>
```

---

分析：

这里使用：

```php
preg_replace()
```

正则过滤。

---

过滤规则：

匹配：

```html
<script>
...
</script>
```

然后删除。

---

# 二、为什么High依然可能存在问题？

很多开发人员认为：

> 正则过滤了script，所以安全。

但是：

XSS不是只有script。

---

浏览器支持：

大量执行入口。

例如：

```html
<img onerror="">
```

---

所以：

安全问题：

不是：

"过滤多少字符串"

而是：

"输出位置是否安全"

---

# 三、High级别绕过思路分析

注意：

这里学习的是漏洞原理分析。

真实项目测试必须获得授权。

---

## 思路1：寻找不同执行上下文

过滤：

```html
<script>
```

但是：

浏览器执行JavaScript的位置：

很多。

例如：

HTML事件：

```html
onload
onerror
onclick
```

---

例如：

HTML：

```html
<img src="不存在资源"
     onerror="执行">
```

浏览器行为：

```
加载图片

↓

失败

↓

触发onerror

↓

执行JS
```

---

## 思路2：大小写

部分过滤：

```text
<script>
```

但是：

HTML解析：

大小写不敏感。

例如：

```
<ScRiPt>
```

浏览器：

仍然认为：

script标签。

---

## 思路3：编码问题

不同组件处理方式不同：

例如：

```
输入层

↓

Web服务器

↓

PHP

↓

浏览器
```

每一层：

可能解析一次。

安全问题：

经常来自：

```
A认为过滤了

B又重新解析
```

---

# 四、XSS过滤绕过总结（面试重点）

整理成知识表：

|防御方式|问题|
|---|---|
|删除script|存在其他标签|
|大小写过滤|HTML解析不敏感|
|黑名单|无法覆盖全部情况|
|关键词过滤|编码绕过|
|输入过滤|无法判断上下文|

---

企业正确方式：

不是：

```
禁止危险字符
```

而是：

```
根据输出环境编码
```

---

# 五、XSS测试方法论

真实渗透不会一上来：

输入payload。

流程：

---

# Step 1：寻找输入点

例如：

页面：

```
搜索框

留言

昵称

评论

个人资料
```

记录：

```
参数:
name

位置:
GET
```

---

# Step 2：确认是否回显

输入：

```
test123
```

观察：

响应：

```html
Hello test123
```

说明：

数据进入页面。

---

# Step 3：判断上下文

例如：

## HTML正文

```html
<div>
test
</div>
```

风险：

HTML标签。

---

## 属性

```html
<input value="test">
```

风险：

属性闭合。

---

## JavaScript

```javascript
var a="test"
```

风险：

JS上下文。

---

# Step 4：验证执行

使用：

测试字符：

```
<script>alert(1)</script>
```

或者：

其他执行方式。

---

# 六、Burp自动化检测思路

企业测试：

不会人工测试所有参数。

需要自动化。

---

Burp核心模块：

## 1. Proxy

作用：

抓请求。

例如：

```
GET

/search?q=test
```

---

## 2. Repeater

作用：

手工验证。

修改：

```
q=test
```

观察：

响应。

---

## 3. Intruder

作用：

批量测试。

例如：

参数：

```
q=
```

批量替换：

测试字符串。

---

流程：

```
抓请求

↓

发送Intruder

↓

设置payload位置

↓

加载测试字典

↓

观察响应差异

```

---

# 七、XSS自动化检测逻辑

扫描器基本逻辑：

---

## 第一步：

插入标记：

例如：

```
xss_test_123
```

观察：

是否回显。

---

## 第二步：

判断位置：

返回：

```html
<div>
xss_test_123
</div>
```

属于：

HTML。

---

## 第三步：

尝试上下文突破。

例如：

如果：

```html
<input value="xxx">
```

尝试：

闭合：

```
"
```

---

## 第四步：

验证执行。

---

工具：

企业常用：

- Burp Scanner
    
- Nessus Web插件
    
- AWVS
    
- AppScan
    
- 自研扫描平台
    

---

# 八、XSS漏洞报告编写

现在模拟企业报告。

---

# 漏洞名称

```
Stored Cross-Site Scripting
存储型跨站脚本漏洞
```

---

# 漏洞等级

```
High
```

---

# 漏洞位置

例如：

```
URL:

/comment/save


参数:

message


请求方式:

POST

```

---

# 漏洞描述

示例：

> 应用程序未对用户提交内容进行有效安全处理，导致攻击者可以提交恶意HTML/JavaScript代码。当其他用户访问包含恶意内容的页面时，浏览器会执行攻击者注入的脚本，可能造成用户信息泄露、页面篡改以及用户身份被冒用等风险。

---

# 复现步骤

## 1.

提交：

```
message=test
```

---

## 2.

修改：

输入：

```
<script>alert(1)</script>
```

---

## 3.

访问页面：

出现：

```
JavaScript执行
```

---

# 影响分析

可能导致：

```
1. 用户Cookie泄露

2. 钓鱼页面

3. 用户操作劫持

4. 管理员账号风险

5. 业务数据泄露
```

---

# 修复建议

## 方案1（推荐）

输出编码：

PHP：

```php
htmlspecialchars(
$data,
ENT_QUOTES,
'UTF-8'
);
```

---

## 方案2

输入校验：

限制：

```
HTML标签
JavaScript关键字
```

---

## 方案3

Cookie安全：

开启：

```
HttpOnly

Secure

SameSite
```

---

## 方案4

CSP：

增加：

```
Content-Security-Policy
```

---

# 九、企业修复验证流程

安全人员不能只提交报告。

需要复测。

---

流程：

```
开发修复

↓

安全人员重新测试

↓

确认payload失效

↓

确认业务正常

↓

关闭漏洞
```

---

例如：

修复前：

输入：

```
<script>alert(1)</script>
```

结果：

执行。

---

修复后：

返回：

```html
&lt;script&gt;
alert(1)
&lt;/script&gt;
```

浏览器：

显示文本。

漏洞关闭。

---

# 十、XSS模块最终知识地图

你现在应该形成：

```
                 XSS

                  |

 ------------------------------------------------

 |                    |                         |

反射型              存储型                  DOM型

 |                    |                         |

参数输入             数据库                  JS代码

 |                    |                         |

Burp验证             影响用户                Source/Sink


 ------------------------------------------------

                  |

              防御

                  |

       输出编码(htmlspecialchars)

                  |

          输入验证

                  |

          CSP

                  |

     Cookie安全(HttpOnly)
```

---

# 十一、你的Web渗透学习进度

目前完成：

✅ HTTP基础  
✅ Cookie / Session  
✅ SQL Injection  
✅ SQLMap  
✅ XSS完整模块

下一阶段进入：

# 🔥 OWASP Top 10 第三个核心漏洞：CSRF（跨站请求伪造）

学习路线：

```
CSRF原理

↓

DVWA CSRF Low

↓

Token防护

↓

Token绕过分析

↓

SameSite Cookie

↓

Burp验证

↓

企业修复方案

↓

漏洞报告
```

CSRF会把你前面学习的：

- Cookie
    
- Session
    
- HTTP请求
    
- Burp抓包
    

全部串起来。你现在进入非常关键的Web安全阶段。