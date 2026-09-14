---
layout:     post
title:      "预安装系统软件包的许可名单（preinstalled-packages-platform.xml）"
subtitle:   ""
date:       2026-09-14 09:50:00
author:     "Tiancm"
catalog: true
tags:
    - Framwork
---

preinstalled-packages-platform 是 Android 系统中一个预安装系统软件包的许可名单（Allowlist）机制。它允许设备制造商（OEM）或开发者根据用户类型，精准控制哪些系统应用应该在设备初次创建用户时被安装，其核心作用是解决 Android 多用户场景下的应用安装问题：并非所有系统应用都适合所有用户（例如，拨号器不应出现在儿童模式，壁纸应用对工作资料无意义）。通过该文件 OEM 可以精细化控制每个应用的安装范围，从而优化存储空间、启动时间和内存占用。

## 一、核心机制：按用户类型安装

Android 支持多用户，但并非所有系统应用都适合所有用户。该机制通过定义不同的用户类型来决定应用的安装范围。

* SYSTEM：User 0，即系统主用户。
* FULL：没有配置个人资料的真实用户。
* PROFILE：带有配置文件的真实用户（如工作资料）。

此外，系统还支持更精细的 AOSP 用户类型，例如 android.os.usertype.full.SECONDARY（次要用户）或 android.os.usertype.profile.MANAGED（托管配置文件）。

## 二、配置方式：XML 文件

配置主要通过位于 frameworks/base/data/etc/preinstalled-packages-platform.xml 的 XML 文件进行，在系统编译时会被安装到设备的 /system/etc/sysconfig/ 目录下，在实际产品中，会基于此文件派生出多个针对特定产品形态的配置文件，例如：

* preinstalled-packages-platform-telephony-product.xml：针对电话产品。
* preinstalled-packages-platform-handheld-product.xml：针对手持设备产品。
* preinstalled-packages-platform-aosp-product.xml：针对 AOSP 产品。

### 文件结构与核心元素

文件采用简单的 XML 结构，核心元素是 \<install-in-user-type> 和\<do-not-install-in>。

#### 1.基本结构

    <install-in-user-type package="com.android.example">
        <install-in user-type="SYSTEM" />
        <install-in user-type="FULL" />
        <do-not-install-in user-type="android.os.usertype.profile.CLONE" />
    </install-in-user-type>

* package 属性：指定目标应用的清单包名（Manifest Package Name），这是系统识别应用的唯一键
* \<install-in user-type="..." />：声明该应用应该在哪些用户类型上安装
* \<do-not-install-in user-type="..." />：声明该应用不应该在哪些用户类型上安装，用于排除特定场景

#### 2.用户类型体系

用户类型分为基础类型和具体类型两个层级

| 层级 | 类型 | 说明 |
| :----: | :----: | :----: |
| 基础类型 | SYSTEM | User 0，系统主用户 |
|         | FULL | 任何非个人资料的用户 |
|         | PROFILE | 个人资料用户 |
| 具体类型 | android.os.usertype.full.SECONDARY | 次要用户 |
|         | android.os.usertype.full.GUEST | 访客用户 |
|         | android.os.usertype.full.DEMO | 演示用户 |
|         | android.os.usertype.full.RESTRICTED | 受限用户 |
|         | android.os.usertype.profile.MANAGED | 托管工作资料 |
|         | android.os.usertype.profile.PRIVATE | 私人资料 |
|         | android.os.usertype.system.HEADLESS | 无头系统用户 |

每个用户都确切地属于一种具体用户类型，因此使用具体类型可以实现更精细的控制。

### 常见配置示例
#### 1.仅安装在系统主用户 (User 0) 上

    <install-in-user-type package="com.android.example">
        <install-in user-type="SYSTEM" />
    </install-in-user-type>

#### 2.安装在所有真实用户上（如浏览器）

    <install-in-user-type package="com.android.example">
        <install-in user-type="FULL" />
        <install-in user-type="PROFILE" />
    </install-in-user-type>

#### 3.安装在所有人类用户，但排除个人资料用户（如壁纸应用）

    <install-in-user-type package="com.android.example">
        <install-in user-type="FULL" />
    </install-in-user-type>

#### 4.排除特定用户类型（如不安装在克隆配置文件中）

    <install-in-user-type package="com.android.dialer">
        <install-in user-type="SYSTEM" />
        <install-in user-type="FULL" />
        <install-in user-type="PROFILE" />
        <do-not-install-in user-type="android.os.usertype.profile.CLONE" />
    </install-in-user-type>

#### 5.无论类型，为所有用户安装（如核心系统组件）

    <install-in-user-type package="com.android.example">
        <install-in user-type="SYSTEM" />
        <install-in user-type="FULL" />
        <install-in user-type="PROFILE" />
    </install-in-user-type>


## 三、相关文件

除了 preinstalled-packages-platform.xml 主文件，系统还包含其他相关的配置文件：

* framework-sysconfig.xml
* preinstalled-packages-platform-overlays.xml
* initial-package-stopped-states.xml

这些文件共同构成了 Android 系统软件包预安装和管理的完整配置体系。