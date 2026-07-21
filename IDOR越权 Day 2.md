# 🔥 IDOR越权 Day 2：高级测试技巧

上一节我们建立了 IDOR 的基础模型：

```text
用户登录
   ↓
访问资源
   ↓
修改对象标识
   ↓
服务器未校验权限
   ↓
越权访问
```

Day 2 进入真实项目测试思路。

重点从：

> “改一个 id 看看”

升级到：

> “系统所有资源访问控制是否正确？”

---

# 一、修改密码越权（高危）

修改密码是 IDOR 中最危险的一类。

原因：

普通信息泄露：

```text
查看别人数据
```

影响：

中高。

修改密码：

```text
控制别人账号
```

影响：

严重。

---

# 1. 正常业务流程

用户修改密码：

请求：

```http
POST /api/user/changePassword HTTP/1.1

Cookie:
session=xxxx


oldPassword=123456
newPassword=abcdef
```

服务器：

检查：

```text
当前登录用户
        |
        ↓
修改自己的密码
```

---

# 2. 存在漏洞情况

抓包：

```http
POST /api/user/changePassword


user_id=1001
newPassword=test123
```

注意：

这里出现：

```text
user_id
```

危险。

攻击者：

修改：

```http
user_id=1002
```

如果成功：

```text
用户A
   |
   |
修改
   |
   ↓
用户B密码
```

漏洞成立。

---

# 3. 为什么危险？

攻击链：

```text
水平越权修改密码

↓

登录目标账号

↓

查看敏感数据

↓

进一步利用
```

严重程度通常：

High / Critical。

---

# 二、文件下载越权

这是企业非常常见的问题。

场景：

企业系统：

- 合同下载
    
- 发票下载
    
- 报表下载
    
- 简历下载
    

---

## 正常请求

例如：

```http
GET /download?id=5001
```

返回：

```
合同.pdf
```

---

## 测试

修改：

```http
GET /download?id=5002
```

如果返回：

```
其他用户合同.pdf
```

说明：

存在文件下载越权。

---

# 重点关注参数

```text
id
file
file_id
doc_id
path
filename
attachment
download
```

---

# 三、文件路径型越权

例如：

请求：

```http
GET /download?file=user1/resume.pdf
```

尝试：

```http
GET /download?file=user2/resume.pdf
```

如果成功：

说明：

服务器没有校验：

```text
当前用户
      ↓
文件拥有者
```

---

# 四、API接口越权（重点）

现在企业大量使用：

REST API。

例如：

前端：

```javascript
GET /api/order/detail?id=1001
```

后端：

返回：

```json
{
"id":1001,
"price":999,
"address":"xxx"
}
```

---

测试：

修改：

```http
id=1002
```

观察：

返回是否变化。

---

# API越权常见位置

## 用户接口

```
/api/user/info
/api/user/detail
/api/account
```

---

## 订单接口

```
/api/order/detail
/api/order/delete
/api/order/update
```

---

## 管理接口

```
/api/admin/*
/api/manage/*
```

---

# 五、JWT权限绕过思路

JWT：

JSON Web Token

常见：

```text
Header.Payload.Signature
```

例如：

```text
xxxxx.yyyyy.zzzzz
```

---

Payload：

可能：

```json
{
"user":"admin",
"role":"user"
}
```

---

问题：

开发者错误相信：

```json
{
"role":"admin"
}
```

来自客户端。

---

例如：

普通用户：

```json
{
"id":1001,
"role":"user"
}
```

如果后端：

只解析：

```json
role
```

没有重新查询数据库。

风险：

用户修改：

```json
{
"role":"admin"
}
```

---

但是注意：

现代系统一般：

- 签名验证
    
- 服务端权限查询
    

所以 JWT 本身不是漏洞。

真正问题：

> 后端是否信任客户端携带的信息。

---

# 六、前端隐藏按钮分析

这是新人容易忽略的地方。

例如：

后台页面：

普通用户：

看不到：

```
删除用户
```

很多人认为：

安全。

错误。

---

因为：

前端：

只是控制显示。

例如：

HTML：

```html
<button style="display:none">
删除用户
</button>
```

接口：

可能仍然存在：

```
/admin/deleteUser
```

---

测试：

查看：

- JS文件
    
- API接口
    
- Network请求
    

寻找：

```text
delete
remove
admin
manage
```

---

核心：

> 前端隐藏 ≠ 权限控制

---

# 七、Burp Intruder批量验证IDOR

真实SRC：

不能手工改：

100个ID。

使用：

Burp Intruder。

---

## 场景：

请求：

```http
GET /api/user?id=1001
```

---

发送到：

```text
Intruder
```

---

设置变量：

```http
id=§1001§
```

---

Payload：

数字：

```
1001
1002
1003
1004
...
```

---

观察：

Response长度。

例如：

|ID|长度|
|---|---|
|1001|2300|
|1002|2305|
|1003|2301|

不同长度：

可能：

不同用户数据。

---

# 八、IDOR测试方法论升级

不要只测试：

```text
id++
```

应该建立：

## 1. 用户维度

A账号：

测试：

B账号资源

---

## 2. 功能维度

测试：

读取：

```
GET
```

修改：

```
POST
PUT
DELETE
```

---

## 3. 参数维度

测试：

```
user_id
uid
account_id
order_id
file_id
```

---

# 九、真实SRC挖掘流程

例如：

发现：

订单系统。

---

## Step 1

注册两个账号：

```
账号A
账号B
```

---

## Step 2

账号A创建订单：

```
订单ID=10001
```

---

## Step 3

抓请求：

```http
GET /order/detail?id=10001
```

---

## Step 4

切换账号B

访问：

```http
GET /order/detail?id=10001
```

---

结果：

如果：

返回订单：

漏洞。

---

# 十、漏洞报告模板（真实SRC格式）

## 标题

```
订单查看接口存在水平越权漏洞
```

---

## 漏洞描述

```
系统订单查询接口未校验当前用户与订单资源之间的归属关系。
攻击者登录普通账号后，可修改订单ID参数访问其他用户订单信息。
```

---

## 复现步骤

账号A：

请求：

```http
GET /api/order/detail?id=10001
```

返回：

```json
{
"order":"A订单"
}
```

账号B：

修改：

```http
GET /api/order/detail?id=10001
```

返回：

```json
{
"order":"A订单"
}
```

---

## 影响

攻击者可能：

- 获取其他用户订单
    
- 泄露个人信息
    
- 扩大攻击范围
    

---

## 修复

服务端增加：

```text
当前用户ID
        +
资源所属ID
        |
        ↓
权限校验
```

---

# 十一、IDOR漏洞总结模型

完整测试思路：

```text
发现资源ID
       |
       ↓
修改对象编号
       |
       ↓
切换用户身份
       |
       ↓
比较响应
       |
       ↓
确认权限边界
       |
       ↓
提交报告
```

---

# Day 2 掌握目标

完成后，你应该能够：

✅ 判断一个接口是否存在权限校验  
✅ 测试水平越权  
✅ 测试垂直越权  
✅ 分析API接口风险  
✅ 发现文件下载越权  
✅ 理解JWT权限设计问题  
✅ 使用Burp批量验证  
✅ 编写SRC级报告

下一节：

# 🔥 IDOR Day 3：综合实战（Pikachu + Juice Shop + SRC思路）

内容：

- Pikachu越权完整复盘
    
- OWASP Juice Shop权限漏洞
    
- API越权挖掘流程
    
- 业务逻辑漏洞分析
    
- IDOR与CSRF组合漏洞
    
- IDOR与信息泄露组合
    
- 完整SRC案例复盘
    

完成 Day 3 后，IDOR 模块就可以进入下一个 OWASP 核心：

# 🔥 SSRF（服务器请求伪造）模块。