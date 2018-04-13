MacOS10.11.6编译安装Nginx With OpenSSL

# 1. 配置

```shell
sudo ./configure --prefix=/opt/feng/nginx-1.9.6 --with-http_ssl_module

# 直接使用上述命令，会提示如下错误
    ./configure: error: SSL modules require the OpenSSL library.
    You can either do not enable the modules, or install the OpenSSL library
    into the system, or build the OpenSSL library statically from the source
    with nginx by using --with-openssl=<path> option.
# 虽然Mac已经自带了OpenSSL，Nginx编译依然是找不到，需要源码
# 自行下载源码，带源码
sudo ./configure --prefix=/opt/feng/nginx-1.9.6 --with-http_ssl_module --with-openssl=../openssl-0.9.8zh

#正常
```
# 2. 编译

```shell
sudo make

# 会有这样的提示
    WARNING! If you wish to build 64-bit library, then you have to
         invoke './Configure darwin64-x86_64-cc' *manually*.
         You have about 5 seconds to press Ctrl-C to abort.
# 要装64位，需要手动调用。一开始没看明白，后来发现直接安装是无法完成的，报错。
    ld: symbol(s) not found for architecture x86_64
    clang: error: linker command failed with exit code 1 (use -v to see invocation)
    make[1]: *** [objs/nginx] Error 1
    make: *** [build] Error 2
# 看着是openssl与darwin的版本不兼容问题，后来发现是新版的openssl与nginx兼容问题。
# 解决方法，nginx configure完了之后
# 在当前 nginx 源码目录
cd objs
vi Makefile
# 找到类似这行
&& ./config --prefix=/Users/wid/Downloads/nginx-1.9.6/../openssl-0.9.8zh/.openssl no-shared  \

# 将 config 修改为 Configure darwin64-x86_64-cc, –prefix 之后的不用修改, 修改后的如:

&& ./Configure darwin64-x86_64-cc --prefix=/Users/wid/Downloads/nginx-1.8.0/../openssl-0.9.8zh/.openssl no-shared  \

# 修改保存, 反回到上级 nginx 源码目录继续执行 make 即可。
# 注意: 修改完 Makefile 文件后不要再次执行 configure, 会重新生成 Makefile 覆盖掉我们的修改。
```

如果你之前就`make`过，那你可能会看到另外一个错误`/bin/sh: ../openssl-0.9.8zh/.openssl/ssl/man/man3/hmac.3: Too many levels of symbolic links`。这是因为openssl多次编译导致创建软链接出错，需要到openssl源码目录下，删除文件夹`.openssl`，重新编译即可。


参考
> https://www.widlabs.com/article/mac-os-x-nginx-compile-symbol-not-found-for-architecture-x86_64
> http://homeway.me/2015/07/10/rebuild-osx-environment/