# Exercise 4 - LibTIFF

我们准备使用fuzz测试LibTIFF来寻找漏洞

首先我们准备环境，下载LIBTIFF并解压

```
cd $HOME
mkdir fuzzing_tiff && cd fuzzing_tiff/
wget https://download.osgeo.org/libtiff/tiff-4.0.4.tar.gz
tar -xzvf tiff-4.0.4.tar.gz
```

然后我们编译和安装LIBTIFF

```
cd tiff-4.0.4/
./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```

然后我们使用/test/images/文件夹中的示例图像作为模糊测试例子。

```
$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
```

![屏幕截图 2023-07-11 220446](D:\github\Fuzz\Exercise4\屏幕截图 2023-07-11 220446.png)

接下来我们使用icov测试软件的代码覆盖率，显示每行代码被触发的次数，可视化模糊过程。

```
sudo apt install lcov
```

现在我们用——coverage标志重新编译libTIFF

```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
  
CFLAGS="--coverage" LDFLAGS="--coverage" ./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```

接下来我们用lcov收集代码覆盖率数据

```
cd $HOME/fuzzing_tiff/tiff-4.0.4/
lcov --zerocounters --directory ./
lcov --capture --initial --directory ./ --output-file app.info
$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
lcov --no-checksum --directory ./ --capture --output-file app2.info
```

我们运行lcov后，设置生成的html后，进行查看

```
genhtml --highlight --legend -output-directory ./html-coverage/ ./app2.info
```

![屏幕截图 2023-07-11 232748](D:\github\Fuzz\Exercise4\屏幕截图 2023-07-11 232748.png)

首先我们进行重新编译

```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
```

再设置AFL_USE_ASAN=1

```
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
AFL_USE_ASAN=1 make -j4
AFL_USE_ASAN=1 make install
```

然后我们运行fuzz测试

```
afl-fuzz -m none -i $HOME/fuzzing_tiff/tiff-4.0.4/test/images/ -o $HOME/fuzzing_tiff/out/ -s 123 -- $HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w @@
```

![屏幕截图 2023-07-12 003641](D:\github\Fuzz\Exercise4\屏幕截图 2023-07-12 003641.png)

我们利用tiffinfo进行查看，发现发生了溢出

![屏幕截图 2023-07-12 010249](D:\github\Fuzz\Exercise4\屏幕截图 2023-07-12 010249.png)

我们利用lcov检查一下代码覆盖率![屏幕截图 2023-07-12 215621](D:\github\Fuzz\Exercise4\屏幕截图 2023-07-12 215621.png)