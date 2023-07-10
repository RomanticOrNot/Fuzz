# Exercise 2 - libexif

首先准备测试环境，我们测试的对象是libexif库，libexif 是一个用来读取数码相机照片中包含的 exif 信息的 C 语言库。所以我们先下载libexif库并解压和安装

```
wget https://github.com/libexif/libexif/archive/refs/tags/libexif-0_6_14-release.tar.gz
tar -xzvf libexif-0_6_14-release.tar.gz
cd libexif-libexif-0_6_14-release/
sudo apt-get install autopoint libtool gettext libpopt-dev
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/"
make
make install
```

我们要测试一个库，需要使用一个调用该库的程序，然后对该程序进行Fuzz测试，在这里我们选用**exif command-line**作为目标程序**exif command-line**可以用来显示图像方向 exif 数据

```
cd $HOME/fuzzing_libexif
wget https://github.com/libexif/exif/archive/refs/tags/exif-0_6_15-release.tar.gz
tar -xzvf exif-0_6_15-release.tar.gz
cd exif-exif-0_6_15-release/
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
```

测试exif command-line

![image-20230708221404799](Images\image-20230708221404799.png)

接下来准备测试用例

![image-20230708221930977](C:\Users\wn\AppData\Roaming\Typora\typora-user-images\image-20230708221930977.png)

```
wget https://github.com/ianare/exif-samples/archive/refs/heads/master.zip
unzip master.zip
$HOME/fuzzing_libexif/install/bin/exif $HOME/fuzzing_libexif/exif-samples-master/jpg/Pentax_K10D.jpg
```

![image-20230708222204517](Images\image-20230708222204517.png)

准备好测试用例后我们要利用**afl-clang-lto**作为编译器去对exif软件和libexif库进行重编译来插桩，监测覆盖率

```
rm -r $HOME/fuzzing_libexif/install
cd $HOME/fuzzing_libexif/libexif-libexif-0_6_14-release/
make clean
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/"
make
make install
//重编译libexif库
cd $HOME/fuzzing_libexif/exif-exif-0_6_15-release
make clean
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
//重编译exif软件
```

afl-clang-lto是一种无碰撞的工具，而且比afl-clang-fast更快。当适用C/C++ 11+时使用LTO模式(afl-clang-lto/afl-clang-lto++)，否则如果适用C/C++ 6+时使用LLVM模式(afl-clang-fast/afl-clang-fast++),如果以上都不适用只适用于gcc 5+以上使用GCC_PLUGIN模式(afl-gcc-fast/afl-g++-fast)，最后的选择是使用GCC mode (afl-gcc/afl-g++) (or afl-clang/afl-clang++ for clang)

接下来我们对exif进行测试

![屏幕截图 2023-07-08 202819](Images\屏幕截图 2023-07-08 202819.png)

然后我们利用Eclipse-CDT来进行调试![屏幕截图 2023-07-08 234443](Images\屏幕截图 2023-07-08 233150.png)

![image-20230708234521394](Images\image-20230708234521394.png)

设置好之后我们对exif进行debug，点击Run -> Resume后程序会在段错误的地方停下，我们可以看见data_load_data函数报错点击发现这个就是[**CVE-2012-2836**](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-2836)漏洞，这个漏洞会产生整数溢出问题。而[**CVE-2009-3895**](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3895) 可能存在堆上的越界读写行为

![image-20230708224352151](Images\image-20230708224352151.png)