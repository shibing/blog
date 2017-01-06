---
title: 使用vagrant和nginx开发静态文件修改不生效问题
date: 2017-01-06 11:14:54
tags: 
      Tips
      Nginx
---

# 现象
最近使用 Vagrant 开发一个网站，里面跑的是 Nginx 服务器，一切工作都很正常，除了 CSS、JS 等静态文件的修改不能实时生效之外。当修改一个静态文件后，查看响应头：

```
HTTP/1.1 200 OK
Server: nginx/1.10.2
Date: Fri, 06 Jan 2017 03:20:44 GMT
Content-Type: text/css
Content-Length: 3217
Last-Modified: Fri, 06 Jan 2017 03:20:42 GMT
Connection: keep-alive
ETag: "586f0d0a-c91"
Accept-Ranges: bytes
```
通过 `Last-Modified` 以及 `ETag` 字段可以看到 Nginx 确实探测到了修改，但是响应内容却依然是以前的，很是奇怪。

# 原因
后来通过 stackoverflow 找到了这个问题的原因，我的 Vagrant 底层用的是 VirtualBox, 而 vboxvfs 在使用 mmap 时会有一些问题，具体表现为在虚拟机之外修改文件后，虚拟机内以 mmap 方式打开文件的不能同步修改。而 Nginx 在 sendfile 选项开启后会通过 mmap 来加速文件的访问，自然就会有上面说的这个问题。

# 解决办法
知道问题原因后，解决办法就很自然了，只要将 nginx 的 sendfile 选项关闭即可。

```
sendfile off;
```

