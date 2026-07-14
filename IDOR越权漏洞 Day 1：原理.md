# 🔥 IDOR越权漏洞 Day 1：原理 + DVWA/Pikachu + Burp实战

进入新的 OWASP 模块：

# Insecure Direct Object Reference（IDOR）

中文：

> **不安全的直接对象引用**

在新版 OWASP Top 10 中，它属于：

> **访问控制失效（Broken Access Control）**

这是 SRC 和企业安全测试中非常高频的一类漏洞。

---

# 一、什么是IDOR？

一句话：

> 用户可以通过修改请求中的对象标识符，访问本不属于自己的资源。

核心问题：

**服务器只验证“你登录了”，没有验证“你有没有权限访问这个资源”。**

---

正常逻辑：

```
用户A登录
        |
        |
访问自己的数据
        |
        ↓
服务器检查：
你是谁？
你是否拥有这个数据？
```

安全：

```
用户A
 |
请求
 |
服务器权限判断
 |
返回A的数据
```

---

漏洞逻辑：

```
用户A登录

↓

请求：

/user?id=1001


↓

修改：

/user?id=1002


↓

服务器：

"你登录了，那给你数据"

↓

返回用户B数据
```

问题：

缺少：

```
资源归属检查
```

---

# 二、IDOR产生原因

常见后端代码：

## 不安全：

PHP：

```php
$id=$_GET['id'];

$sql="
SELECT *
FROM users
WHERE id=$id
";

$result=mysqli_query($conn,$sql);
```

问题：

只判断：

```
id存在
```

没有判断：

```
当前用户 == 数据拥有者
```

---

安全：

例如：

```php
SELECT *
FROM users
WHERE id=$id
AND user_id=$current_user
```

增加：

```
权限绑定
```

---

# 三、IDOR核心分类

主要分：

# 1. 水平越权（Horizontal Privilege Escalation）

同级用户之间。

例如：

系统：

```
用户A
用户B
用户C
```

权限一样。

用户A：

查看订单：

```
/order?id=1001
```

修改：

```
/order?id=1002
```

如果看到B订单：

漏洞：

```
水平越权
```

---

# 2. 垂直越权（Vertical Privilege Escalation）

不同权限。

例如：

普通用户：

```
role=user
```

管理员：

```
role=admin
```

普通用户访问：

```
/admin/deleteUser
```

成功：

漏洞：

```
垂直越权
```

---

# 四、IDOR和认证的区别

很多新人容易混淆。

## 认证 Authentication

问题：

```
你是谁？
```

例如：

登录：

```
username
password
```

---

## 授权 Authorization

问题：

```
你能做什么？
```

例如：

普通用户：

能不能：

```
删除用户？
查看订单？
修改密码？
```

---

IDOR属于：

```
认证成功

但是授权失败
```

---

# 五、DVWA中的IDOR模拟

DVWA没有专门IDOR模块。

可以使用：

- DVWA Weak Session IDs
    
- DVWA User ID类场景
    

或者使用：

## Pikachu靶场

推荐：

```
Pikachu
 ↓
Over permission
```

---

# 六、Pikachu水平越权实战思路

场景：

用户中心：

```
查看用户信息
```

请求：

```
GET /pikachu/vul/overpermission/level_1.php?id=1
```

---

当前用户：

```
user1
```

查看：

```
id=1
```

返回：

```
user1信息
```

---

修改：

```
?id=2
```

如果：

返回：

```
user2信息
```

说明：

存在水平越权。

---

# 七、Burp测试流程（重点）

真实项目主要靠这个。

---

## Step 1：找到敏感接口

重点关注：

```
/user/info
/user/detail
/order/query
/account
/download
/file
/api/
```

---

例如：

抓包：

```http
GET /api/order/detail?id=10001 HTTP/1.1

Cookie:
session=xxxxx
```

---

## Step 2：修改对象ID

原：

```
id=10001
```

改：

```
id=10002
```

发送。

---

## Step 3：比较响应

观察：

### 状态码

例如：

正常：

```
200
```

异常：

```
403
```

---

### 返回内容

原：

```json
{
"name":"Alice",
"order":"phone"
}
```

修改：

```json
{
"name":"Bob",
"order":"laptop"
}
```

漏洞。

---

# 八、常见IDOR参数位置

测试时重点关注：

---

## 用户类

```
uid
user_id
userid
account
member_id
```

---

## 文件类

```
file_id
doc_id
download_id
attachment
```

---

## 订单类

```
order_id
trade_id
invoice_id
```

---

## API类

```
id
uuid
key
token
```

---

# 九、IDOR测试技巧

## 1. 替换数字

例如：

```
1001
1002
1003
```

---

## 2. 替换UUID

例如：

原：

```
550e8400-e29b
```

测试：

其他UUID。

---

## 3. 修改HTTP方法

例如：

原：

```http
GET /user?id=2
```

尝试：

```http
POST /user/update
```

---

## 4. 参数污染

例如：

请求：

```
id=1001&id=1002
```

观察：

后端处理哪个。

---

# 十、真实SRC测试思路

寻找：

## 个人数据

高价值：

```
手机号
身份证
地址
订单
账单
聊天记录
```

---

## 操作接口

更危险：

```
修改密码
修改邮箱
绑定手机
删除账号
上传文件
```

---

原因：

读取数据：

普通IDOR

修改数据：

严重IDOR

---

# 十一、漏洞等级判断

|类型|影响|
|---|---|
|查看公开信息|低|
|查看个人信息|中|
|查看敏感数据|高|
|修改他人数据|高|
|删除他人资源|高|
|管理员功能越权|严重|

---

# 十二、IDOR漏洞报告模板

## 标题

```
水平越权导致任意用户敏感信息泄露
```

---

## 描述

```
系统接口未对用户资源访问权限进行校验，
攻击者可修改请求参数中的用户标识，
访问其他用户数据。
```

---

## 请求

```http
GET /api/user/info?id=1002

Cookie:
session=xxx
```

---

## 结果

返回：

```json
{
"name":"other_user",
"phone":"xxx"
}
```

---

## 修复建议

### 1. 服务端权限校验

不要相信：

```
客户端传入ID
```

---

### 2. 资源绑定用户

例如：

```
查询订单

必须：

order.user_id == current_user.id
```

---

### 3. 使用随机不可预测ID

例如：

UUID

但注意：

UUID不是权限控制。

---

# Day 1 总结

今天建立：

```
登录成功
    |
    |
访问资源
    |
    |
是否验证资源归属？
    |
    |
没有
    |
    ↓
IDOR越权
```

掌握：

✅ IDOR定义  
✅ 水平越权  
✅ 垂直越权  
✅ 认证与授权区别  
✅ Burp测试流程  
✅ SRC寻找方向  
✅ 报告编写思路

下一节：

# 🔥 IDOR越权 Day 2：高级测试技巧

内容：

- 修改密码越权
    
- 文件下载越权
    
- API接口越权
    
- JWT权限绕过思路
    
- 前端隐藏按钮分析
    
- Burp Intruder批量验证
    
- SRC真实案例分析思路
    

这一节之后，IDOR模块基本达到实际测试水平。