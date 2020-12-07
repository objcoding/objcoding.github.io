---
layout: post
title: "使用Collection的Stream拼装数据"
categories: Java
tags: Java8 Lambda Collection Stream
author: 张乘辉
---

* content
{:toc}
在分布式系统上开发时，各个子系统又有独立的数据库，进行跨数据库联表查询往往导致系统性能会很差，这相当于把业务逻辑也放在数据库上执行了，且大量采用关联查询会导致项目很难进行分布式扩展。

这时候，我想到了Collection的Stream API，再结合Lambda表达式，解决拼装数据再完美不过了。





假设有个商户实体类：

```java
public class Shop implements Serializable {
  private Integer shopId;
  private String shopCode;// 商户标志码(商户自编码)
  private String shopNo;// 商户联盟号
  private String shopName;// 商户名称
  private String shopBrand;// 商户品牌
  // 此处省略gettet和setter
}
```

管理员实体类：

```java
public class Admin implements Serializable {
  private Integer shopId;
  private String adminName;
  private String adminAge;
  private String adminSex;
  // 此处省略gettet和setter
}
```

现在页面列表有个需求需要每列展示管理员信息和其对应的商户信息，这时我们创建一个包装类并继承Admin：

```java
public class Admin4WebManage extends Admin {
  // 需要拼装的商户信息字段
  private String shopCode;// 商户标志码(商户自编码)
  private String shopNo;// 商户联盟号
  private String shopName;// 商户名称
  private String shopBrand;// 商户品牌
  
  public Admin4WebManage(Admin admin) {
    setshopId(admin.getShopId());
    setAdminName(admin.getAdminName());
    setAdminAge(admin.getAdminAge());
    setAdminSex(admin.getAdminSex());
  }
  // 此处省略gettet和setter 
}
```

从数据库获取信息：

```java
public List<Admin4WebManage> getAdminInfos() {
  List<Admin> admins = adminDao.getAdmins();
  List<Admin4WebManage> admin4WebManages =
    // Lambda表达式之方法引用
    Admins.stream().map(this::admin2Domain).collect(Collectors.toList());
}
```

拼装方法：

```java
private Admin4WebManage admin2Domain(Admin admin) {
  Admin4WebManage webManage = new Admin4WebManage(admin);// 构造方法中添加admin信息
  String shopId = webManage.getShopId();
  if (shopId != null) {
    // 从其它子系统中获取商户信息
    Shop shop = shopRest.getShopById(Integer.parseInt(shopId));
    if (shop != null) {// 拼装商户信息
      webManage.setShopCode(shop.getShopCode());
      webManage.setShopNo(shop.getShopNo());
      webManage.setShopNo(shop.getShopName());
      webManage.setShopBrand(shop.getShopBrand());
    }
  }
  return webManage;
}
```

使用Spring提供的RestTemplate类用于访问集群内其它服务器：

```java
@Component
public class ShopRest {

  @Autowired
  private RestTemplate restTemplate;

  public Shop getShopById(int shopId) {
    JSONObject shopJson = restTemplate.getForObject("http://shop-server/api/shop/" + shopId, JSONObject.class);
    if (shopJson.getInteger("errcode") == 0) {
      return JsonUtil.Json2Object(shopJson.getJSONObject("shop"), Shop.class);
    }
    return null;
  }
}
```

