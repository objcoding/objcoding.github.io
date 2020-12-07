---
layout: post
title: "Dubbo服务暴露之注册地址和端口"
categories: Dubbo
tags: 服务暴露
author: 张乘辉
---

* content
{:toc}
今天在SIT环境部署dubbo容器的时候遇到一个问题, 明明我的dubbo容器已经注册到zk了, 而且在dubbo控制台也能看到提供者和消费者的信息, 可就是死活调用不了, 仔细一看, 发现提供者的地址不对啊, 这明明不是我的dubbo容器对应的那天服务器地址.这是为什么呢?下面我们来分析一波dubbo服务暴露注册地址和端口那一段代码. 







## 背景分析

由于我公司是做海外业务, 服务器的部署都在国外, 由于一些政策原因, 我们是访问不了的, 这时候就需要服务器端口地址进行转发, 我们的做法是在一台国内机器上面再用一层nginx做中间转发, 那么这时候问题就来了, 如果我们部署的项目地址是本机的话, 服务之间是调用不通的, 首先说明项目的服务器也是通过跳板机的形式登陆的. 我们回到文章刚开始那个问题, 其实是没错的, 确实得注册到国内作跳板机服务器的地址上, 原因是因为提供转发的那台机器没有做对我的项目地址进行nginx映射. 由于我初来乍到, 这就坑了我一波啊, 我肯定不甘心的.



## 源码分析

我们直接就定位到dubbo服务暴露url组装的方法`com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol`这个方法源码太多了, 其实查找Host和Port就两行调用代码:

```java
 private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
     // ...
     // 查找host
     String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
     // 查找port
     Integer port = this.findConfigedPorts(protocolConfig, name, map);
     // ...
 }
```

继续往下撸:

```java
private String findConfigedHosts(ProtocolConfig protocolConfig, List<URL> registryURLs, Map<String, String> map) {
    boolean anyhost = false;

    String hostToBind = getValueFromConfig(protocolConfig, Constants.DUBBO_IP_TO_BIND);
    if (hostToBind != null && hostToBind.length() > 0 && isInvalidLocalHost(hostToBind)) {
        throw new IllegalArgumentException("Specified invalid bind ip from property:" + Constants.DUBBO_IP_TO_BIND + ", value:" + hostToBind);
    }

    // if bind ip is not found in environment, keep looking up
    if (hostToBind == null || hostToBind.length() == 0) {
        hostToBind = protocolConfig.getHost();
        if (provider != null && (hostToBind == null || hostToBind.length() == 0)) {
            hostToBind = provider.getHost();
        }
        if (isInvalidLocalHost(hostToBind)) {
            anyhost = true;
            try {
                hostToBind = InetAddress.getLocalHost().getHostAddress();
            } catch (UnknownHostException e) {
                logger.warn(e.getMessage(), e);
            }
            if (isInvalidLocalHost(hostToBind)) {
                if (registryURLs != null && !registryURLs.isEmpty()) {
                    for (URL registryURL : registryURLs) {
                        if (Constants.MULTICAST.equalsIgnoreCase(registryURL.getParameter("registry"))) {
                            // skip multicast registry since we cannot connect to it via Socket
                            continue;
                        }
                        try {
                            Socket socket = new Socket();
                            try {
                                SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
                                socket.connect(addr, 1000);
                                hostToBind = socket.getLocalAddress().getHostAddress();
                                break;
                            } finally {
                                try {
                                    socket.close();
                                } catch (Throwable e) {
                                }
                            }
                        } catch (Exception e) {
                            logger.warn(e.getMessage(), e);
                        }
                    }
                }
                if (isInvalidLocalHost(hostToBind)) {
                    hostToBind = getLocalHost();
                }
            }
        }
    }

    map.put(Constants.BIND_IP_KEY, hostToBind);

    // registry ip is not used for bind ip by default
    String hostToRegistry = getValueFromConfig(protocolConfig, Constants.DUBBO_IP_TO_REGISTRY);
    if (hostToRegistry != null && hostToRegistry.length() > 0 && isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
    } else if (hostToRegistry == null || hostToRegistry.length() == 0) {
        // bind ip is used as registry ip by default
        hostToRegistry = hostToBind;
    }

    map.put(Constants.ANYHOST_KEY, String.valueOf(anyhost));

    return hostToRegistry;
}
```

这个方法是获取地址的方法, 从源码我们也看到了这个地址分为两个, 分别是hostToBind和hostToRegistry, 这里很巧妙, 从名称看这是绑定地址和注册地址, 也就是绑定地址用于本地netty监听, 而注册地址是注册到zk上面提供给消费者调用的, 一般来说, 如果没有特殊要求, dubbo是拿本地服务器网卡0的内网ip作为地址, 绑定地址和注册地址都一样, 到是我们公司就遇到了这么个坑爹问题, 服务器在国外,如果绑定地址和监听地址都一样, 那么服务之间就无法通讯了,我们必须在注册地址上面做点手脚才行, 因此我继续往下看.



```java
private String getValueFromConfig(ProtocolConfig protocolConfig, String key) {
    String protocolPrefix = protocolConfig.getName().toUpperCase() + "_";
    String port = ConfigUtils.getSystemProperty(protocolPrefix + key);
    if (port == null || port.length() == 0) {
        port = ConfigUtils.getSystemProperty(key);
    }
    return port;
}
```

该方法的作用是从系统环境变量里面获取地址信息：

```java
/**
 * System environment -> System properties
 *
 * @param key key
 * @return value
 */
public static String getSystemProperty(String key) {
  String value = System.getenv(key);
  if (value == null || value.length() == 0) {
    value = System.getProperty(key);
  }
  return value;
}
```

这里`System.getenv(key);`方法作用是获取环境变量key中的地址值，如果环境变量没有配置对应key的值，那么就会调用`System.getProperty(key);`方法自动获取本地网卡的内网地址。

```java
public class Constants {
  public static final String DUBBO_IP_TO_REGISTRY = "DUBBO_IP_TO_REGISTRY";
  public static final String DUBBO_PORT_TO_REGISTRY = "DUBBO_PORT_TO_REGISTRY";
  public static final String DUBBO_IP_TO_BIND = "DUBBO_IP_TO_BIND";
  public static final String DUBBO_PORT_TO_BIND = "DUBBO_PORT_TO_BIND";
}
```

以上就是我们地址和端口的绑定与注册的key值，如果我们想要将指定的地址和端口注册到zk，我们可以在本地配置环境变量：

```bash
export DUBBO_DUBBO_IP_TO_REGISTRY=指定ip
export DUBBO_PORT_TO_REGISTRY=指定端口
```

我将本地服务器这两个环境变量配置成跳板机的ip和端口，下面我还要将跳板机配置nginx映射。

```nginx
upstream service-172.17.10.121-20040 {
    server 172.17.10.121:20040 weight=1 max_fails=2 fail_timeout=30s;
}

server {
    listen 20040;
    access_log /log/nginx/access_service-172.17.10.121-20040.log str_basic;
    error_log /log/nginx/error_service-172.17.10.121-20040.log;
    proxy_connect_timeout 2s;
    proxy_pass service-172.17.10.121-20040;
}

```

由于我的服务注册到zk的地址是跳板机的地址，那么其他消费者调用我提供的服务，就会访问跳板机，然后跳板机再做一层转发，就完美解决了这个问题。