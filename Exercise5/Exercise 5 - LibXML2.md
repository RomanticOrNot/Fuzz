# Exercise 5 - LibXML2

首先我们准备测试目标libxml2，libxml2是个C语言的XML程式库，能简单方便的提供对XML文件的各种操作，并且支持XPATH查询，及部分的支持XSLT转换等功能。

```
cd $HOME
mkdir Fuzzing_libxml2 && cd Fuzzing_libxml2
wget http://xmlsoft.org/download/libxml2-2.9.4.tar.gz
tar xvf libxml2-2.9.4.tar.gz && cd libxml2-2.9.4/
sudo apt-get install python-dev
CC=afl-clang-lto CXX=afl-clang-lto++ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --prefix="$HOME/Fuzzing_libxml2/libxml2-2.9.4/install" --disable-shared --without-debug --without-ftp --without-http --without-legacy --without-python LIBS='-ldl'
make -j$(nproc)
make install
```

然后我们测试libxml2是否安装成功。

```
./xmllint --memory ./test/wml.xml
```

![屏幕截图 2023-07-14 201022](D:\github\Fuzz\Exercise5\屏幕截图 2023-07-14 201022.png)

为了对libxml2进行fuzz测试我们要准备XML样例

```
mkdir afl_in && cd afl_in
wget https://raw.githubusercontent.com/antonio-morales/Fuzzing101/main/Exercise%205/SampleInput.xml
cd ..
```

接下来我们准备一个XML字典，当我们想fuzz复杂的基于文本的文件格式(如XML)时，为fuzz提供包含基本语法标记列表的字典能有效提升效率。fuzz可以利用该语法字典进行替换和添加。

```
mkdir dictionaries && cd dictionaries
wget https://raw.githubusercontent.com/AFLplusplus/AFLplusplus/stable/dictionaries/xml.dict
cd ..
```

我们可以运行多个命令窗口运行afl-fuzz的多个例子来充分利用多核cpu，来加快我们fuzz测试的速度，同时如果使用-s标志，则需要为每个实例使用不同的种子

```
afl-fuzz -m none -i ./afl_in -o afl_out -s 123 -x ./dictionaries/xml.dict -D -M master -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@
afl-fuzz -m none -i ./afl_in -o afl_out -s 234 -S slave1 -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@
```

![屏幕截图 2023-07-19 150357](D:\github\Fuzz\Exercise5\屏幕截图 2023-07-19 150357.png)









