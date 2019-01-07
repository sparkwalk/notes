# Makefile.am 配置选项

## 生成库

### 1. 静态库(.a)

```
lib_LIBRARIES     = libccan.a

libccan_a_CFLAGS  =
libccan_a_LDFLAGS =   
libccan_a_LDADD   =
libccan_a_SOURCES =
```

### 2. libtool格式的静态库(.la)

```
noinst_LTLIBRARIES = libccan.la

libccan_la_CFLAGS  =                
libccan_la_LDFLAGS =
libccan_a_LDADD    =
libccan_la_SOURCES =
```

### 3. 动态库(.so)

```
lib_LTLIBRARIES    = libccan.la

libccan_la_CFLAGS  =
libccan_la_LDFLAGS =
libccan_a_LDADD    =
libccan_la_SOURCES =
```

## 生成可执行文件

```
bin_PROGRAMS=transfer

transfer_CFLAGS  =
transfer_LDFLAGS =
transfer_LDADD   =
transfer_SOURCES =
```

