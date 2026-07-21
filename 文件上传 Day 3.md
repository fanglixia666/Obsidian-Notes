# 🔥 文件上传 Day 3：DVWA High分析 + 文件头检测 + 双后缀问题

今天进入文件上传漏洞更接近真实环境的阶段。

前面你已经完成：

```text
Low

↓

任意文件上传

↓

理解multipart/form-data

↓

Burp修改请求
```

以及：

```text
Medium

↓

Content-Type绕过思路
```

今天重点：

> 从“修改请求参数”进入“分析服务器校验逻辑”。

真实渗透测试中，核心不是记payload，而是：

```
上传文件
    |
    ↓
服务器做了哪些检查？
    |
    ↓
检查在哪里？
    |
    ↓
有没有逻辑缺陷？
```

---

# 一、DVWA High环境切换

进入：

```
DVWA
 ↓
DVWA Security
 ↓
High
```

然后：

```
Vulnerabilities
 ↓
File Upload
```

---

尝试上传：

```
test.php
```

一般返回：

```
You can only upload JPEG or PNG files.
```

说明：

High增加了检测。

---

# 二、查看High源码（核心）

位置：

```
DVWA/vulnerabilities/upload/source/high.php
```

典型代码：

```php
<?php

if(isset($_FILES['uploaded'])){


$target_path =
DVWA_WEB_PAGE_TO_ROOT .
"hackable/uploads/";


$target_path .= 
basename($_FILES['uploaded']['name']);


$file = $_FILES['uploaded']['tmp_name'];


$image_info = getimagesize($file);


if(
$image_info &&
(
$image_info['mime']=="image/jpeg"
||
$image_info['mime']=="image/png"
)
)
{

move_uploaded_file(
$file,
$target_path
);

echo "uploaded";

}

else
{

echo "You can only upload JPEG or PNG files.";

}

}

?>
```

---

# 三、High相比Medium变化在哪里？

对比：

## Low

```php
没有检查
```

---

## Medium

检查：

```php
$_FILES['uploaded']['type']
```

也就是：

HTTP里的：

```
Content-Type
```

---

## High

检查：

```php
getimagesize()
```

---

区别：

|等级|检查|
|---|---|
|Low|无|
|Medium|客户端Content-Type|
|High|文件真实内容|

---

# 四、为什么Medium的Content-Type可以绕？

回顾：

上传请求：

```http
Content-Type: image/jpeg
```

这个值：

来自：

浏览器。

攻击者可以：

Burp修改：

原：

```http
Content-Type: application/x-php
```

改：

```http
Content-Type: image/jpeg
```

服务器被骗。

---

# 五、High为什么绕不过Content-Type？

因为：

High：

不看：

```http
Content-Type
```

而是：

读取文件。

例如：

上传：

```
test.php
```

内容：

```php
<?php
echo "hello";
?>
```

服务器执行：

```php
getimagesize(test.php)
```

结果：

没有图片结构。

返回：

false。

---

所以：

修改：

```http
Content-Type:image/jpeg
```

无效。

---

# 六、Magic Number（文件头检测）

这是企业上传检测的重要概念。

什么是Magic Number？

文件开头固定字节。

例如：

## JPEG

十六进制：

```
FF D8 FF
```

---

## PNG

```
89 50 4E 47
```

---

## GIF

```
47 49 46 38
```

---

服务器可以读取：

文件前几个字节。

判断：

是不是图片。

---

例如：

一个PHP文件：

```
<?php
phpinfo();
?>
```

十六进制：

```
3C 3F 70 68 70
```

不是：

```
FF D8 FF
```

所以拒绝。

---

# 七、文件头绕过思想（理解即可）

如果检测：

只看：

```
文件开头
```

那么理论上：

文件可以：

前面放图片头。

后面放代码。

例如结构：

```
FF D8 FF E0

图片数据


<?php
代码
?>
```

---

这种叫：

图片马。

但是注意：

现代服务器：

即使上传成功：

还要满足：

```
上传目录可执行
+
服务器解析PHP
```

才危险。

---

所以文件上传漏洞判断：

不是：

上传成功=漏洞。

而是：

完整链：

```
上传
 |
 ↓
保存
 |
 ↓
可访问
 |
 ↓
可解析
 |
 ↓
代码执行
```

---

# 八、文件名解析陷阱

这是非常重要的一部分。

开发经常：

```php
$ext = pathinfo($filename, PATHINFO_EXTENSION);
```

判断：

扩展名。

例如：

```
test.jpg
```

扩展名：

```
jpg
```

---

但是攻击者会测试：

```
test.php.jpg
```

问题：

不同语言解析方式不同。

---

例如：

代码：

```php
explode(".", $filename)
```

得到：

```
[
test,
php,
jpg
]
```

---

如果开发取：

最后：

```
jpg
```

认为安全。

---

但是服务器解析规则可能不同。

所以：

文件名处理非常危险。

---

# 九、双后缀问题

常见：

```
xxx.php.jpg
```

为什么危险？

假设：

上传检查：

```text
最后一个后缀必须jpg
```

通过。

保存：

```
xxx.php.jpg
```

---

如果服务器配置错误：

可能：

把：

```
.php.
```

解析。

---

不同环境：

行为不同。

例如：

Apache：

依赖：

```
AddHandler
```

配置。

Nginx：

通常：

只解析最后扩展名。

---

所以：

双后缀：

属于：

```
配置依赖型风险
```

不是一定成功。

---

# 十、真实渗透测试上传检测流程

企业测试不会只上传一个php。

流程：

---

## 1. 正常文件测试

上传：

```
test.jpg
```

确认功能。

---

## 2. 后缀测试

测试：

```
test.php

test.php5

test.phtml

test.jsp

test.asp
```

观察。

---

## 3. Content-Type测试

修改：

```
image/jpeg
```

观察。

---

## 4. 文件内容测试

修改：

文件头。

---

## 5. 路径测试

确认：

上传位置。

例如：

响应：

```
/upload/abc.jpg
```

---

## 6. 访问测试

访问：

```
/upload/abc.jpg
```

观察：

- 是否可访问
    
- 是否执行
    
- 是否下载
    

---

# 十一、企业修复方案

真正安全上传：

## 1. 后缀白名单

推荐：

允许：

```
jpg
png
gif
```

拒绝：

```
php
jsp
exe
```

---

## 2. 服务端MIME检测

不要相信：

```
Content-Type
```

使用：

服务器检测。

---

## 3. Magic Number检测

读取文件头。

---

## 4. 随机文件名

不要：

```
user.jpg
```

保存：

```
8f72aa91.jpg
```

---

## 5. 上传目录禁止执行

例如：

Apache：

```
uploads/.htaccess
```

禁止：

```
php解析
```

---

## 6. 文件权限隔离

上传目录：

只允许：

```
write
read
```

不要：

```
execute
```

---

# 十二、今天实验任务

## 任务1

DVWA High：

上传：

```
test.jpg
```

抓Burp。

观察：

请求：

```
filename

Content-Type

文件内容
```

---

## 任务2

尝试：

修改：

```
Content-Type:image/jpeg
```

上传：

```
test.php
```

记录结果。

思考：

为什么失败？

---

## 任务3

查看源码：

找到：

```php
getimagesize()
```

理解：

它和：

```php
$_FILES['type']
```

区别。

---

# 十三、知识总结

现在你的文件上传漏洞知识树：

```
文件上传漏洞

├── Low
│
│   无检测
│
├── Medium
│
│   Content-Type检测
│       |
│       └── Burp修改
│
├── High
│
│   文件内容检测
│       |
│       ├── Magic Number
│       └── 图片结构
│
├── 文件名问题
│
│   ├── 双后缀
│   ├── 大小写
│   └── 特殊字符
│
└── 企业防护
    ├── 白名单
    ├── 内容检测
    ├── 随机命名
    └── 禁止执行
```

---

下一节进入：

# 🔥 文件上传 Day 4：真实上传漏洞挖掘思路（SRC/渗透测试流程）

内容：

1. 如何寻找上传点
    
2. Burp批量测试思路
    
3. 黑名单绕过原理
    
4. 白名单绕过原理
    
5. 解析漏洞理解
    
6. 上传漏洞报告编写
    
7. 企业修复验证流程
    

这一节会把DVWA实验连接到真实Web渗透流程。