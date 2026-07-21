# 🔥 IDOR Day 3：综合实战（Pikachu + Juice Shop + SRC思路）

前两节我们完成了：

Day 1：

```text
IDOR是什么
↓
水平越权
↓
垂直越权
↓
Burp测试基础
```

Day 2：

```text
修改密码越权
文件下载越权
API越权
JWT权限问题
Intruder批量验证
```

Day 3 进入综合实战。

目标：

> 从“会改ID”升级到“能发现业务授权缺陷”。

---

# 一、Pikachu 越权完整复盘

Pikachu 是国内非常经典的漏洞靶场。

进入：

```
Pikachu
 ↓
越权漏洞
```

常见分类：

- 水平越权
    
- 垂直越权
    

---

# 1. 水平越权

## 场景

系统存在用户信息查询：

请求：

```http
GET /pikachu/vul/overpermission/level_1.php?id=1
```

当前：

用户A登录。

返回：

```json
{
"id":1,
"username":"admin"
}
```

---

修改：

```
?id=2
```

请求：

```http
GET /pikachu/vul/overpermission/level_1.php?id=2
```

返回：

```json
{
"id":2,
"username":"test"
}
```

说明：

当前用户访问了其他用户数据。

---

## 漏洞本质

后端：

类似：

```php
$id=$_GET['id'];

$sql="
select *
from user
where id=$id
";
```

问题：

没有：

```text
判断当前登录用户
是否拥有该id资源
```

---

安全代码：

逻辑：

```text
当前用户ID
        +
目标资源ID

必须匹配
```

例如：

```sql
select *
from user
where id=$id
and owner_id=current_user
```

---

# 二、OWASP Juice Shop 权限漏洞

OWASP Juice Shop 是学习现代 Web 漏洞非常好的靶场。

重点关注：

- API
    
- JWT
    
- 前后端分离
    
- Angular 前端
    
- REST接口
    

---

## 1. 查看前端接口

打开：

浏览器：

```
F12
 ↓
Network
```

寻找：

```
/api/
```

例如：

```http
GET /api/Users/1
```

---

## 2. 用户ID测试

当前：

用户：

```
id=5
```

修改：

```
id=6
```

观察：

返回。

---

如果：

返回：

```json
{
"email":"other@example.com"
}
```

说明：

存在IDOR。

---

# 三、API越权挖掘流程（重点）

现代企业：

大量漏洞发生在 API。

因为：

前端只是调用接口。

---

# 1. 找接口

来源：

## Burp历史记录

查看：

```
HTTP history
```

关注：

```
/api/
/user/
/account/
/order/
/file/
```

---

## JS文件分析

下载：

```
main.js
app.js
```

搜索：

关键词：

```
admin
delete
update
user
id
```

---

# 2. 建立接口表

实际测试建议记录：

|接口|参数|功能|
|---|---|---|
|/user/info|id|查询用户|
|/order/detail|id|订单|
|/file/download|file_id|下载|

---

# 3. 测试权限边界

准备：

两个账号：

```
A账号
B账号
```

流程：

A创建资源：

```
订单1001
```

B访问：

```
订单1001
```

结果：

---

返回：

❌ 越权

403：

✅ 正确

---

# 四、业务逻辑漏洞分析

IDOR本质属于：

业务逻辑漏洞。

因为：

代码没有错误。

但是：

业务规则错误。

---

例如：

业务：

```
用户只能查看自己的订单
```

代码：

```
根据订单ID查询
```

缺少：

```
订单属于谁？
```

---

# 常见业务逻辑位置

## 订单

```
order_id
```

测试：

- 查看
    
- 修改
    
- 删除
    

---

## 钱相关

例如：

优惠券：

```
coupon_id
```

测试：

是否可以：

- 重复领取
    
- 跨用户使用
    

---

## 文件

例如：

```
resume_id
contract_id
```

---

## 用户操作

例如：

```
user_id
```

测试：

- 修改资料
    
- 修改密码
    
- 绑定手机
    

---

# 五、IDOR + CSRF组合漏洞

这是非常重要的攻击链。

前面学过：

CSRF：

> 利用用户登录状态执行操作。

IDOR：

> 操作对象没有权限校验。

两个结合：

危害提升。

---

场景：

修改邮箱接口：

```
POST /changeEmail
```

请求：

```http
user_id=1001
email=test@test.com
```

问题：

没有检查：

```
user_id是否属于当前用户
```

---

攻击流程：

用户登录：

↓

访问恶意页面：

↓

CSRF触发请求：

↓

修改其他用户信息

形成：

```
CSRF
 +
IDOR
 =
跨用户操作
```

---

# 六、IDOR + 信息泄露组合

常见：

接口：

```
/api/user/detail?id=
```

返回：

```json
{
"name":"xxx",
"phone":"138xxxx",
"id_card":"xxxx"
}
```

如果ID可遍历：

攻击者：

```
id=1
id=2
id=3
...
```

大量获取数据。

---

影响：

从：

普通IDOR

升级：

```
敏感信息泄露
```

严重程度提升。

---

# 七、真实SRC案例分析思路

假设发现：

接口：

```
GET /api/order?id=10001
```

---

## 第一步

注册账号：

```
A
B
```

---

## 第二步

A创建订单：

```
order_id=10001
```

---

## 第三步

A查看：

请求：

```
GET /api/order?id=10001
```

返回：

订单信息。

---

## 第四步

B登录：

访问：

```
GET /api/order?id=10001
```

---

结果：

如果返回：

A订单：

漏洞成立。

---

# 八、报告编写（SRC级）

## 标题

```
订单查询接口存在水平越权漏洞
```

---

## 漏洞描述

```
系统订单查询接口仅通过订单ID进行资源定位，
未校验当前用户与订单之间的归属关系，
导致普通用户可访问其他用户订单信息。
```

---

## 复现

账号A：

```
创建订单10001
```

请求：

```http
GET /api/order?id=10001
```

账号B：

请求：

```http
GET /api/order?id=10001
```

返回：

```
A用户订单信息
```

---

## 修复

增加：

```text
资源所有权验证

当前用户ID
        |
        +
订单所属用户ID

必须一致
```

---

# 九、IDOR综合测试Checklist

以后测试项目，可以直接套：

## 查询类

```
GET
/user?id=
/order?id=
/file?id=
```

测试：

✅ 是否能读取其他资源

---

## 修改类

```
POST
PUT
PATCH
```

测试：

✅ 是否能修改别人数据

---

## 删除类

```
DELETE
```

测试：

✅ 是否能删除别人资源

---

## 下载类

```
download?id=
```

测试：

✅ 是否能下载别人文件

---

# 十、IDOR模块最终总结

完整攻击模型：

```
发现对象编号
        |
        ↓
修改对象编号
        |
        ↓
切换身份验证
        |
        ↓
观察响应差异
        |
        ↓
确认权限缺失
        |
        ↓
形成漏洞报告
```

完成 IDOR 三节后，你现在掌握：

✅ 水平越权  
✅ 垂直越权  
✅ API越权  
✅ 文件越权  
✅ 修改密码越权  
✅ JWT权限设计问题  
✅ Burp验证流程  
✅ SRC报告方法

---

# 当前路线进度更新

你的 OWASP 模块现在：

```
SQL注入        ✅
XSS             ✅
CSRF            ✅
文件上传        ✅
IDOR            ✅

下一模块：

🔥 SSRF（Day34-36）
```

下一节进入：

# 🔥 SSRF漏洞 Day 1：原理 + 网络模型 + DVWA/Pikachu实战

重点：

- SSRF产生原因
    
- 服务端请求模型
    
- 内网探测
    
- 云环境Metadata风险
    
- URL过滤绕过
    
- Burp测试流程
    
- SSRF漏洞报告
    

完成 SSRF 后，你的 OWASP Top 10 核心漏洞模块基本闭环。