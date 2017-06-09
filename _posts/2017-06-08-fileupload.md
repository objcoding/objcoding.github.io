---
layout: post
title: "使用Apache的Fileupload工具实现文件上传"
categories: Java
tags: 文件上传 Apache Fileupload IO
author: zch
---

* content
{:toc}
## 1. commons-fileupload API

这个小组件，它会帮我们解析request中的上传数据，解析后的结果是一个表单项数据封装到一个FileItem对象中。我们只需要调用FileItem的方法即可。

### 1.1 Maven依赖

```xml
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.3.2</version>
</dependency>
```

### 1.2 相关类

- 工厂：DiskFileItemFactory
- 解析器：ServletFileUpload
- 表单项：FileItem

### 1.3上传三大步

1.  创建工厂：`DiskFileItemFactory factory = new DiskFileItemFactory();`
2.  创建解析器：`ServletFileUpload sfu = new ServletFileUpload(factory);`
3.  使用解析器解析request：`List<FileItem> fileItemList = sfu.parseRequest(request);`

### 1.4 FileItem API

```java
boolean isFormField();// 是否为普通表单项！返回true为普通表单项，如果为false即文件表单项！
String getFieldName();// 返回当前表单项的名称；
String getString(String charset);// 返回表单项的值；
String getName();// 返回上传的文件名称
long getSize();// 返回上传文件的字节数
InputStream getInputStream();// 返回上传文件对应的输入流
void write(File destFile);// 把上传的文件内容保存到指定的文件中。
String getContentType();
```

### 1.5 需要注意的一些上传细节

#### 1.5.1 保存地址

上传到服务器的地址最好是在WEB-INF下，因为这个目录浏览器是访问不到的

#### 1.5.2 文件名称相关问题

- 有的浏览器上传的文件名是绝对路径，这需要切割！C:\files\baibing.jpg

```java
String filename = fi2.getName();
    int index = filename.lastIndexOf("\\");
    if(index != -1) {
        filename = filename.substring(index+1);
    }
```

- 文件名乱码或者普通表单项乱码：`request.setCharacterEncoding("utf-8");`因为fileupload内部会调用`request.getCharacterEncoding();`   > `request.setCharacterEncoding("utf-8");`//优先级低`servletFileUpload.setHeaderEncoding("utf-8");`//优先级高
- 文件同名问题；我们需要为每个文件添加名称前缀，这个前缀要保证不能重复。uuid    > `filename = CommonUtils.uuid() + "_" + filename;`

#### 1.5.3 目录打散

不能在一个目录下存放之多文件：

* 首字符打散：使用文件的首字母做为目录名称，例如：abc.txt，那么我们把文件保存到a目录下。如果a目录这时不存在，那么创建之；
* 时间打散：使用当前日期做为目录；
* 哈希打散：1. 通过文件名称得到int值，即调用hashCode()；2. 它int值转换成16进制0~9, A~F；3.  获取16进制的前两位用来生成目录，目录为二层！例如：1B2C3D4E5F，/1/B/保存文件。

#### 1.5.4 上传文件的大小限制

* 单文件上传大小控制：`sfu.setFileSizeMax(100*1024)：`限制单个文件大小为100KB（必须在解析开始之前调用）
* 整个请求大写控制：`sfu.setSizeMax(1024 * 1024);`//限制整个表单大小为1M

#### 1.5.5 缓存大小与临时目录

* 缓存大小：超出多大，才向硬盘保存！默认为10KB;
* 临时目录：向硬盘的什么目录保存;
* 设置缓存大小与临时目录：`new DiskFileItemFactory(20*1024, new File("F:/temp"));`

## 2. 实战演练

### 2.1 请求表单

```html
<form action="xxx" method="post" enctype="multipart/form-data">
  用户名:<input type="text" name="username"/><br/>
  照　片:<input type="file" name="zhaoPian"/><br/>
  <input type="submit" value="上传"/>
</form>
```

- 请求方式需要设置成POST；

- 文件表单类型为file；

- 增加属性enctype="multipart/form-data"。

*注：request.getParametere("xxx");这个方法在表单为enctype="multipart/form-data"时，它作废了。它永远都返回null ，ServletInputStream request.getInputStream();包含整个请求的体！*

### 2.2 后台处理

```java
@Controller
@RequestMapping("/fileupload")
public class UpLoadController {
  		
       @RequestMapping("/upload")
       public void upload(HttpServletRequest request, HttpServletResponse response)
                     throws ServletException, IOException {
              response.setContentType("text/html;chatset=utf-8");
              
              /*
               * 上传三大步
               */
              DiskFileItemFactory factory=new DiskFileItemFactory();
              ServletFileUpload sfu=new ServletFileUpload(factory);
              try {
                     List<FileItem> fList = sfu.parseRequest(request);
                     
                     //得到文件表单项
                     FileItem fileItem = fList.get(1);
                     
                     // 得到文件保存根路径
                     String root=getServletContext().getRealPath("/upload/files/");
                     
                     //得到文件名字
                     String fName = fileItem.getName();
                     //处理文件名绝对路径的问题
                     int lastIndexOf = fName.lastIndexOf("\\");
                     if(lastIndexOf!=-1) fName=fName.substring(lastIndexOf+1);
                     //处理相同文件名字的问题
                     String saveName=CommonUtils.uuid()+"_"+fName;

                     //得到hashCode
                     int hCode = fName.hashCode();
                     String hex = Integer.toHexString(hCode);
                     //生成完整的抽象路径名
                     File dirFile = new File(root,hex.charAt(0)+"/"+hex.charAt(1));
                     //生成抽象路径名指定的目录
                     dirFile.mkdirs();
                     
                     //创建完整文件目录
                     File destFile = new File(dirFile,saveName);
                     //输出,保存文件
                     try {
                           fileItem.write(destFile);
                     } catch (Exception e) {
                           throw new RuntimeException(e);
                     }
                     
              } catch (FileUploadException e) {
                     throw new RuntimeException(e);
              }
       }
}
```

## 2.3 总结

![源码大致流程](https://raw.githubusercontent.com/zchdjb/zchdjb.github.io/master/images/fileupload.png)

