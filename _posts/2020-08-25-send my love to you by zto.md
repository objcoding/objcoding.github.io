---
layout: post
title: "我的6封情书通过中通快递送达给你"
categories: ZTO
tags: 
author: zch
---

* content
{:toc}


## 1

```java
public void loveLetter() {
  String reason1 = "I love You!";
  String reason2 = "I love ZTO!";
  ZTO.send(MY.getGift())
    .reason(reason1)
    .reason(reason2)
    .to(YOU);
}
```

用中通快递寄送一份礼物给你

不只是因为我喜欢你

而是因为我喜欢中通快递 



## 2

```java
public void loveLetter2() {
  if (ZTO.coldChain()
      .send(MY.getAdmiration())
      .to(YOU)) {
    YOU.feel(MY.getLove());
  }
}
```

如果我的倾慕可以用中通冷链寄送

那你一定可以感受到

我 100% 真实新鲜的爱



## 3

```java
public void loveLetter3() {
  ZTO.setFirstWeightPrice(new BigDecimal("8"));
  MY.setLoveWeight(Long.MAX_VALUE);
  if (ZTO.send(MY.getLove()).to(YOU)) {
    System.out.println("I will bankrupt！！！");
  }
}
```

中通快递首重按照 8 元来计算的话

如果想把我的爱快递给你

那我可能会倾家荡产



## 4

```java
public void loveLetter4() {
  ZTO.global()
    .although(Condition.ANY_WHERE)
    .although(Condition.OFFLINE)
    .find(YOU)
    .send(MY.getLove())
    .to(YOU);
}
```

无论你在世界的哪个角落

就算网络不能抵达

中通国际也都能找到你 



## 5

```java
public void loveLetter5() {
  int num = 0;
  do {
    ZTO.send(MY.gifts(num))
      .by(ZTO.courier())
      .to(YOU);
  } while (++num < 1024);
  YOU.have(new byte[Integer.MAX_VALUE]);
}
```

让中通小哥将一件件礼物送到你手上

就这样

你拥有了全世界



## 6

```java
public void loveLetter6() {
  MY.setMyLastName("X");
  if (YOU.receivedEmail().from("Mrs.X")) {
    System.out.println("Would you like to sign for?");
  }
}
```

我的姓氏：X

如果寄一封收件人名字是 Mrs.X 的信给你

你愿意签收吗？



*文案：中通科技-规划部 徐蕊*

*代码：中通科技-技术平台部 张乘辉*

*源码地址：https://gist.github.com/objcoding/212432565d8b6a45ff8e742fdf55657b*