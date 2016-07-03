---
layout: post
title: 成功在树莓派上运行恒星节点
published: true
date: 2016-06-25 15:56:01 +0800
---

[恒星网络](http://stellar.org)是新一代去中心化分布式账本，在比特币和瑞波的基础上做了的很多改进，其中一项就是运行节点的资源需求非常低。所以一直有在树莓派上运行恒星节点的想法。

#### 64bit！64bit？

之前有运行stellar-core的[经验](https://github.com/strllar/poor_dockers/)，知道它[需要64位的time_t](https://github.com/stellar/stellar-core/blob/v0.5.0/src/util/Timer.h#L63-L64)。所以在树莓派3发布时很期待，因为第一次集成了64位处理器。但进一步调研发现官方系统仍然是32位，[社区在尝试64位模式](https://www.raspberrypi.org/forums/viewtopic.php?f=72&t=137963)，其中有日本网友制作了[64位docker环境](http://atbsd.com/?p=39)，看上去和移植stellar-core需要的基础已经非常接近。但考虑到这个方式64位host系统还不稳定，而且arm64的docker镜像也没有经过足够测试，不知道还有多少坑，所以一直没有下决定订购。同样因为对社区成熟度的担心，原生64位支持的[PINE64](http://www.pine64.com)也仅是关注。

#### Rasberry Pi Model B (2014)

直到上星期，从回收来的显卡挖矿机零件中里找出一个树莓派，还是2013年底全民挖矿时的配置，当时用做矿机的看门狗，负责死机后充启。综合外观和年份，确定应该是[第一代Model B](https://www.raspberrypi.org/products/model-b/)，配置有ARMv6 (32-bit)的BCM2835处理器和512M内存，存储为4G SD卡。

![Photo of Nezha](/images/nezha_rpi.jpg)

拿着这个小设备的时候突然冒出一个想法，我们知道32位平台上处理int64_t是没有问题的，编译器能够转化为32位操作模拟。那有没有可能通过软件模拟出64位time_t呢，很快看到[google的结果是不能](http://stackoverflow.com/questions/14361651/is-there-any-way-to-get-64-bit-time-t-in-32-bit-program-in-linux)。但同时看到这个结果：[NetBSD下time_t都是64位啦！](https://news.ycombinator.com/item?id=7678847)

#### NetBSD Is the Key

于是从[这里](https://wiki.netbsd.org/ports/evbarm/raspberry_pi/)找到NetBSD 7.0的镜像，装到SD卡，开机，用root登入，安装pkgin。因为evbarm编译好的二进制包中只有4.8版本的gcc，不满足stellar-core最低4.9的要求，花费数天试图通过pkgsrc交叉编译出evbarm上可用的gcc5不成功之后，选择使用clang。安装过程如下：

```sh
# @netbsd/arm
export PKG_PATH=http://ftp.netbsd.org/pub/pkgsrc/packages/NetBSD/earmv6hf/7.0_HEAD/All
pkg_add pkgin
pkgin install git libtool-base pkg-config automake gmake bison flex pandoc clang
```

但clang仍然使用系统自带的libstdc++ 4.8版本，不符合stellar-core的要求。于是决定构建libc++库以支持stellar－core需要的c++11特性。3.7版本libc++需要cmake 3.5版本，而给evbarm编译好的二进制包中只有3.4版本。这些依赖软件如果直接在树莓派上编译将耗费大量的时间，所以首先按照 <https://ftp.netbsd.org/pub/pkgsrc/current/pkgsrc/doc/HOWTO-use-crosscompile> 中的指导，本地安装一个NetBSD7的系统，并配置交叉编译环境(非root方式)。

```sh
# @netbsd/x64
wget ftp://ftp.netbsd.org/pub/NetBSD/NetBSD-7.0.1/source/sets/gnusrc.tgz
wget ftp://ftp.netbsd.org/pub/NetBSD/NetBSD-7.0.1/source/sets/sharesrc.tgz
wget ftp://ftp.netbsd.org/pub/NetBSD/NetBSD-7.0.1/source/sets/src.tgz
wget ftp://ftp.netbsd.org/pub/NetBSD/NetBSD-7.0.1/source/sets/syssrc.tgz

for file in *.tgz
do
    tar xfz $file
done

wget ftp://ftp.netbsd.org/pub/pkgsrc/stable/pkgsrc.tar.xz
tar --xz -xf pkgsrc.tar.xz -C usr

cd usr/src && ./build.sh -U -u -m evbarm -a earmv6hf tools distribution
cd ~/usr/pkgsrc/bootstrap && ./bootstrap  --prefix=~/usr/pkg --unprivileged
```

构造出工具链并完成pkgsrc自举之后，修改`~usr/pkg/etc/mk.conf`为:

```
# @netbsd/x64
# Example /home/rpi/usr/pkg/etc/mk.conf file produced by bootstrap-pkgsrc
# Wed Jun 15 08:24:20 UTC 2016

.ifdef BSD_PKG_MK       # begin pkgsrc settings

#ABI=                   64

UNPRIVILEGED=           yes
PKG_DBDIR=              /home/rpi/usr/pkg/var/db/pkg
LOCALBASE=              /home/rpi/usr/pkg
VARBASE=                /home/rpi/usr/pkg/var
PKG_TOOLS_BIN=          /home/rpi/usr/pkg/sbin
PKGINFODIR=             info
PKGMANDIR=              man


.endif                  # end pkgsrc settings

USE_CROSS_COMPILE?=  yes
CROSSBASE=           ${LOCALBASE}/cross-${TARGET_ARCH:U${MACHINE_ARCH}}
.if !empty(USE_CROSS_COMPILE:M[yY][eE][sS])
MACHINE = evbarm
MACHINE_ARCH=        earmv6hf
TOOLDIR=             /home/rpi/usr/src/obj/tooldir.NetBSD-7.0.1-amd64
CROSS_DESTDIR=       /home/rpi/usr/src/obj/destdir.evbarm
PACKAGES=            ${PKGSRCDIR}/packages.${MACHINE_ARCH}
WRKDIR_BASENAME=     work.${MACHINE_ARCH}
.endif
```

然后构造cmake以及相关依赖项目，

```sh
# @netbsd/x64
export PATH=~/usr/pkg/bin/:~/usr/pkg/sbin:$PATH
cd ~/usr/pkgsrc/devel/cmake && bmake package
```

将构造出的软件包从`~/usr/pkgsrc/packages.earmv6hf/All`目录复制到树莓派并安装.然后在树莓派中签出并构建libc++

```sh
# netbsd/arm
export PATH=~/usr/pkg/bin/:~/usr/pkg/sbin:$PATH
svn co http://llvm.org/svn/llvm-project/llvm/tags/RELEASE_370/final  llvm37
cd llvm/projects
svn co http://llvm.org/svn/llvm-project/compiler-rt/tags/RELEASE_370/final compiler-rt
svn co http://llvm.org/svn/llvm-project/libcxxabi/tags/RELEASE_370/final libcxxabi
svn co http://llvm.org/svn/llvm-project/libcxx/tags/RELEASE_370/final libcxx
svn co http://llvm.org/svn/llvm-project/libunwind/tags/RELEASE_370/final libunwind
cd ../../ && mkdir libcxx-build && cd libcxx-build && CXX=/usr/pkg/bin/clang++ CC=/usr/pkg/bin/clang cmake -G "Unix Makefiles" ../llvm/
make cxx
```

构建libc++成功后，就完成了stellar-core需要的环境准备。按照常规步骤，

```sh
#nerbsd/arm
git clone https://github.com/stellar/stellar-core.git
cd stellar-core
./autogen.sh
./configure --disable-postgres CC=clang CXX=clang++ CXXFLAGS="-I/home/rpi/libcxx-build/include -I/home/rpi/libcxx-build/include/c++/v1 -L/home/rpi/libcxx-build/lib -stdlib=libc++"
LD_LIBRARY_PATH=/home/rpi/libcxx-build/lib/ gmake
```

成功构建stellar-core之后，通过

```sh
# @netbsd/arm
LD_LIBRARY_PATH=/home/rpi/libcxx-build/lib/ stellar-core

```

运行成功。

#### Crash!
构建出stellar－core后，发现经常突然退出，于是通过

```sh
ktruss -t s -p <stellar-core-pid>
```

观察到退出时总是显示，

```
...
24049 2 stellar-core SIGCHLD caught handler=0x3cc9d0 mask=0x0 code=0x0
24049 2 stellar-core SIGPIPE SIG_DFL
```

发现是`SIGPIPE`导致的退出，进一步排查发现
[4b4eb669](http://github.com/stellar/stellar-core/pull/1043/files#diff-4b4eb6693b4956a1cc728c7e7dc69eb9)
中所需要的修改，修改后stellar-core不再异常退出。

#### Pull Requests

代码在NetBSD环境下需要的一些改动，已经提交：

- <https://github.com/stellar/medida/pull/6>
- <https://github.com/xdrpp/xdrpp/pull/8>
- <https://github.com/stellar/stellar-core/pull/1043>

#### Last but Most Important

最重要的是广告时间：

```
"GDPJ4DPPFEIP2YTSQNOKT7NMLPKU2FFVOEIJMG36RCMBWBUR4GTXLL57 nezha"
```

敬请各路英雄将本节点加入`QUORUM_SET`，小[哪吒](https://en.wikipedia.org/wiki/Nezha_(deity))是正直的！
