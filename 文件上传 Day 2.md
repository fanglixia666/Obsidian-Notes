# 🔥 文件上传 Day 2：DVWA Medium绕过分析

今天进入文件上传漏洞的核心阶段。

上一节你已经完成：

```text
DVWA Low

上传任意文件

↓

理解multipart/form-data

↓

Burp查看上传请求结构
```

今天开始分析：

> **为什么开发人员增加了限制后，仍然可能存在上传漏洞。**

---

# 一、DVWA Medium文件上传环境

进入：

```text
DVWA

↓

DVWA Security

↓

Medium
```

进入：

```text
Vulnerabilities

↓

File Upload
```

---

尝试上传：

普通文件：

```text
test.jpg
```

应该可以。

---

尝试上传：

```text
test.php
```

观察。

可能出现：

```text
Only JPEG and PNG images are allowed.
```

说明：

Medium增加了检测。

---

# 二、查看Medium源码（重点）

文件：

```text
DVWA/vulnerabilities/upload/source/medium.php
```

打开。

你会看到类似：

```php
<?php

if(isset($_FILES['uploaded'])){


$target_path =
DVWA_WEB_PAGE_TO_ROOT .
"hackable/uploads/";


$target_path .=
basename($_FILES['uploaded']['name']);


$uploaded_type =
$_FILES['uploaded']['type'];


if(
($uploaded_type == "image/jpeg")
||
($uploaded_type == "image/png")
)
{

move_uploaded_file(
$_FILES['uploaded']['tmp_name'],
$target_path
);

echo "uploaded";

}

else{

echo "Only JPEG and PNG images are allowed.";

}

}

?>
```

---

# 三、漏洞在哪里？

重点：

```php
$_FILES['uploaded']['type']
```

这里。

---

很多开发认为：

```php
Content-Type=image/jpeg
```

代表：

文件就是图片。

错误。

---

实际上：

Content-Type来自：

客户端。

也就是说：

浏览器告诉服务器：

> 我上传的是图片。

服务器相信了。

---

这就是：

# 客户端数据不可信

---

# 四、HTTP上传请求分析

打开Burp：

上传：

```text
test.jpg
```

抓包。

请求：

```http
POST /DVWA/vulnerabilities/upload/

HTTP/1.1

Host: lvh.me

Content-Type:
multipart/form-data;
boundary=----WebKitFormBoundary
```

---

文件部分：

```http
------WebKitFormBoundary

Content-Disposition:
form-data;
name="uploaded";
filename="test.jpg"


Content-Type:
image/jpeg


文件内容


------WebKitFormBoundary--
```

---

注意三个位置：

---

## 1. filename

```text
filename="test.jpg"
```

文件名。

---

## 2. Content-Type

```text
Content-Type:
image/jpeg
```

类型声明。

---

## 3. 文件真实内容

例如：

```text
<?php

echo "test";

?>
```

---

三者可以不一致。

---

# 五、第一次实验：修改Content-Type

目标：

验证：

> 只改Content-Type是否可以绕过。

---

准备：

创建：

```text
test.php
```

内容：

```php
<?php

echo "upload test";

?>
```

---

上传。

Burp拦截。

找到：

```http
filename="test.php"
```

保持。

修改：

原：

```http
Content-Type:
application/x-php
```

改：

```http
Content-Type:
image/jpeg
```

---

完整：

变成：

```http
Content-Disposition:
form-data;
name="uploaded";
filename="test.php"


Content-Type:
image/jpeg


<?php
echo "upload test";
?>
```

---

Forward。

---

# 六、结果分析

如果成功：

说明：

DVWA Medium只检查：

```text
$_FILES['uploaded']['type']
```

也就是：

Content-Type。

---

如果失败：

检查：

Security等级。

确保：

```text
Medium
```

不是：

High。

---

# 七、为什么Content-Type可以伪造？

HTTP协议：

本质：

文本。

例如：

请求头：

```http
User-Agent:
Chrome
```

是不是Chrome？

服务器不知道。

---

Cookie：

也可以修改。

例如：

```http
Cookie:
admin=true
```

---

Referer：

也可以修改。

---

所以：

客户端提交：

全部不可信。

包括：

- User-Agent
    
- Cookie
    
- Referer
    
- Content-Type
    
- filename
    

---

# 八、企业开发为什么不能相信Content-Type？

错误：

```php
if($_FILES['file']['type']=="image/jpeg")
```

原因：

这个值来自：

```text
HTTP请求
```

不是：

文件本身。

---

正确方式：

检查：

文件内容。

---

例如：

PHP：

```php
$finfo = finfo_open(FILEINFO_MIME_TYPE);

$type=finfo_file(
$finfo,
$file
);
```

---

服务器自己判断：

而不是相信客户端。

---

# 九、Burp修改上传包技巧

这是以后渗透非常常用技能。

流程：

```text
浏览器上传

↓

Burp Proxy

↓

Intercept

↓

修改请求

↓

Forward

↓

观察响应
```

---

重点修改位置：

## 文件名

测试：

```text
avatar.jpg
```

变：

```text
avatar.php
```

---

## Content-Type

测试：

```text
image/jpeg
```

---

## 内容

测试：

```text
文件头
文件内容
```

---

# 十、文件上传漏洞检测思路（企业）

真实测试：

不是直接上传危险文件。

流程：

---

## 1. 找上传点

例如：

- 头像
    
- 附件
    
- 工单
    
- 简历
    
- 图片
    

---

## 2. 判断限制

测试：

|测试|目的|
|---|---|
|jpg|正常上传|
|txt|类型限制|
|php|危险后缀|
|改Content-Type|客户端校验|
|改文件名|后缀检测|

---

## 3. 判断文件位置

上传后：

观察：

返回：

```text
uploads/test.jpg
```

---

## 4. 判断是否可执行

关键：

上传目录：

是否解析脚本。

---

# 十一、今天实验任务

完成：

## 任务1

DVWA Medium：

上传：

```text
test.jpg
```

Burp抓包。

---

## 任务2

修改：

```http
Content-Type:image/jpeg
```

观察结果。

---

## 任务3

回答：

为什么：

```php
$_FILES['uploaded']['type']
```

不安全？

---

## 任务4

画流程：

```text
浏览器

↓

multipart/form-data

↓

Content-Type

↓

PHP接收

↓

服务器判断

↓

保存文件
```

---

# 十二、今天掌握后，你进入真实上传漏洞分析阶段

知识树：

```text
文件上传漏洞

├── Low
│
│   任意上传
│
├── Medium
│
│   Content-Type绕过
│
├── High
│
│   文件内容检测
│
├── 黑名单绕过
│
├── 白名单绕过
│
└── 企业防护
    ├── 后缀白名单
    ├── MIME检测
    ├── 文件头检测
    ├── 随机文件名
    └── 禁止执行
```

---

下一节：

# 🔥 文件上传 Day 3：DVWA High分析 + 文件头检测 + 双后缀问题

内容：

1. High源码分析
    
2. 为什么Content-Type绕不过
    
3. Magic Number文件头检测
    
4. 文件名解析陷阱
    
5. 双后缀问题
    
6. 上传漏洞真实检测流程
    
7. 企业修复方案
    

这一节开始接近真实SRC/渗透测试中的上传漏洞挖掘思路。