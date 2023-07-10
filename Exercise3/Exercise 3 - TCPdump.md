# Exercise 3 - TCPdump

这个实验是来测试TCPdump软件的，所以我们要先准备环境

```
cd $HOME
mkdir fuzzing_tcpdump && cd fuzzing_tcpdump/
wget https://github.com/the-tcpdump-group/tcpdump/archive/refs/tags/tcpdump-4.9.2.tar.gz
tar -xzvf tcpdump-4.9.2.tar.gz
wget https://github.com/the-tcpdump-group/libpcap/archive/refs/tags/libpcap-1.8.0.tar.gz
tar -xzvf libpcap-1.8.0.tar.gz
mv libpcap-libpcap-1.8.0/ libpcap-1.8.0
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
./configure --enable-shared=no
make
cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
./configure --prefix="$HOME/fuzzing_tcpdump/install/"
make
make install
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -h
```

安装后我们打开tcpdump查看

![屏幕截图 2023-07-10 104150](Images\屏幕截图 2023-07-10 104150.png)

然后我们为fuzz测试准备测试样例

```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r ./tests/geneve.pcap
```

![屏幕截图 2023-07-10 104245](Images\屏幕截图 2023-07-10 104245.png)

Tcpdump是用来抓包的一款软件，打开捕获的pcap数据包，可以看见多个包的内容。

AddressSanitizer是一种用于C和c++的快速内存错误检测器。能够查找对堆、堆栈和全局对象的越界访问，以及use-after-free、double-free和内存泄漏错误。可以用来监测fuzz测出来的cash文件。

我们清除之前编译过的文件，为后来的插桩做准备

```
rm -r $HOME/fuzzing_tcpdump/install
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
make clean

cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
make clean
```

进行插桩

```
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make

cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
AFL_USE_ASAN=1 CC=afl-clang-lto ./configure --prefix="$HOME/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install
```

我们对插桩编译后的程序进行测试

```
afl-fuzz -m none -i $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/tests/ -o $HOME/fuzzing_tcpdump/out/ -s 123 -- $HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r @@
```

![屏幕截图 2023-07-10 163132](Images\屏幕截图 2023-07-10 163132.png)

我们使用ASan测试cash文件

```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r $HOME/fuzzing_tcpdump/out/default/crashes/id:000001,sig:06,src:009131,time:11955870,execs:4926739,op:havoc,rep:11
```

我们可以看见对cash文件的崩溃总结，包括执行跟踪

![屏幕截图 2023-07-10 202422](Images\屏幕截图 2023-07-10 202422.png)