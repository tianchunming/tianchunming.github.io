---
layout:     post
title:      "替换开机动画bootanimation"
subtitle:   ""
date:       2026-09-12 11:40:00
author:     "Tiancm"
catalog: true
tags:
    - Framwork
---
替换 Android 开机动画，核心是制作一个符合规范的 bootanimation.zip 文件，并将其集成到系统镜像中。整个过程可以拆解为制作动画文件、了解系统加载路径，以及通过源码编译完成替换。

## 一、制作 bootanimation.zip

开机动画本质上是一个 ZIP 压缩包，其内部结构有严格规定，必须使用 **仅存储** 模式压缩，否则系统无法读取。

#### 1.文件结构   
 ZIP 包解压后的顶层必须包含一个 desc.txt 文件和若干以 part 开头的文件夹（如 part0, part1）。每个 part 文件夹内存放该动画片段的一系列 PNG 帧图片。   

    bootanimation.zip
    ├── desc.txt
    ├── part0/
    │   ├── 001.png
    │   ├── 002.png
    │   └── ...
    └── part1/
        ├── 001.png
        └── ...

**建议：图片文件名采用补零命名（如 001.png），以确保系统按正确的数字顺序播放。**

#### 2.desc.txt 格式详解
desc.txt 是动画的“剧本”，其格式如下：

    1080 1920 30
    p 1 0 part0
    c 0 0 part1

这表示一个 1080x1920 分辨率、30帧/秒的动画。先播放一次 part0，然后无限循环播放 part1，且 part1 会完整播放直到开机结束。

**第一行：定义全局参数**

WIDTH HEIGHT FPS [PROGRESS]

* WIDTH, HEIGHT：动画的宽和高（像素）
* FPS：每秒帧数。
* PROGRESS：（可选）是否显示进度百分比（Android 12+）。

**后续行：定义每个动画片段，每行对应一个 part 文件夹**

TYPE COUNT PAUSE PATH [FADE ...]

* TYPE：p 表示播完即止（开机完成会中断），c 表示完整播完（不受开机完成影响）
* COUNT：0 表示无限循环，其他数字表示循环次数
* PAUSE：片段结束后暂停的帧数
* PATH：对应的文件夹名，如 part0

#### 3.打包命令

在包含 desc.txt 和 part 文件夹的目录下，执行以下命令生成 ZIP（-0 或 -Z store 确保无压缩）：

    zip -0qry -i \*.txt \*.png \*.wav @ ../bootanimation.zip *.txt part*

或者：

    zip -r -X -Z store bootanimation part*/* desc.txt

## 二、了解系统的加载路径与优先级

系统会按优先级从多个路径查找 bootanimation.zip，找到第一个可用文件后就停止搜索。优先级从高到低为：

* /data/local/bootanimation.zip：这是调试专用路径，优先级最高，非常适合在不刷机的情况下快速验证动画效果。
* /apex/com.android.bootanimation/etc/bootanimation.zip (Android 10+)
* /product/media/bootanimation.zip (Android 9+)
* /oem/media/bootanimation.zip
* /system/media/bootanimation.zip：传统默认路径，大多数厂商的默认动画放在这里。

## 三、集成到 AOSP 源码（编译方式）
若要将动画固化到系统镜像，需通过 AOSP 的编译系统进行集成。

#### 1.放置文件

将制作好的 bootanimation.zip 放入设备源码目录，例如:
    
    device/mediatek/common/mid/system/media/

#### 2.修改 Makefile

在该设备的 device.mk 或类似编译配置文件中（device/mediatek/system/common/device.mk），添加 PRODUCT_COPY_FILES 规则，将文件拷贝到系统分区。

    PRODUCT_COPY_FILES += \
        device/mediatek/common/mid/system/media/bootanimation.zip:system/media/bootanimation.zip

或者：
 
    $(shell mkdir -p out_sys/target/product/mssi_64_cn_armv82/system/media)
    $(shell cp device/mediatek/common/mid/system/media/bootanimation.zip out_sys/target/product/mssi_64_cn_armv82/system/media)

**注意：如果设备已有默认动画，请确保你的规则不会与原有规则冲突，或直接替换原有文件。**

#### 3.编译刷机

执行编译流程，刷入 system 镜像后生效。

## 四、调试与验证（ADB 方式）

重启后观察动画是否正常播放。如果异常，可通过以下命令排查：

* 查看 bootanim 进程是否运行： 

      adb shell "ps -A | grep bootanim"

* 过滤日志：

      adb shell "logcat | grep -i BootAnimation"

* 常见错误包括：Permission denied（文件权限不对）、Can't access part1/001.png（路径或文件名有误）
