---
layout:     post
title:      "安全许可名单（package-shareduid-allowlist.xml）"
subtitle:   ""
date:       2026-09-14 10:00:00
author:     "Tiancm"
catalog: true
tags:
    - Framwork
---

**package-shareduid-allowlist.xml** 是 **Android 15** 引入的一个安全许可名单文件，用于严格控制平台签名的非系统应用加入平台签名共享 UID 的行为。它是对 framework-sysconfig.xml 中提到的共享 UID 许可机制的具体实现文件。

## 一、核心目的：收紧共享 UID 的控制权

在 Android 15 之前，设备制造商几乎无法控制哪些平台签名的非系统应用（即使用平台签名但未预装在系统分区中的应用）可以在其清单中声明 android:sharedUserId 并加入平台共享 UID（如 android.uid.system）。这存在安全风险，因为加入该 UID 意味着应用可以获得接近系统级别的权限。

从 Android 15 开始，Google 引入了这一许可名单机制。如果平台签名的非系统应用未在此名单中，并且仍尝试加入平台签名共享 UID，则该应用将无法安装在非调试（user）版本上。这有效防止了通过侧载方式滥用平台签名权限。

## 二、文件位置与格式

该文件的 AOSP 源码位于 **frameworks/base/data/etc/package-shareduid-allowlist.xml**，编译后会被安装到设备的 **system/etc/permissions** 或 **system/etc/sysconfig** 目录下。可以在单个文件中列出所有条目，也可以分散在多个 XML 文件中。

#### 1.XML 结构示例

文件的核心是 <allow-package-shareduid> 标签，它包含两个属性：

    <config>
        <!-- package: 应用的清单包名 -->
        <!-- shareduid: 允许加入的共享 UID 名称 -->
        <allow-package-shareduid 
            package="android.test.settings" 
            shareduid="android.uid.system" />
        
        <allow-package-shareduid 
            package="oem.example.app" 
            shareduid="oem.uid.custom" />
    </config>

* package：目标应用的包名。
* shareduid：允许该应用加入的共享 UID 名称（例如 android.uid.system 或自定义的 OEM UID）。

#### 2.工作机制与安装失败排查

当系统在非调试版本上安装一个平台签名的非系统应用时，会检查其清单中的 sharedUserId：

* 如果该应用未被列入许可名单，安装将直接失败。
* 系统日志中会输出一条明确的警告信息，格式如下，便于开发者定位问题：   

      Non-preload app {PACKAGE_NAME} signed with platform signature and joining shared uid: {SHARED_UID_NAME}   

   例如，日志中可能显示类似：
         
      allowList: {android.test.settings=android.uid.system, com.ganxinkj.haliboboluancher=android.uid.system}    

    的信息，这直接展示了当前生效的许可名单内容。

## 三、与相关配置文件的关系

| 配置文件 | 核心职责 | 管控对象 |
| :----: | :----: | :----: |
| package-shareduid-allowlist.xml | 权限准入：控制哪些应用可以使用哪个共享 UID | 平台签名的非系统应用 |
| framework-sysconfig.xml        | 系统行为豁免：定义后台广播、备份通道等系统级白名单 | 系统框架行为 |
| preinstalled-packages-platform.xml        | 预安装范围：控制应用在何种用户类型下安装 | 所有系统应用的安装范围 |

简而言之，package-shareduid-allowlist.xml 关注的是“权限安全”，而另外两个文件分别关注“系统行为”和“安装策略”。

## 四、实践建议

* 严格遵守最小权限原则：仅将确实需要平台级共享 UID 的应用加入名单，避免为图方便而随意添加。
* 优先改造应用：如果非系统应用需要特定系统能力，优先考虑使用系统权限（System Permissions） 或系统服务来授予，而非加入共享 UID。
* 调试版本不受影响：在 userdebug 或 eng 版本上，该限制不会生效，方便开发调试。
* 为 OEM 自定义 UID 使用：除了 android.uid.system，你也可以为 OEM 自定义的共享 UID（如 oem.uid.custom）创建相应的许可名单条目。