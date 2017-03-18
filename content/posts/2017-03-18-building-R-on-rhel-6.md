---
date: '2017-03-18'
tags:
- linux
- R
- build
title: "Building R on RedHat Linux 6"
---

Newer versions of R(>=3.3.x) will not build without newer bzip2, zlib, libcurl & pcre. These dependencies are not available on older version of RHEL & CentOS i.e. anything below RHEL & CentOS 7. This blog post is a step-by-step journal of what I encountered when trying to compile & install R v3.3.2 on a CentOS 6 host.
<!--more-->

New versions of R have removed several dependencies including compression libraries that used to be shipped together which would make the compilation process much easier. Instead of providing these dependencies R assumes that they are installed in the build host. On a CentOS 7 build host, this is not a problem, because it already has newer versions of R dependencies, but the opposite is the case with older version of RHEL & CentOS.

Since I'm using an older version of CentOS—CentOS 6—I prefer using [Software Collections](https://www.softwarecollections.org/en/) when building stuff because it provides a bit newer versions of build tools e.g. `cc`, `gcc`, e.t.c.

1. Enable [devtoolset-3](https://www.softwarecollections.org/en/scls/rhscl/devtoolset-3/):

        $ scl enable devtoolset-3 bash

2. Download R & build it

        $ cd src
        $ mkdir -p ~/apps/R/3.3.2
        $ wget -qO- https://cran.r-project.org/src/base/R-3/R-3.3.2.tar.gz | tar zxv
        $ cd R-3.3.2/
        $ ./configure --prefix=$HOME/apps/R/3.3.2 --enable-R-static-lib

    - It fails with the following error about missing newer version of `zlib`

            checking for XDR support... yes
            checking if zlib version >= 1.2.5... no
            checking whether zlib support suffices... configure: error: zlib library and headers are required

3. Download zlib v1.2.8, compile & install it

        $ cd ../
        $ mkdir -p ~/apps/zlib/1.2.8
        $ wget -qO- "https://downloads.sourceforge.net/project/libpng/zlib/1.2.8/zlib-1.2.8.tar.gz" | tar zvx
        $ cd zlib-1.2.8/
        $ ./configure --prefix=$HOME/apps/zlib/1.2.8
        $ make -j4
        $ make install

4. Create environment variables which will be used when building R

        $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/apps/zlib/1.2.8/lib
        $ export CFLAGS="-I$HOME/apps/zlib/1.2.8/include"
        $ export LDFLAGS="-L$HOME/apps/zlib/1.2.8/lib"

    - Re-configure R once more, this time round it should fix the missing `zlib` library error

            $ cd ../R-3.3.2/
            $ ./configure --prefix=$HOME/apps/R/3.3.2 --enable-R-static-lib

    - `configure` fails with the following error about missing `bzip2`:

            checking bzlib.h usability... yes
            checking bzlib.h presence... yes
            checking for bzlib.h... yes
            checking if bzip2 version >= 1.0.6... no
            checking whether bzip2 support suffices... configure: error: bzip2 library and headers are required

5. Download bzip2 v1.0.6, compile and install it

        $ cd ../
        $ mkdir -p $HOME/apps/bzip/1.0.6
        $ wget -qO- http://www.bzip.org/1.0.6/bzip2-1.0.6.tar.gz | tar xzv
        $ cd bzip2-1.0.6/
        $ make -f Makefile-libbz2_so
        $ make clean
        $ make
        $ make -n install PREFIX=$HOME/apps/bzip/1.0.6/
        $ make install PREFIX=$HOME/apps/bzip/1.0.6/

    > **NOTE**
    > 
    > Building bzip2 requires extra steps since it is not built with the standard GNU auto tools.
    > I wouldn't have built it without help from this site: http://www.linuxfromscratch.org/lfs/view/development/chapter06/bzip2.html

6. Try building R once more

    - Export new shell environment variables to be used in the build process; this includes bzip2 binaries and libraries installed in the previous step: 

            $ export PATH=$HOME/apps/bzip/1.0.6/bin:$PATH
            $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/apps/bzip/1.0.6/lib
            $ export LDFLAGS="-L$HOME/apps/zlib/1.2.8/lib -L$HOME/apps/bzip/1.0.6/lib/"
            $ export CFLAGS="-I$HOME/apps/zlib/1.2.8/include -I$HOME/apps/bzip/1.0.6/include"

    - Build R

            $ cd ../R-3.3.2/
            $ ./configure --prefix=$HOME/apps/R/3.3.2 --enable-R-static-lib

    - Configure fails with the following error about `liblzma`

            checking lzma.h usability... yes
            checking lzma.h presence... yes
            checking for lzma.h... yes
            checking if lzma version >= 5.0.3... no
            configure: error: "liblzma library and headers are required"

7. We can use `xz` instead of `liblzma`.

        $ cd ../
        $ wget -qO- http://tukaani.org/xz/xz-5.2.2.tar.gz | tar xzv
        $ cd xz-5.2.2/
        $ mkdir -p $HOME/apps/xz/5.2.2
        $ ./configure --prefix=$HOME/apps/xz/5.2.2
        $ make -j4
        $ make install

8. Configure R once again

    - Export shell environment variables to contain binaries and libraries from `xz` installed in the previous step

            $ export PATH=$HOME/apps/xz/5.2.2/bin:$PATH
            $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/apps/xz/5.2.2/lib
            $ export LDFLAGS="-L$HOME/apps/zlib/1.2.8/lib -L$HOME/apps/bzip/1.0.6/lib -L$HOME/apps/xz/5.2.2/lib"
            $ export CFLAGS="-I$HOME/apps/zlib/1.2.8/include -I$HOME/apps/bzip/1.0.6/include -I$HOME/apps/xz/5.2.2/include"

    - Build R

            $ cd ../R-3.3.2/
            $ ./configure --prefix=$HOME/apps/R/3.3.2 --enable-R-static-lib

    - Configure fails with the following error about prce

            checking for pcre.h... yes
            checking pcre/pcre.h usability... no
            checking pcre/pcre.h presence... no
            checking for pcre/pcre.h... no
            checking if PCRE version >= 8.10, < 10.0 and has UTF-8 support... no
            checking whether PCRE support suffices... configure: error: pcre >= 8.10 library and headers are required


9. Download pcre v8.35, compile and install it

        $ cd ../
        $ wget -qO- http://downloads.sourceforge.net/pcre/pcre-8.35.tar.bz2 | tar xjv
        $ cd pcre-8.35/
        $ mkdir -p $HOME/apps/pcre/8.35
        $ ./configure --prefix=$HOME/apps/pcre/8.35 --enable-unicode-properties --enable-pcre16 --enable-pcre32 --enable-pcregrep-libz --enable-pcregrep-libbz2 --enable-pcretest-libreadline --enable-static
        $ make -j4
        $ make check
        $ make install


10. Configure R once again

    - Export shell environment variables to contain binaries and libraries from pcre installed in the previous step

            $ export PATH=$HOME/apps/pcre/8.35/bin:$PATH
            $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/apps/pcre/8.35/lib
            $ export LDFLAGS="-L$HOME/apps/zlib/1.2.8/lib -L$HOME/apps/bzip/1.0.6/lib -L$HOME/apps/xz/5.2.2/lib -L$HOME/apps/pcre/8.35/lib"
            $ export CFLAGS="-I$HOME/apps/zlib/1.2.8/include -I$HOME/apps/bzip/1.0.6/include -I$HOME/apps/xz/5.2.2/include -I$HOME/apps/pcre/8.35/include"

    - Build R

            $ cd ../R-3.3.2/
            $ ./configure --prefix=$HOME/apps/R/3.3.2 --enable-R-static-lib

    - configure fails with the following error about missing libcurl

            checking for curl-config... /usr/bin/curl-config
            checking libcurl version ... 7.19.7
            checking curl/curl.h usability... yes
            checking curl/curl.h presence... yes
            checking for curl/curl.h... yes
            checking if libcurl is version 7 and >= 7.28.0... no
            configure: error: libcurl >= 7.28.0 library and headers are required with support for https


11. Download libcurl v7.47, compile and install it

        $ cd ../
        $ wget -qO- https://curl.haxx.se/download/curl-7.47.1.tar.gz | tar xvz
        $ cd curl-7.47.1
        $ mkdir -p $HOME/apps/curl/7.47.1/
        $ ./configure --prefix=$HOME/apps/curl/7.47.1
        $ make -j4
        $ make install


12. Configure R once again

    - Export new environment variables to contain binaries and libraries from libcurl installed in the previous step

            $ export PATH=$HOME/apps/curl/7.47.1/bin:$PATH
            $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/apps/curl/7.47.1/lib
            $ export LDFLAGS="-L$HOME/apps/zlib/1.2.8/lib -L$HOME/apps/bzip/1.0.6/lib -L$HOME/apps/xz/5.2.2/lib -L$HOME/apps/pcre/8.35/lib -L$HOME/apps/curl/7.47.1/lib"
            $ export CFLAGS="-I$HOME/apps/zlib/1.2.8/include -I$HOME/apps/bzip/1.0.6/include -I$HOME/apps/xz/5.2.2/include -I$HOME/apps/pcre/8.35/include -I$HOME/apps/curl/7.47.1/include"

    - Try building R once again

            $ cd ../R-3.3.2/
            $ ./configure --prefix=$HOME/apps/R/3.3.2 --enable-R-static-lib --enable-memory-profiling --enable-R-profiling

    - Finally! It worked! Now finish up the installation

            $ make -j4
            $ make install

