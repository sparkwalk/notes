# 编译GCC 4.6.3

## 1.下载源码包

gcc依赖三个库gmp, mpfr和mpc.

- [gcc-4.6.3.tar.gz](http://ftp.tsukuba.wide.ad.jp/software/gcc/releases/gcc-4.6.3/gcc-4.6.3.tar.gz)

- [gmp-4.3.2.tar.bz2](ftp://gcc.gnu.org/pub/gcc/infrastructure/gmp-4.3.2.tar.bz2)

- [mpfr-2.4.2.tar.bz2](ftp://gcc.gnu.org/pub/gcc/infrastructure/mpfr-2.4.2.tar.bz2)

- [mpc-0.8.1.tar.gz](ftp://gcc.gnu.org/pub/gcc/infrastructure/mpc-0.8.1.tar.gz)

## 2.编译安装依赖库

### 1).编译gmp库

``` bash
./configure --prefix=/home/licg/.local
make
make install
```

### 2).编译mpfr库

``` bash
./configure --prefix=/home/licg/.local --with-gmp=/home/licg/.local
make
make install
```

### 3).编译mpc库

``` bash
./configure --prefix=/home/licg/.local --with-gmp=/home/licg/.local --with-mpfr=/home/licg/.local
make
make install
```

## 3.设置环境变量

编辑home下的.profile文件, 添加

``` bash
export C_INCLUDE_PATH=/home/licg/.local/include   			    #C头文件搜索路径
export LD_LIBRARY_PATH=/home/licg/.local/lib:$LD_LIBRARY_PATH	#运行时库搜索路径 
export LIBRARY_PATH=$LD_LIBRARY_PATH			  				#编译链接库搜索路径
```

使环境变量生效

``` bash
. .profile
```



## 4.编译gcc

配置编译

``` bash
./configure --prefix=/home/licg/.local
make 
make install
```

编译安装完成后, 记得修改PATH变量和LIB位置

``` bash
export PATH=/home/licg/.local/bin/:$PATH 
export LD_LIBRARY_PATH=/home/licg/.local/lib
```



## 5.可能出现的问题

### 1).`LIBRARY_PATH shouldn't contain the current directory`

可能是因为LIBRARY_PATH中有两个连续的`:`，删除多余的`:`就行。

### 2).`fatal error: gnu/stubs-32.h: No such file or directory`

可能是因为系统没有安装32位库的头文件，可以安装`libc6-dev-i386（Ubuntu）/glibc-devel.i686（RedHat）`或configure时`--disable-multilib.`禁用32位功能。

``` bash
sudo apt-get install libc6-dev-i386
```

### 3).`gcc-v4.6.1/gcc/system.h:462:20: error: conflicting types for ‘strsignal’`

需要清空这两个环境变量：

``` bash
export CPLUS_INCLUDE_PATH=
export C_INCLUDE_PATH=
```

### 4).出现错误找不到`mpfr.h`或`mpc.h`头文件

设置环境变量包含头文件路径

```bash
export CPLUS_INCLUDE_PATH=/home/licg/.local/include 
export C_INCLUDE_PATH=/home/licg/.local/include 
```

### 5) `../../gcc/doc/cppopts.texi:772: @itemx must follow @item`

需要安装texinfo库[texinfo-4.13a.tar.gz](http://ftp.gnu.org/gnu/texinfo/texinfo-4.13a.tar.gz)

### 6)`/usr/include/features.h:324:26: fatal error: bits/predefs.h: No such file or directory`

``` bash
C_INCLUDE_PATH=/home/licg/.local/include:~/download/libc6-dev/usr/include/i386-linux-gnu:/usr/include/$(gcc -print-multiarch)
```

