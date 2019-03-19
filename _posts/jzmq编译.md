---
title: jzmq编译
date: 2016-04-06 20:04:41
tags: zmq
categories: java tool
---
版本：zmq4.14
<!--more-->
### 下载地址
* [libsodium1.0.8][libsodium] 
* [zmq][zmq]
* [jzmq][jzmq]

### window编译
#### libsodium-1.0.8编译
1. 进入libsodium\builds\msvc\vs2012，双击libsodium.sln
2. 项目编译平台属性设置为：ReleaseDLL, X64
3. 设置编译环境为vs2013:项目->属性配置属性->平台工具集，选VS2013（V120）

#### zeromq-4.1.4
1. 下载[Windows sources][Windows sources]
1. 打开项目zeromq-4.1.4\builds\msvc\vs2013
2. 只编译libzmq这个项目
3. 项目编译平台属性设置为：DynRelease, X64
4. 报错：`1>..\..\..\..\src\tcp_address.cpp(440): error C3861: “if_nametoindex”:  找不到标识符。`
	解决方法：在libzmq工程上右键-属性，弹出的属性页中，在配置属性-连接器-输入中的“附加依赖项”中增加Iphlpapi.lib。然后在出错的“tcp_address.cpp”文件上方增加“#include <netioapi.h>”，如下所示：
	```
	#ifdef ZMQ_HAVE_WINDOWS
	#include "windows.hpp"
	#include <netioapi.h> //新添加
	#else
	#include <sys/types.h>
	#include <arpa/inet.h>
	#include <netinet/tcp.h>
	#include <netdb.h>
	#endif
	```
	[参考][error-if_nametoindex]
5. 报错：
	```
	e:\zmq\zeromq-4.1.4\src\curve_client.hpp(41): fatal error C1083: 
	无法打开包括文件:“sodium.h”: No such file or directory (..\..\..\..\src\stream_engine.cpp)
	```
	解决：libzmq 属性-配置属性-c/c++-常规-附加包含的目录，加入
	libsodium-1.0.8\src\libsodium\include
	libsodium-1.0.8\src\libsodium\include\sodium
6. 报错：
	```
	1>LINK : fatal error LNK1181: 无法打开输入文件“libsodium.lib”
	```
	解决方法：在libzmq 项目->属性配置属性->链接器->常规->附加库目录，增加libsodium的编译好的dll的目录。（比如：E:\libsodium\Build\ReleaseDLL\x64）
7. 报错：
	```
	E:\zeromq-4.1.4\builds\msvc\vs2013\libsodium.import.props(39,5): error MSB3030:  
	无法复制文件“E:\zmq\zeromq-4.1.4\builds\msvc\vs2012\libzmq\..\..\..\..\..\libsodium\bin\x64\Release\v110\dynamic\libsodium.dll”，原因是找不到该文件。
	```
	解决方法：把libsodium-1.0.8改名为libsodium

#### 编译jzmq-3.1.0
1. 打开jzmq-3.1.0\builds\msvc
2. 编译平台属性设置为：Release, X64
3. 设置包含目录：属性配置-vc++目录-包含目录加入：zeromq\include,jdk\include,jdk\include\win32
4. 设置库目录：属性配置-vc++目录-库目录加入：
	zeromq-4.1.4编译后的目录
5. 报错
	```
	1>e:\zmq\jzmq-3.1.0\src\main\c++\jzmq.hpp(23): fatal error C1083: 
	无法打开包括文件:“config.hpp”: No such file or directory
	```
	解决：将jzmq-3.1.0\builds\msvc下的config.hpp放入jzmq-3.1.0\src\main\c++中。



### linux编译
#### 安装需要的组件
* autoconf
```
	查看当前版本
	#rpm -qf /usr/bin/autoconf
	卸载  
	rpm -e --nodeps autoconf   
	下载
	wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.gz  
	解压安装
	tar zxvf autoconf-2.69.tar.gz  
	cd autoconf-2.69  
	./configure --prefix=/usr  
	make && make install  
```
* libtool gcc gcc-c++
	```
	yum install libtool
	yum install gcc
	yum install gcc-c++
	```

#### 编译libsodium-1.0.8
```
./autogen.sh
./configure
make && make check
make install
```
#### 编译zeromq-4.1.4
下载[POSIX tarball][POSIX tarball]
```
./autogen.sh
./configure
make
make install 
```

#### 编译jzmq-3.1.0
* 将`src/main/c++/Event.cpp`替换，具体[查看][event.cpp]，否则会报错：
` ‘zmq_event_t’ was not declared in this scope `,查看[issues][issues]
```
./autogen.sh
./configure
make
make install 
```

#### 编译后文件位置
* so文件
/usr/local/lib
* jar
/usr/local/share/java/zmq.jar


参考文章
* [https://www.javacodegeeks.com/2015/09/how-to-configure-and-install-zeromq-libsodium-on-centos-6-7.html][ck1]
* [https://github.com/zeromq/jzmq/issues/401][ck2]

[libsodium]: https://github.com/jedisct1/libsodium/tree/1.0.8
[zmq]: http://zeromq.org/intro:get-the-software
[jzmq]: https://github.com/zeromq/jzmq/tree/v3.1.0
[error-if_nametoindex]: http://www.th7.cn/system/win/201602/152056.shtml
[event.cpp]: https://github.com/zeromq/jzmq/commit/eb40d6db43ce3545e623dad6cc6721a90885b5ba
[issues]: https://github.com/zeromq/jzmq/issues/401
[POSIX tarball]: http://download.zeromq.org/zeromq-4.1.4.tar.gz
[Windows sources]: http://download.zeromq.org/zeromq-4.1.4.zip
[ck1]: https://www.javacodegeeks.com/2015/09/how-to-configure-and-install-zeromq-libsodium-on-centos-6-7.html
[ck2]: https://github.com/zeromq/jzmq/issues/401
