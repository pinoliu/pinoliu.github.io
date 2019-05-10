---
title: Shell实用类-公共日志(common_log)
date: 2018-09-23 21:42:33
subtitle: shell_common_log
tags:
- shell
- commonlib
---

## 简介

本文核心为介绍 编写shell脚本中 常用到 公共日志类 的写法，并提供 简单的模板 可供大家直接使用。

## 用例

### 目录结构

```shell
## 目录结构
# opt
# -- demo_assert
# -- -- exec.sh
# -- -- dir
# -- -- -- common_log.sh
# -- -- -- log.sh
```

### 效果展示

```shell
# Liu18:/opt/demo_assert # sh demo.sh 
# Liu18:/opt/demo_assert # cd dir/
# Liu18:/opt/demo_assert/dir # cd logs/
# Liu18:/opt/demo_assert/dir/logs # ll
# total 8
# -rw-r----- 1 root root 218 Sep 23 21:22 exec.log
# -rw-r----- 1 root root  71 Sep 23 21:22 test_demo.log

# exec.log
# [2018-09-23 21:22:46][WARNING][exec.sh][######################################## exec.sh Bgein ########################################]
# [2018-09-23 21:22:46][INFO][main][Test demo.]
# [2018-09-23 21:22:46][WARNING][exec.sh][######################################## exec.sh End ########################################]

# test_demo.log
# [2018-09-23 21:22:46][INFO][main][Test function = log_touch_exec_file]
```

### 调测的代码

```shell
#!/bin/bash
# source /etc/profile
# source ./dir/common_log.sh
# 
# log_touch_file
# log_begin "exec.sh"
# log "INFO" "main" "Test demo."
# log_end "exec.sh"
#
# log_touch_exec_file "test_demo_log"
# log "INFO" "main" "Test function = log_touch_exec_file"
```

### 日志类代码

```shell
#!/bin/bash

# 确保系统环境变量存在
source /etc/profile

## 简介：
# $0 = 执行脚本的文件名（exec.sh）
# $BASH_SOURCE = 源码相对执行文件（exec.sh）的相对路径（../common/log.sh）

# dirname xxx  = xxx的 当前目录
# 官方描述： Output each NAME with its last non-slash component and trailing slashes removed; if NAME contains no /'s, output '.' (meaning the current directory).

# readlink -f xxx = 获取指向xxx的绝对路径（非软链接、非相对路径）
# 官方描述： Print value of a symbolic link or canonical file name

# 获取该日志脚本当前所在目录
LOG_SH_PASH="`readlink -f $(dirname $BASH_SOURCE)`"

# 获取执行脚本（加载log.sh的脚本）所在目录，并进入该目录
EXEC_SH_PATH=$(cd "$(dirname "$0")";pwd)

## 验证：
# $BASH_SOURCE = ./dir/common_log.sh
# $(dirname $BASH_SOURCE) = ./dir
# readlink -f $(dirname $BASH_SOURCE) = /opt/demo_assert/dir

# $(dirname "$0") = .
# $(cd "$(dirname "$0")";pwd) = /opt/demo_assert

# test
# $(readlink -f $BASH_SOURCE) = /opt/demo_assert/dir/common_log.sh

# 日志存放的统一路径
LOG_FILES_PATH="$LOG_SH_PASH/logs"

# 默认的日志文件
LOG_FILE_NAME="$LOG_FILES_PATH/exec.log"

function log_current_time()
{
        # 打印时间 = 2018-09-23 20:38:01
        echo "$(date '+%Y-%m-%d %H:%M:%S')"
}


# 假设：脚本使用的用户名为dbuser，群组为dbgroup
# 假设：文件目录必然存在 = mkdir -p ${LOG_FILES_PATH}
# 假设：日志内容不多，不需要压缩日志文件、不要管控日志数量、不需要管控文件夹大小

# 初始化默认的日志文件
function log_touch_file()
{
        local file="${LOG_FILE_NAME}"
        touch ${file} &>/dev/null 
        chmod 640 ${file} &>/dev/null 
        # chown dbuser:dbgroup ${file} &>/dev/null
        return 0
}

# 初始化定制的日志文件
function log_touch_exec_file()
{
        # 路径需要进行校验
        local file="$LOG_FILES_PATH/$1"
        touch ${file} &>/dev/null 
        chmod 640 ${file} &>/dev/null 
        # chown dbuser:dbgroup ${file} &>/dev/null
        # 全局变量赋值，确保exec.sh中日志都会输出到指定文件中
        LOG_FILE_NAME=${file}
        return 0
}

# 常用日志信息：脚本启动日志
function log_begin()
{
        local name=$1
        log "WARNING" "${name}" "######################################## ${name} Bgein ########################################"
}

function log_end()
{
        local name=$1
        log "WARNING" "${name}" "######################################## ${name} End ########################################"
}

########################################
#
# 记录日志的公共方法
# level：级别，DEBUG、INFO、WARNING、ERROR
# function_name：方法名
# info：日志信息
# echo_to_console：是否控制台输出, yes or no
#
########################################
function log()
{
        local level=$1
        local function_name=$2
        local info=$3
        local echo_to_console=$4

        local echo_str="[$(log_current_time)][$level][$function_name][$info]"

        echo ${echo_str}>>${LOG_FILE_NAME}
        if [[ "${echo_to_console}" == "yes" ]]
        then
                echo ${echo_str}
        fi
        return 0
}
```

