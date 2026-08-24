---
layout:     post
title:      "（Mediatek）使用platform.pk8和platform.x509.pem文件生成platform.jks签名文件"
subtitle:   ""
date:       2026-09-11 15:46:00
author:     "Tiancm"
catalog: true
tags:
    - Framwork
    - Mediatek
    - Tool
---

使用 **platform.pk8** 和 **platform.x509.pem** 生成 **platform.jks** 文件的过程涉及多个步骤，主要使用包括 OpenSSL 和 Keytool 工具进行格式转换和密钥导入。

## 一、环境准备
#### 1.文件详解   
 - **\*\*.pk8（私钥文件）**：这是私钥，采用 DER 编码的 PKCS#8 格式，是二进制文件，不可读，它主要用于对 APK 进行签名，证明该应用的身份  
 - **\*\*.x509.pem（公钥证书文件）**：这是公钥证书，采用 PEM 格式（Base64 编码的文本文件，以 -----BEGIN CERTIFICATE----- 开头），它主要用于验证签名的有效性，确保应用未被篡改且来源可信
 - 这两个文件都是成对出现的

#### 2.在 MTK 平台上的存放位置
**在 MTK 平台的源码中，这两个文件的存放路径取决于项目的具体配置，主要有以下两种可能：**
- AOSP 标准路径：alps/build/target/product/security/
- MTK 定制路径：alps/device/mediatek/common/security/ 或项目专属的 device/mediatek/security/ 目录   
![img](/img/in-post/post_key_directory.png)     

具体使用哪个路径，通常由源码中的 MTK_SIGNATURE_CUSTOMIZATION 这个宏（yes/no）来控制。

#### 3.核心用途：为 APK 赋予系统权限
这两个文件最核心的用途是对 APK 进行“平台签名”。一个应用如果需要获得系统级权限（例如，在 AndroidManifest.xml 中声明了 **android:sharedUserId="android.uid.system**"），就必须使用这对密钥进行签名。

**签名后的应用可以：**
- 访问受保护的系统 API。
- 实现开机自启动、后台启动 Activity 等高级功能。
- 与使用相同平台密钥签名的其他应用共享数据和进程。

## 二、使用OpenSSL（Windows系统可以使用Git bash）工具进行格式转换：

首先，需要将 **platform.pk8** 和 **platform.x509.pem** 文件转换为 **PKCS#8** 格式的PEM文件（platform.priv.pem）和 PKCS#12 格式的 PFX 文件（platform.p12）。可以通过使用OpenSSL命令行工具完成，具体命令如下：

#### 1.将 platform.pk8 转换为 PEM 格式：  

    openssl pkcs8 -in platform.pk8 -inform DER -outform PEM -out platform.priv.pem -nocrypt

**参数解释：**
- openssl pkcs8：使用 OpenSSL 的 PKCS#8 私钥处理功能
- -in platform.pk8：指定输入文件为 platform.pk8
- -inform DER：告诉 OpenSSL，输入文件是 DER 二进制格式
- -outform PEM：指定输出格式为 PEM 文本格式
- -out platform.priv.pem：指定输出文件为 platform.priv.pem
- -nocrypt：表示输出的私钥不加密，也就是不设置密码

执行这个命令后，会得到一个名为 platform.priv.pem 的未加密 **PEM** 私钥文件，其中包含了从 **platform.pk8** 文件中提取并转换为 **PEM** 编码的私钥。这个 PEM 编码的私钥可以用于需要私钥的任何应用程序或服务，例如 SSL/TLS  通信、代码签名等。

#### 2.使用 platform.priv.pem 和 platform.x509.pem 合并公私钥生成 PKCS#12 格式的 PFX 文件（platform.p12）：    

    openssl pkcs12 -export -in platform.x509.pem -inkey platform.priv.pem -out platform.p12 -name platform_alias -password pass:keyPassword

**参数解释：**
- openssl pkcs12：使用 OpenSSL 的 PKCS#12 工具，用来处理 .p12 / .pfx 文件
- -export：表示导出 PKCS#12 文件，也就是把证书和私钥打包
- -in platform.x509.pem：输入证书文件，这里是 platform.x509.pem
- -inkey platform.priv.pem：输入私钥文件，这里是上一步生成的 platform.priv.pem
- -out platform.p12：输出 PKCS#12 文件，文件名为 platform.p12
- -name platform_alias：给这个证书/私钥条目设置一个别名，别名是 platform_alias
- -password pass:keyPassword：直接指定密码为 keyPassword，而不是交互式输入

**执行结果，会生成一个 platform.p12 文件。这个文件里面同时包含：**
- 证书：platform.x509.pem
- 私钥：platform.priv.pem
- 别名：platform_alias
- 密码：keyPassword

它通常用于 **Android** 签名、**Java keystore**、**jarsigner**、**apksigner** 等场景。

**验证生成的 platform.p12，可以用 OpenSSL 查看：**

    openssl pkcs12 -info -in platform.p12 -passin pass:keyPassword -noout

**或者用 keytool 查看：**

    keytool -list -v -keystore platform.p12 -storetype PKCS12 -storepass keyPassword

#### 3.使用 keytool 将 PKCS12 转换为 JKS (需要安装 JDK)

    keytool -importkeystore -deststorepass keyPassword -destkeypass keyPassword -destkeystore platform.jks -deststoretype JKS -srckeystore platform.p12 -srcstoretype PKCS12 -alias platform_alias -srcstorepass keyPassword

**参数解释：**

- keytool -importkeystore：导入密钥库，把一个密钥库转换成另一个密钥库
- -deststorepass keyPassword：目标密钥库 platform.jks 的密码是 keyPassword
- -destkeypass keyPassword：目标密钥库中私钥条目的密码是 keyPassword。这个参数主要用于 JKS 这类支持单独密钥密码的格式
- -destkeystore platform.jks：输出目标文件为 platform.jks
- -srckeystore platform.p12：源密钥库文件是 platform.p12
- -srcstoretype PKCS12：源密钥库类型是 PKCS12
- -alias platform_alias：指定要导入的源条目别名是 platform_alias
- -srcstorepass keyPassword：源密钥库密码是 keyPassword

执行这个命令后，会得到一个名为 **platform.jks** 的 Java 密钥库文件，其中包含了从 **platform.p12** 密钥库中导入的密钥和证书。可以将这个 Java 密钥库文件用于需要证书和私钥的任何 Java 应用程序或服务。    

**如果想修改目标别名，可以加：**    
     
    -destalias platform_alias

**执行成功后，可以验证：**


    keytool -list -v -keystore platform.jks -storetype JKS -storepass keyPassword

如果能看到别名 platform_alias，并且包含私钥和证书链，就说明转换成功。

## 三、在Android Studio 的build.gradle文件中进行配置    

    signingConfigs {
            main {
                storeFile file('./platform.jks')
                storePassword "keyPassword"
                keyAlias "platform_alias"
                keyPassword "keyPassword"
            }
        }
    }