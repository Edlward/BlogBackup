---
title: 利用七牛云+PicGo进行图片管理与上传
date: 2019-12-15 19:19:23
img: http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_picgo_img1.jpg
top: false
toc: true
summary: 白嫖七牛云作为免费博客图床
categories: 记录
tags: 
	- 效率工具
---



## 1.注册七牛云账号
- [点击注册七牛云](https://portal.qiniu.com/signup)

![注册七牛云](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_signup.png)
> 需要**实名认证**，认证之后再进行上传图片
> 点击进入“个人中心”-> “个人信息”，进行实名认证，即有10G免费存储空间


## 2.创建标准存储空间
![点击进入控制台](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_controller.png)
### 2.1 添加对象存储

![点击对象存储](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_store.png)

### 2.2 设置存储空间


存储空间名称：自行命名
存储区域：选与你地域较近的区域
访问控制：选“公开空间”，否则外网无法访问
![存储空间设置](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_store_setting.png)

### 2.3 上传图片

![上传图片](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_upload.png)
### 2.4 获取图片外链

![获取图片外链](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_copy_url.png)

## 3.使用PicGo进行管理与上传
![PicGo](http://q3996b08i.bkt.clouddn.com/embedded-study/picgo.png)

- [点击下载PicGo](https://github.com/Molunerfinn/PicGo/releases)


### 3.1 设置七牛图床

![设置七牛图床](http://q3996b08i.bkt.clouddn.com/embedded-study/picgo_qiniu_setting.png)



-  1.获得AK与SK

![](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_ak_sk.png)
- 2.存储空间名要与之前创建的空间名移植
- 3.获得访问网址

![访问网址](http://q3996b08i.bkt.clouddn.com/embedded-study/qiniu_copy_web.png)


### 3.2 复制外链

上传完的图片就会在相册中，并且预览、复制链接

![](http://q3996b08i.bkt.clouddn.com/embedded-study/picgo_photos.png)

### 3.3 自定义链接格式

- `Custom`模式可以自定义链接格式

![](http://q3996b08i.bkt.clouddn.com/embedded-study/picgo_set_url.png)

-用占位符`$url`来表示ul的位置
- 用占位符`$fileName`来表示文件名的位置

![自定义链接格式](http://q3996b08i.bkt.clouddn.com/embedded-study/picgo_set_by_custom.png)

点击复制，就会按照自定义的格式复制链接，eg：
```
![在这里描述图片](http://q3996b08i.bkt.clouddn.com/embedded-study/lcdcon1.png)
```


### 3.4 PicGo插件
- [点击进入PicGo插件合集页面](https://github.com/PicGo/Awesome-PicGo)

---

- [PicGo -plugin-autocopy](https://github.com/PicGo/picgo-plugin-autocopy):用于上传后将url自动复制到剪贴板的PicGo插件。【CLI】
- [PicGo -plugin-github-plus](https://github.com/zWingz/picgo-plugin-github-plus): PicGo上传github & gitee同步功能。【GUI】
- [PicGo -plugin- pico -migrater](https://github.com/picgo/picgo-plugin-pico-migrater) :一个PicGo插件，用于将markdown文件中的图片从一个picBed迁移到另一个picBed。【CLI&GUI】
- [PicGo -plugin-web-uploader](https://github.com/yuki-xin/picgo-plugin-web-uploader): PicGo上传自定义web api。【GUI】
- [PicGo -plugin-qingstor-uploader](https://github.com/chengww5217/picgo-plugin-qingstor-uploader): PicGo插件与添加的青石图片托管。【GUI】
- [PicGo -plugin-vscode-migrator](https://github.com/upupming/picgo-plugin-vscode-migrator): PicGo从[' vs-picgo '](https://github.com/Spades-S/vs-picgo/)导入图像的插件【GUI】
- [PicGo -plugin-super-prefix](https://github.com/gclove/picgo-plugin-super-prefix#readme): PicGo自定义图像文件名和前缀的插件【CLI&GUI】
- [PicGo -plugin-smms-user](https://github.com/xlzy520/picgo-plugin-smms-user.git):针对SM注册用户的PicGo插件。MS【GUI】
- [PicGo -plugin- GitLab](https://github.com/bugwz/picgo-plugin-gitlab): PicGo GitLab上传器【CLI&GUI】
- [PicGo -plugin-bilibili](https://www.npmjs.com/package/picgo-plugin-bilibili): PicGo上传bilibli【CLI&GUI】- [PicGo -plugin-gitee](https://github.com/zhanghuid/picgo-plugin-gitee): PicGo gitee上传器【CLI&GUI】
- [PicGo -plugin-gitee-uploader](https://github.com/lizhuangs/picgo-plugin-gitee-uploader#readme)
- [PicGo -plugin- NextCloud - Uploader](https://github.com/jiajiajia343434/picgo-plugin-nextcloud-uploader): PicGo - Uploader for NextCloud【CLI&GUI】
- [PicGo -plugin-watermark](https://github.com/Dec-F/picgo-plugin-watermark): PicGo的水印插件【CLI&GUI】【v2.2.0+】
- [PicGo -plugin-quick-capture](https://github.com/PicGo/picgo-plugin-quick-capture): PicGo【CLI&GUI】【v2.2.0+】快速截图及上传插件
