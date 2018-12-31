---
layout: article
title:  "Android Real Time Capture"
categories: programing
tags: [android, communication]
toc: false
image:
    teaser: programing/2018-12-31-Android-Real-Time-Capture/teaser.jpg

date: 2018-12-31
---

本文将详细介绍Android实时抓包实现方案，抓包详情可在WireShark上实时展示，
以解决手机抓包需先保存成pcap文件再由Wireshark打开UI显示的问题，**注意：当前方案的前提条件是手机已被root**

## 1. 安装BusyBox

Android上的[BusyBox](https://play.google.com/store/apps/details?id=stericson.busybox&hl=en_US)提供了大多数linux的命令，主要是需要nc (netcat)工具来提供网络IO转发功能，本目录下已提供了BusyBox的apk

![01-busybox](/images/programing/2018-12-31-Android-Real-Time-Capture/01-busybox.png)

## 2. 安装netcat

[netcat](https://sourceforge.net/projects/netcat/)提供了网络转发功能，以便将Android手机上的抓包数据通过adb建立的转发通道实时转发到PC上作为wireshark的数据源，从而实现Android的实时抓包。

Windows如果使用的是cygwin，则可通过cygwin的安装包进行安装

![02-netcat](/images/programing/2018-12-31-Android-Real-Time-Capture/02-netcat.png)

Linux一般自带nc (netcat)工具，一般可通过apt-get或通过源码安装

```bash
# apt-get安装
apt-get install netcat
## 或者通过源码安装
wget ftp://heanet.dl.sourceforge.net/n/ne/netcat/netcat-0.7.1.tar.gz
tar zvxf netcat-0.7.1.tar.gz
./compile
make
make install
```


## 3. 安装tcpdump

[tcpdump](https://www.androidtcpdump.com/android-tcpdump/downloads)用于抓包，可运行`./cap_install.sh`脚本（见附件）来安装，注意同文件夹下需要有tcpdump执行文件。若提示安装结束，则表明安装成功，需要adb环境。windows上可能需要cygwin或gitBash等工具来执行linux脚本，当然也可以手动执行该脚本的指令进行安装

```bash
adb push tcpdump /mnt/sdcard/tcpdump
sleep 2s
adb shell "su -c 'cp /mnt/sdcard/tcpdump /data/local/tcpdump'"
adb shell "su -c 'rm /mnt/sdcard/tcpdump'"
adb shell "su -c 'chmod 777 /data/local/tcpdump'"
```

![03-tcpdump](/images/programing/2018-12-31-Android-Real-Time-Capture/03-tcpdump.png)

## 4. 安装Wireshark

[Wireshark](https://www.wireshark.org/download.html)可以对数据包进行UI分析，**注意：安装后还需将其执行文件放到Path环境变量中，最后的抓包脚本需要**

![04-wireshark](/images/programing/2018-12-31-Android-Real-Time-Capture/04-wireshark.png)

## 5. 进行抓包

将以下的指令保存成`cap_rt.sh`脚本并执行，便可在wireshark中看到实时的抓包情况，点击ctrl-c可以停止抓包，在wireshark中可以保存成pcapng文件

```bash
# 如果localhost的31337端口被占用，请更换其他端口，当然一般都不会
ncPort=31337
wiresharkPath=Wireshark
# 设置设备和PC的转发端口
adb forward tcp:31337 tcp:31337
# 显示设置结果
adb forward --list
# 因为通过127.0.0.1的地址与PC通信，所以抓包命令注意排除该地址，否则会产生巨大的递归流量
adb shell "su -c '/data/local/tcpdump -i any -s 0 host !127.0.0.1 -w - | nc -l -p '$ncPort''" &
sleep 1s
nc localhost $ncPort | $wiresharkPath -kS -i -
```



---
The End.

zhlinh

Email: zhlinhng@gmail.com

2018-12-27