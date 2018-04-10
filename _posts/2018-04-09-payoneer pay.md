---
layout: post
title: "payoneer 支付流程"
categories: pay
tags: payoneer
author: zch
---

* content
{:toc}


Payoneer 支付类似于 Oauth 授权，用户要支付费用给商家，必须授权给商家，授权过程中会将唯一标识的用户 id 作为 payeeId 注册到商户系统中，每个商家对应一个 programId。



![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/payoneer1.png)



## 商户账号

官网上注册一个账号，注册成功后，会发一份开发者文档到你的邮箱：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/payoneer3.png)

*programId：商户唯一标识ID，类似于公众号的appId*

*Username：登陆用户名*

*password：登陆密码*

*API password：用于 Basic Auth 授权用的*

**请求 Payoneer API 需要附带 programId、username 和 API password 进行 Basic Auth 授权。**



## 获取授权页面

请求 API

```bash
curl -X POST https://api.payoneer.com/v2/programs/{program_id}/payees/registration-link/

-H "Content-Type: application/json"
-d {"payee_id": "ClientPayeeID"} 
```

这时商户系统会返回一个经过授权的url，格式如下：

```bash
https://payouts.sandbox.payoneer.com/partners/lp.aspx?token=14c982ed04354629810375ddc9721312B6B5851C51
```

打开后页面：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/payoneer2.png)



## 付款

如果有账户，直接登陆，登陆后就将 payeeId 注册到对应的商户系统里面了，这时该用户就可以付款给商户了，付款 API 如下：

```bash
Curl -X POST https://api.sandbox.payoneer.com/v2/programs/{Program_Id}/charges  
-H "Content-Type: application/json"  
–d {"payee_id":"ID123", "amount":"1.2", "client_reference_id":"test_88",      "description":"Charge test", "currency":"USD" 
```

付款时只要附带了授权时注册到商户系统的 payeeId，就可以对相应的 payoneer 账户进行扣款了，有没有一种免密支付的体验？其实免密支付也是用这种类似的技术实现的。