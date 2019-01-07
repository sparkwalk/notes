# Error opening terminal: xterm-256color.

使用最新的 `ncurses-6.1`的库, 无法打开`xterm-256color`终端文件, 但是使用`ncurses-5.9`的库就可以打开.

使用`ncurses-6.1`的库生成**tic**重新编译安装下`xterm-256color`即可.

xterm的源文件为`ncurses-6.1/misc/terminfo.src`

```
tic ncurses-6.1/misc/terminfo.src
```

