---
layout: post
title: "Payoneer 支付流程"
categories: Payment
tags: Payoneer
author: 张乘辉
---

* content
{:toc}


Payoneer 支付类似于 Oauth 授权，用户要支付费用给商家，必须授权给商家，授权过程中会将唯一标识的用户 id 作为 payeeId 注册到商户系统中，每个商家对应一个 programId。



![](https://gitee.com/objcoding/md-picture/raw/master/img/payoneer1.png)



## 商户账号

官网上注册一个账号，注册成功后，**你还需要跟 payoneer API 对接的相关客服申请开发者 API 对接的信息，因为他们的 API 接口是不开放的**，申请通过后会发一份 API 接口需要的信息的文档给你，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/payoneer3.png)

*programId：商户唯一标识ID，类似于公众号的appId*

*Username：登陆用户名*

*password：登陆密码*

*API password：用于 Basic Auth 授权用的*

**请求 Payoneer API 需要附带 programId、username 和 API password 进行 Basic Auth 授权。**



## 获取授权状态

授权状态请求 API

```
curl -X POST https://api.payoneer.com/v2/programs/{program_id}/payees/{payee_id}/status/

-H "Content-Type: application/json"
-d {"payee_id": "ClientPayeeID"} 
```

授权状态返回数据：

- payeeId 已经授权：

```json
{
  "audit_id": 38935280,
  "code": 0,
  "description": "Success",
  "status": "ACTIVE"
}
```

- payeeId 未授权：

```json
{
  "audit_id": 38935299,
  "code": 10005,
  "hint": "Please ensure that the payee has registered with Payoneer",
  "description": "Payee was not found"
}
```

**前端根据该接口返回的数据判断该用户是否有授权，如果有授权直接跳到付款页面，没有则去请求授权页面接口。**



## 获取授权页面

获取授权页面请求 API

```bash
curl -X POST https://api.payoneer.com/v2/programs/{program_id}/payees/registration-link/

-H "Content-Type: application/json"
-d {"payee_id": "ClientPayeeID"} 
```

*注：最好用后台用户表的用户 id 作为 payeeId。*

这时商户系统会返回一个经过授权的 url，格式如下：

```bash
https://payouts.sandbox.payoneer.com/partners/lp.aspx?token=14c982ed04354629810375ddc9721312B6B5851C51
```

打开后页面：

![](https://gitee.com/objcoding/md-picture/raw/master/img/payoneer2.png)

如果有账户，直接登陆，登陆后就将 payeeId 注册到对应的商户系统里面了，这时该用户就可以付款给商户了。



## 授权成功重定向

授权成功之后，需要重定向到商户付款页面，重定向地址需要在商户账户后台开发人员选项里面配置：

![](https://gitee.com/objcoding/md-picture/raw/master/img/payoneer6.png)



## 授权回调

类似于微信支付的支付回调，微信那边处理完成后，回调到商户系统，让商户系统更新支付信息，payoneer 也不例外，在用户授权成功之后，异步回调到商户系统，让商户系统更新用户的授权状态，避免每次需要调用授权状态接口去校验用户的授权状态了。

这个回调地址也是在商户账户后台开发人员选项里面配置：

![](https://gitee.com/objcoding/md-picture/raw/master/img/payoneer5.png)



## 主动付款

付款 API 如下：

```bash
Curl -X POST https://api.sandbox.payoneer.com/v2/programs/{Program_Id}/charges  
-H "Content-Type: application/json"  
–d {"payee_id":"ID123", "amount":"1.2", "client_reference_id":"test_88",      "description":"Charge test", "currency":"USD"}
```

付款时只要附带了授权时注册到商户系统的 payeeId（也就是付款时拿到的该用户 ID） 就可以对相应的 payoneer 账户进行扣款了，有没有一种免密支付的体验？其实免密支付也是用这种类似的技术实现的。



## 流程图



![](https://gitee.com/objcoding/md-picture/raw/master/img/payoneer4.png)