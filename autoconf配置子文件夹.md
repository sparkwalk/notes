# configure.ac 配置子目录

## 不带参数配置子目录

``` shell
AC_CONFIG_SUBDIRS([libs/jansson-2.11])
```

相当于默认执行子目录下面的`./configure`

## 带参数配置子目录
`AX_SUBDIRS_CONFIGURE( [subdirs], [mandatory arguments], [possibly merged arguments], [replacement arguments], [forbidden arguments])`

1.下载脚本[m4_ax_subdirs_configure.m4](http://git.savannah.gnu.org/gitweb/?p=autoconf-archive.git;a=blob_plain;f=m4/ax_subdirs_configure.m4)放到工程下的m4文件夹.
2.添加配置参数

``` shell
AX_SUBDIRS_CONFIGURE([libs/libevent-2.1.8], [[--disable-samples], [--disable-openssl]])
```

相当于默认执行子目录下面的
`./configure --disable-samples --disable-openssl`