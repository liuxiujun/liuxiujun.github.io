---
title: Mac平台VSCode配置C51开发环境
date: 2020-01-06 10:35:08
tags:
	- 单片机
---

### 安装 sdcc 编译器

``` shell
brew install sdcc
```

### 安装CH341驱动

>  开发版集成了usb转串口模块, 使用的是CH340芯片, 需要安装驱动

http://www.wch.cn/download/CH341SER_MAC_ZIP.html

### 确认驱动是否安装成功

> 需要把单片机连接上电脑

``` shell
$ ls /dev/tty.wchusbserial*
/dev/tty.wchusbserial1420
```

### 安装 stcgal 烧录程序

``` shell
pip3 install stcgal
```

### VSCode 配置

需要安装 Code Runner 插件

编辑配置文件 `.vscode/c_cpp_properties.json`

``` json
{
    "configurations": [
        {
            "name": "51",
            "includePath": [
                "${workspaceFolder}/**",
                "/usr/local/Cellar/sdcc/3.9.0/share/sdcc/include",
                "/usr/local/Cellar/sdcc/3.9.0/share/sdcc/include/mcs51"
            ],
            "defines": [],
            "compilerPath": "/usr/local/Cellar/sdcc/3.9.0/bin/sdcc",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        }
    ],
    "version": 4
}
```

编辑配置文件 `.vscode/settings.json`

``` json
{
    "files.associations": {
        "8052.h": "c",
        "8051.h": "c"
    },
    "code-runner.executorMap": {
        "c": "cd $dir & sdcc $fileName & stcgal -P stc89 -p /dev/tty.wchusbserial1420 $fileNameWithoutExt.ihx",
    }
}
```

### 第一个测试程序

``` c
#include <8052.h>
#define ADDR0 P0_0

void main() {
    ADDR0 = 0;
}
```

⌃⌥N 运行, 提示

``` shell
Waiting for MCU, please cycle power:
```

开发版重新上电后，第一个led亮起