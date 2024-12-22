
# 1\. 概述


OpenSSL是一个开源的加密工具包和库，主要实现了安全套接字层（SSL）和传输层安全（TLS）协议，以及各种加密算法、数字签名、消息摘要、加密证书等功能。这个库可以说是Web开发尤其是HTTPS通信的基石了。这里就具体讲解一下如何构建它。


# 2\. 构建过程


# 2\.1 Windows环境


首先要说明的是OpenSSL目前的版本（我使用的是3\.4\.0版本）还没有支持使用CMake构建。但是好在作为老牌的开源库，它的构建文档非常详细。先介绍一下Windows环境下的构建，Windows下当然使用MSVC编译器进行构建了，这就要用到MSVC的命令行的工具。我这里使用的是x64 Native Tools Command Prompt for VS 2019，如下图1所示：


![图1 MSVC命令行的工具](https://img2024.cnblogs.com/blog/1000410/202412/1000410-20241221223338012-1125802849.png)


除此之外，MSVC的命令nmake似乎缺少像linux Make或者CMake的Configure这一步，因此，需要额外安装[perl](https://github.com)。另外，还需要安装[NASM](https://github.com):[MeoMiao 萌喵加速](https://biqumo.org)作为汇编器，一般使用这个是为了获得指令集级别的性能优化。安装好这两个程序之后，一般会自动在Path环境变量中增加相应的可执行程序位置。如果没有添加成功，就手动添加一下。当你在CMD终端中分别输入：



```
perl -version
nasm -v

```

有相应的版本号出现的时候，就说明正确安装并且能被系统所识别了，如下所示：



```
C:\Users\Charlee>perl -version
Locale 'Chinese (Simplified)_China.936' is unsupported, and may crash the interpreter.

This is perl 5, version 38, subversion 2 (v5.38.2) built for MSWin32-x64-multi-thread

Copyright 1987-2023, Larry Wall

Perl may be copied only under the terms of either the Artistic License or the
GNU General Public License, which may be found in the Perl 5 source kit.

Complete documentation for Perl, including FAQ lists, should be found on
this system using "man perl" or "perldoc perl".  If you have access to the
Internet, point your browser at https://www.perl.org/, the Perl Home Page.

C:\Users\Charlee>nasm -v
NASM version 2.16.01 compiled on Jun  1 2023

```

由于MSVC的命令行工具是基于CMD终端的，也就是使用不了更方便的Powershell终端。这里给出完整的构建bat脚本如下所示，读者需要注意跟自己的环境相适配，并修改相应的设置值：



```
@echo off
REM 区域设置
set LC_ALL=C
set LANG=C

REM 配置 Visual Studio 版本
SET VS_VERSION=2019
SET VS_EDITION=Enterprise

REM 配置架构（x64 或 x86）
SET TARGET_ARCH=x64

REM 设置 OpenSSL 源码目录和目标目录
SET OPENSSL_SRC_DIR=../Source/openssl-openssl-3.4.0
SET OPENSSL_INSTALL_DIR=%GISBasic%

REM 启动 Visual Studio 的开发者命令行环境
CALL "C:\Program Files (x86)\Microsoft Visual Studio\%VS_VERSION%\%VS_EDITION%\VC\Auxiliary\Build\vcvarsall.bat" %TARGET_ARCH%

IF ERRORLEVEL 1 (
    echo Failed to configure Visual Studio environment.
    EXIT /B 1
)

REM 进入 OpenSSL 源码目录
CD /D %OPENSSL_SRC_DIR%

REM 配置 OpenSSL
perl Configure VC-WIN64A --prefix=%OPENSSL_INSTALL_DIR% --release
IF ERRORLEVEL 1 (
    echo Configuration failed.
    EXIT /B 1
)

REM 构建 OpenSSL
nmake
IF ERRORLEVEL 1 (
    echo Build failed.
    EXIT /B 1
)

REM 测试构建
REM nmake test
REM IF ERRORLEVEL 1 (
REM    echo Tests failed.
REM    EXIT /B 1
REM )

REM 安装到目标目录
nmake install
IF ERRORLEVEL 1 (
    echo Installation failed.
    EXIT /B 1
)

echo Build and installation successful!
EXIT /B 0

```

根据笔者的实际测试，OpenSSL提供的构建选项虽然多，但是只有`--prefix=%OPENSSL_INSTALL_DIR% --release`是生效的，好在release模式也能生成pdb文件，能够满足我的需求了。另外nmake test这一步可以省略，测试构建这一步非常慢。


# 2\.2 Linux环境


在Linux环境下构建OpenSSL就相对简单了，我这里使用的Ubuntu20\.4，构建安装到GISBasic环境变量指定的目录中，具体脚本如下：



```
#!/bin/bash

BuildDir="./openssl-openssl-3.4.0"
InstallDir=$GISBasic

# 加载环境变量文件
source /etc/environment

# 解压缩
unzip -q -o "../Source/openssl-openssl-3.4.0.zip" -d "../Source"

# 检查构建目录是否存在
if [ -d "$BuildDir" ]; then
    rm -rf "$BuildDir" # 目录存在，删除它
fi
# 创建构建目录
mkdir -p "$BuildDir"

cd "../Source/openssl-openssl-3.4.0"

./Configure --openssldir=$BuildDir --prefix=$InstallDir --release

make

make install

```

在Openssl的官方文档中提供了非常多的构建配置选项，笔者这里也没有使用太多，后续有需要再进行修改吧。


# 3\. 使用方式


虽然Openssl并没有提供CMake的编译方式，但是构建完成后却提供了OpenSSLConfig.cmake配置文件，能够被CMake正常识别引入。只需要再CMakeList.txt文件中使用如下语句：



```
find_package(OpenSSL REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE OpenSSL::SSL)
target_link_libraries(${PROJECT_NAME} PRIVATE OpenSSL::Crypto)

```

就可以在我们的主执行程序中使用OpenSSL了。关于这一步读者如果不太理解可以参考一下笔者前面的文章[《CMake构建学习笔记15\-组建第一个程序项目》](https://github.com)。


另外，OpenSSL还提供了一个可执行程序，通过这个可执行程序可以创建一个SSL证书。虽然这个创建的证书是自己给自己签发的，不能被浏览器安全认证，但是可以用来做测试https通信。第一步是创建私钥：



```
openssl genrsa -out server.key 2048

```

这将在当前目录下生成一个名为server.key的2048位RSA私钥。接下来是生成证书签名请求：



```
openssl req -new -key server.key -out server.csr

```

按照提示，输入证书信息。比较关键的是要将Common Name配置程服务器域名或IP地址。如果是本地测试，就填写 localhost。最后就是生成一个有效期为365天的自签名证书server.crt：



```
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

```

