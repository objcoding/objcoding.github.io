---
layout: post
title: "解决fastJson循环引用的问题"
categories: Java
tags: json
author: 张乘辉
---

* content
{:toc}
最近发现返回给前段的json串出现了循环引用的问题，这是由于在json串中存在相同的属性信息，对象的属性之间存在相互引用导致循环，而fastjson支持循环引用/重复引用，并且是缺省打开的。所以会出现以下现象：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/json.jpg)









解决方式是关闭fastJson的循环引用：

```java
String listJsonStr = JSON.toJSONString(list, SerializerFeature.DisableCircularReferenceDetect);
```

但通常不这么写，我们直接让其项目启动的时候就生效，所以我们直接配置bean：

```java
@Configuration
@ConditionalOnClass({FastJsonHttpMessageConverter.class}) //1
@ConditionalOnProperty(//2
  name = {"spring.http.converters.preferred-json-mapper"},
  havingValue = "fastjson",
  matchIfMissing = true
)
protected static class FastJson2HttpMessageConverterConfiguration {
  protected FastJson2HttpMessageConverterConfiguration() {
  }

  @Bean
  @ConditionalOnMissingBean({FastJsonHttpMessageConverter.class})//3
  public FastJsonHttpMessageConverter fastJsonHttpMessageConverter() {
    FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
    FastJsonConfig fastJsonConfig = new FastJsonConfig();//4
    fastJsonConfig.setSerializerFeatures(SerializerFeature.WriteNullBooleanAsFalse,
                                         SerializerFeature.WriteNullStringAsEmpty,
                                         SerializerFeature.WriteNullNumberAsZero,
                                         SerializerFeature.DisableCircularReferenceDetect
                                        );

    ValueFilter valueFilter = (o, s, o1) -> {
      //5
      //o 是class
      //s 是key值
      //o1 是value值
      if (null == o1) {
        return null;
      }
      return o1;
    };


    fastJsonConfig.setSerializeFilters(valueFilter);

    converter.setFastJsonConfig(fastJsonConfig);

    return converter;
  }
}
```

