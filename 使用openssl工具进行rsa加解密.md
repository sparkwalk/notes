## 1.生成私钥

使用aes256算法和密码hello对私钥进行加密, 生成private.key

``` shell
openssl genrsa -out private.key -aes256 -passout pass:hello 4096
```

## 2.提取公钥

使用aes256算法和密码hello生成公钥 public.key

``` shell
openssl rsa -in private.key -pubout -out public.key -passin pass:hello
```

## 3.加密明文

cleartext 内容:

```
Mary has a little lamp.
```

使用公钥加密:

``` shell
openssl rsautl -encrypt -in cleartext -inkey public.key -pubin -out chiper.bin
```

chiper.bin是二进制文件并不能直接读取, 需要做base64编码

``` 
openssl base64 -e -in chiper.bin -out chiper.base64
```

## 4.解密密文

base64编码后的密文需要解码为二进制文件

```
openssl base64 -d -in chiper.base64 -out chiper.bin
```

使用私钥解密

```
openssl rsautl -decrypt -in chiper.bin -inkey private.key -out cleartext -passin pass:hello
```

