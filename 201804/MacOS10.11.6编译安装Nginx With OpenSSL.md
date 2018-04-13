MacOS10.11.6编译安装Nginx With OpenSSL

# 1. 编译配置

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

# 3. 编译安装

```
make install
# 如果以前已经安装过了，只是要加入SSL模块，可以直接复制nginx/objs里面编译好的nginx到之前安装的目录，覆盖即可。
```

# 4. 配置SSL证书

### 说明：
只是为了本地测试nginx添加ssl功能，因此就本地生成了密钥，证书，但这样证书是不被信任的，但是也是安全的。

### 实现：

##### 1. 使用如下命令并根据提示输入信息，生成证书

```
openssl genrsa -des3 -out localhost.key 1024 //创建自身密钥
openssl req -new -key localhost.key -out localhost.csr  //通过密钥生成相应CSR申请文件
openssl rsa -in localhost.key -out localhost_nopass.key //生成浏览器浏览网页时不需要输入密码的密钥
openssl x509 -req -days 365 -in localhost.csr -signkey localhost.key -out localhost.crt
//生成证书
```

##### 2. 在nginx的server配置中添加如下配置：

```
server {
    listen 443;
    server_name www.nobody.com nobody.com;
    ssl on;
    ssl_certificate /usr/local/nginx/conf/localhost.crt;
    ssl_certificate_key /usr/local/nginx/conf/localhost_nopass.key;
}
```
注：如果HTTP和HTTPS虚拟主机的功能是一致的，可以配置一个虚拟主机，既处理HTTP请求，又处理HTTPS请求。 配置的方法是删除ssl on的指令，并在*:443端口添加参数ssl：

```
server {
    listen              80;
    listen              443 ssl;
    server_name www.nobody.com nobody.com;
    ssl_certificate     /usr/local/nginx/conf/localhost.crt;
    ssl_certificate_key /usr/local/nginx/conf/localhost_nopass.key;
    ...
}
```
在0.8.21版本以前，只有添加了default参数的监听端口才能添加ssl参数：listen 443 default ssl;

现在生成的证书是不受信任的，如果需要受信任的证书，需要证书颁发机构颁发（需要用钱解决）。

# 5. 强制使用HTTPS（HTTP跳转到HTTPS）

配置如下
```
server {
    listen       80;
    listen       443 ssl;
	server_name  localhost 127.0.0.1;
	ssl on; #只允许https请求访问
    ssl_certificate     /usr/local/nginx/conf/localhost.crt;
    ssl_certificate_key /usr/local/nginx/conf/localhost_nopass.key;
    error_page 497 https://$host$request_uri;
}
```
上述配置中，由于使用了`ssl on`，所以只允许`https`访问，nginx会给出497错误（非标准错误，浏览器看到的是400错误）。然后使用`error_page`将该错误重定向到`https`即可。
也可以使用判断`$scheme`的方式。或者配置两个`server`的方式，`rewrite`或者`return 301`。

```
if ($scheme = http) {
    return 301 https://$host$request_uri;
}
```


# 参考
> https://www.widlabs.com/article/mac-os-x-nginx-compile-symbol-not-found-for-architecture-x86_64
> http://homeway.me/2015/07/10/rebuild-osx-environment/
> http://coolnull.com/1245.html
> http://nginx.org/en/docs/http/configuring_https_servers.html


