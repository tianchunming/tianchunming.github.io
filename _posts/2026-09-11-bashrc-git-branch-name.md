---
layout:     post
title:      ".bashrc配置文件添加显示git分支名功能"
subtitle:   ""
date:       2026-09-11 16:30:00
author:     "Tiancm"
catalog: true
tags:
    - Git
    - Tool
---

## 一、环境准备
#### 1.文件详解   
 .bashrc 是 Bash Shell 的用户级配置文件，通常位于：    

    ~/.bashrc

也就是：    
    
    /home/你的用户名/.bashrc
    
它主要用于配置交互式非登录 Shell 的环境。每次你打开一个新的终端窗口，或者新开一个 Bash 会话时，通常会读取这个文件。

#### 2.常见用途
**设置环境变量**

    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    export ANDROID_HOME=$HOME/Android/Sdk
    export PATH=$PATH:$ANDROID_HOME/platform-tools

**设置命令别名**

    alias ll='ls -alF'
    alias gs='git status'
    alias gpf='git push --force-with-lease'

**定义函数**

    pk8_to_pem() {
        openssl pkcs8 -inform DER -nocrypt -in "$1" -out "${1%.pk8}.pem"
    }

之后就可以直接：

    pk8_to_pem platform.pk8

**修改提示符**
    
    export PS1='\u@\h:\w\$ '

**自动加载其他配置**

    source ~/.bash_aliases

#### 3.修改后如何生效
**修改 ~/.bashrc 后，不会自动影响当前已经打开的终端。需要执行：**

    source ~/.bashrc

或者：

    . ~/.bashrc

也可以直接关闭终端，重新打开一个。

## 二、添加显示git分支名功能

在.bashrc配置文件首行添加如下代码：

    ## Parses out the branch name from .gitHEAD
    find_git_branch () {
        local dir=. head
        until [ $dir -ef  ]; do
            if [ -f $dir.gitHEAD ]; then
                head=$( $dir.gitHEAD)
                if [[ $head = ref refsheads ]]; then
                    git_branch=${head#}
                elif [[ $head != '' ]]; then
                    git_branch=(detached)
                else
                    git_branch=(unknow)
                fi
                return
            fi
            dir=..$dir
        done
        git_branch=''
    }
    PROMPT_COMMAND=find_git_branch; $PROMPT_COMMAND
    # Here is bash color codes you can use
    black=$'[e[1;30m]'
    red=$'[e[1;31m]'
    green=$'[e[1;32m]'
    yellow=$'[e[1;33m]'
    blue=$'[e[1;34m]'
    magenta=$'[e[1;35m]'
    cyan=$'[e[1;36m]'
    white=$'[e[1;37m]'
    normal=$'[e[m]'
    PS1=$white[$magentau$white@$greenh$white$cyanw$yellow$git_branch$white]$ $normal
