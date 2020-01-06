---
title: CSI摄像头推流
date: 2018-11-28 20:16:08
img: http://q3996b08i.bkt.clouddn.com/embedded-study/raspberrypi2.jpg
top: false
toc: true
summary: 树莓派推流CSI摄像头推流
categories: 记录
tags: 
    - Raspberry Pi
---

[mjpg-streamer Github](https://github.com/jacksonliam/mjpg-streamer)

<kbd>**用到的工具材料**：</kbd>
 - [x] 树莓派3B+
 - [x] 网线
 - [x] 电脑
 - [x] CSI摄像头
 - [x] USB摄像头

> 实现的功能有：
> ①树莓派通过网线将CSI摄像头与USB摄像头的实时画面推流至上位机电脑中的显示
> ②实测双摄延时低至200ms以下

我的另一篇博文中介绍了使用树莓派H.264硬件编解码推流CSI摄像头，所以本文仅介绍如何用 MJPG-Streamer推流USB摄像头实时画面。所以需要同时推流双摄时，可选择CSI+USB Camera或者双USB Camera。

<table>
    <tr>
        <td ><center><img src="https://img-blog.csdnimg.cn/20181128153213623.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70" >图1  树莓派接线整体图 </center></td>
        <td ><center><img src="https://img-blog.csdnimg.cn/20181128154230707.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70"  >图2 树莓派接线整体图</center></td>
</table>


# 总体流程
1.树莓派硬件连接与软件及驱动安装
2.上位机PC端的接收视频流
3.延迟效果测试

## 1.硬件连接与软件及驱动配置

###  1）检测是否存在USB摄像头设备

输入以下指令：
```
pi@raspberrypi:~ $ lsusb
```
- 未插入USB摄像头

```

pi@raspberrypi:~ $ lsusb 
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

```

- 插入USB摄像头

```
pi@raspberrypi:~ $ lsusb
Bus 001 Device 004: ID 05a3:9230 ARC International
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

```
pi@raspberrypi:~ $ v4l2-ctl --list-formats
```

```
pi@raspberrypi:~ $ v4l2-ctl --list-formats

ioctl: VIDIOC_ENUM_FMT
        Index       : 0
        Type        : Video Capture
        Pixel Format: 'MJPG' (compressed)
        Name        : Motion-JPEG

        Index       : 1
        Type        : Video Capture
        Pixel Format: 'YUYV'
        Name        : YUYV 4:2:2

```


<table>
    <tr>
        <td ><center><img src="https://img-blog.csdnimg.cn/20181202170523751.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70" >图1  还未插入USB摄像头 </center></td>
        <td ><center><img src="https://img-blog.csdnimg.cn/20181202170531278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70"  >图2 插入USB摄像头</center></td>
</table>


### 2）安装 MJPG-Streamer from github

> 按照以下命令安装MJPG-Streamer及其相关配置 每一行为一个命令

```
sudo apt-get install cmake libjpeg8-dev

wget https://github.com/jacksonliam/mjpg-streamer/archive/master.zip

unzip master.zip

cd mjp*g-*

cd mjpg-*

make

sudo make install

cd $home
```

流程如下
![](https://img-blog.csdnimg.cn/2018120219295880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70)

### 3）启动MJPG-Streamer

> 启动如下指令：
TCP端口8085被指定为输出端口，以确保没有对其他任何东西的干扰。

```
/usr/local/bin/mjpg_streamer -i "/usr/local/lib/mjpg-streamer/input_uvc.so -n -f 10 -r 1280x720" -o "/usr/local/lib/mjpg-streamer/output_http.so -p 8085 -w /usr/local/share/mjpg-streamer/www"
```

![树莓派推流成功](https://img-blog.csdnimg.cn/20181202185954705.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70)

**如上图所示，推流已经完成，分辨率为720P(1280X720)，端口号为8085，到此树莓派的任务已经完成**
```
pi@raspberrypi:~ $ /usr/local/bin/mjpg_streamer -i "/usr/local/lib/mjpg-streamer/input_uvc.so -n -f 10 -r 1280x720" -o "/usr/local/lib/mjpg-streamer/output_http.so -p 8085 -w /usr/local/share/mjpg-streamer/www"
MJPG Streamer Version.: 2.0
 i: Using V4L2 device.: /dev/video0
 i: Desired Resolution: 1280 x 720
 i: Frames Per Second.: 10
 i: Format............: JPEG
 i: TV-Norm...........: DEFAULT
 i: FPS coerced ......: from 10 to 60
 o: www-folder-path......: /usr/local/share/mjpg-streamer/www/
 o: HTTP TCP port........: 8085
 o: HTTP Listen Address..: (null)
 o: username:password....: disabled
 o: commands.............: enabled

```
### 4）开启指定/多个摄像头
- USB设备存在两个摄像头

![](https://img-blog.csdnimg.cn/20190507154817290.png)
- 如果你有多个摄像头，也可以开启多个摄像头
- -d /dev/video1    参数标明需要开启的摄像头

```
/usr/local/bin/mjpg_streamer -i "/usr/local/lib/mjpg-streamer/input_uvc.so -d /dev/video1 -n -f 10 -r 1280x720" -o "/usr/local/lib/mjpg-streamer/output_http.so -p 8085 -w /usr/local/share/mjpg-streamer/www"
```

## 2.上位机PC端的接收视频流

> 采用web网页接收视频流

打开浏览器输入：
```
http://树莓派ip地址:开启的端口号
```
```
// A case in point 例如：
http://192.168.2.150:8085
```

> 输入 http://192.168.2.150:8085   打开后如下图：为MJPG Streamer 首页
![](https://img-blog.csdnimg.cn/20181202190911467.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70)


- 查看视频点击Stream

![](https://img-blog.csdnimg.cn/20181202190927946.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70)

看到网页中有树莓派USB摄像头拍摄的画面，证明视频推流成功。
## 3.双摄延迟效果测试        
        
> 我们打开一个在线秒表，让其开始计时，而后我们用树莓派CSI摄像头及USB摄像头去拍摄电脑屏幕，然后截屏，计算拍摄的时间与秒表实际时间的差值，可以粗略测得延时的结果

![](https://img-blog.csdnimg.cn/20181202191338911.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70)




"如上图所示"    在线秒表为1.18.697  
USB摄像头（Web MJPG-Streamer）推流视频中的在线秒表为1.18.567  
CSI摄像头（MPlayer  视频播放器）推流视频中的在线秒表为1.18.525
其差值△t1=0.130s=130ms                  △t1=0.172s=172ms


之后我通过多次测试多种组合得到如下图数据（仅做参考）：

![](https://img-blog.csdnimg.cn/20181202192130874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDkyOTMy,size_16,color_FFFFFF,t_70)




总结：
```
   分辨率都为 720P  延迟大致为100-200ms之间[带光纤延迟增加并不明显]
   分辨率都为 1080P 延迟为200-300ms
```