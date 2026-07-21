# 🔥 XSS Day 2：反射型 XSS 绕过与过滤分析

这一节进入真正的漏洞分析阶段。

上一节你完成了：

```
输入
 ↓
服务器
 ↓
HTML响应
 ↓
浏览器解析
 ↓
JavaScript执行
```

今天重点：

> **为什么简单过滤不能阻止XSS？**

这也是企业真实漏洞中最常见的问题。

---

# 一、切换 DVWA Medium XSS

进入：

```
DVWA
 ↓
DVWA Security
 ↓
Medium
```

然后：

```
XSS (Reflected)
```

---

输入：

```html
<script>alert(1)</script>
```

提交。

观察。

---

你可能发现：

没有弹窗。

页面可能显示：

```
Hello
alert(1)
```

或者：

```
<script>alert(1)</script>
```

被破坏。

---

说明：

开发人员增加了过滤。

---

# 二、查看Medium源码

路径：

```
DVWA

/vulnerabilities/xss_r/source/medium.php
```

类似：

```php
$name = $_GET['name'];

$name = str_replace(
    '<script>',
    '',
    $name
);

echo "Hello ".$name;
```

---

重点：

这里做了什么？

简单字符串替换：

```
<script>
      |
      ↓
     删除
```

---

例如：

输入：

```html
<script>alert(1)</script>
```

处理后：

变：

```html
alert(1)</script>
```

所以失败。

---

# 三、为什么这种过滤不安全？

因为：

开发人员错误理解：

> 禁止标签 = 禁止JavaScript

实际上：

JavaScript执行方式很多。

---

浏览器支持：

## 方式1：

script标签

```html
<script>
alert(1)
</script>
```

---

## 方式2：

事件属性

```html
<img src=x onerror=alert(1)>
```

---

## 方式3：

其他HTML标签

```html
<body onload=alert(1)>
```

---

所以：

过滤一个字符串：

不等于：

禁止脚本执行。

---

# 四、绕过思路1：大小写绕过

假设过滤：

```php
str_replace(
"<script>",
"",
)
```

注意：

PHP默认：

区分大小写。

---

过滤：

```
<script>
```

但是：

输入：

```html
<ScRiPt>alert(1)</ScRiPt>
```

结果：

过滤器：

找不到：

```
<script>
```

---

浏览器：

HTML解析：

认为：

```
script
```

标签。

执行。

---

## 为什么？

HTML标签：

大小写不敏感。

例如：

浏览器认为：

```html
<DIV>
```

等于：

```html
<div>
```

---

但是：

PHP字符串比较：

敏感。

---

# 五、绕过思路2：标签替代

如果：

```

观察响应。


---

搜索：

```

script

````

看：

服务器返回：

有没有：

```html
<script>
````

---

然后测试：

大小写：

```
<ScRiPt>alert(1)</ScRiPt>
```

观察。

---

再测试：

事件：

```
<img src=x onerror=alert(1)>
```

---

你的目标不是：

“找到一个payload”。

而是：

确认：

```
过滤规则是什么
```

---

# 八、XSS Payload设计思路

不要背payload。

企业测试流程：

---

## 第一步：确认输出位置

输入：

```
test123
```

观察：

返回：

```html
Hello test123
```

说明：

HTML正文。

---

## 第二步：判断上下文

不同位置：

---

### HTML标签里面

例如：

```html
<div>
输入
</div>
```

关注：

标签。

---

### 属性里面

例如：

```html
<input value="输入">
```

关注：

属性闭合。

---

### JavaScript里面

例如：

```javascript
var name="输入";
```

关注：

JS上下文。

---

# 九、为什么HTML实体编码能修复？

这是重点。

危险：

```php
echo $name;
```

输入：

```html
<script>alert(1)</script>
```

输出：

```html
<script>alert(1)</script>
```

浏览器执行。

---

安全：

```php
echo htmlspecialchars($name);
```

输出：

```html
&lt;script&gt;
alert(1)
&lt;/script&gt;
```

---

浏览器看到：

不是标签。

而是文本：

```
<script>alert(1)</script>
```

---

# 十、htmlspecialchars转换什么？

例如：

输入：

```
<
```

转换：

```
&lt;
```

---

输入：

```
>
```

转换：

```
&gt;
```

---

输入：

```
"
```

转换：

```
&quot;
```

---

输入：

```
'
```

转换：

```
&#039;
```

---

本质：

让浏览器：

```
代码
↓
普通文本
```

---

# 十一、企业XSS防护原则

不要：

❌ 黑名单过滤

例如：

```
删除<script>
```

---

推荐：

## 1. 输出编码（核心）

HTML：

```php
htmlspecialchars()
```

---

## 2. 输入验证

例如：

用户名：

允许：

```
字母
数字
下划线
```

拒绝：

```
HTML标签
```

---

## 3. CSP策略

Content Security Policy

限制：

```
script来源
```

例如：

禁止：

```
inline script
```

---

## 4. HttpOnly Cookie

防止：

XSS直接读取：

```
document.cookie
```

---

# 十二、今天实战任务

完成：

## 任务1

DVWA Medium：

测试：

```html
<script>alert(1)</script>
```

记录结果。

---

## 任务2

查看源码：

找到：

```php
str_replace()
```

或者过滤函数。

---

## 任务3

Burp测试：

观察：

```
输入
 ↓
响应HTML
```

---

## 任务4

理解：

为什么：

```html
<img src=x onerror=alert(1)>
```

可以执行。

---

# 今日知识闭环

你应该掌握：

```
XSS

原理:
输入 → 输出 → 浏览器执行

绕过:
<script>
  ↓
大小写
  ↓
标签替代
  ↓
事件触发

修复:
输出编码
  ↓
HTML实体化
  ↓
浏览器不执行
```

---

下一节进入：

# 🔥 XSS Day 3：存储型XSS + Cookie窃取原理 + BeEF概念

内容：

1. DVWA Stored XSS
    
2. 为什么存储型危害更高
    
3. XSS攻击链
    
4. Cookie安全问题
    
5. HttpOnly防护
    
6. 企业真实案例分析
    

这一节会把 XSS 从“弹窗”提升到真实攻击场景。