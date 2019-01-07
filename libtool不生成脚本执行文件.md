## autoconf 不生成脚本执行文件
当使用libtool配置工程时, 生成的最终执行文件为一个脚本, 实际的可执行文件在.libs文件夹内.
脚本类型为:
`transfer: Bourne-Again shell script, ASCII text executable, with very long lines`

在Makefile.am的LDFLAGS中添加`-no-install`选项, 即可不用生成中间脚本文件, 而是实际的可执行文件.