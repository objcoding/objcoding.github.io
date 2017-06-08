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

**Maven依赖：**

```xml
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.3.2</version>
</dependency>
```

**相关类：**

- 工厂：DiskFileItemFactory
- 解析器：ServletFileUpload
- 表单项：FileItem

**上传三大步：**

1.  创建工厂：`DiskFileItemFactory factory = new DiskFileItemFactory();`
2. 创建解析器：`ServletFileUpload sfu = new ServletFileUpload(factory);`
3. 使用解析器解析request：`List<FileItem> fileItemList = sfu.parseRequest(request);`

**FileItem API：**

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

**需要注意的一些上传细节：**

1. 上传到服务器的地址
2. ​



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

