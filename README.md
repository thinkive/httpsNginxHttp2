# httpsNginxHttp2
Http2.0在nginx上实践
1.在开始我们的任务之前，请先下载 OpenSSL ， pcre ， Zlib 跟 Nginx源码 ，并且全部解压到同一个目录，假设为nginx-http2/nginxsource ，这时候的代码结构如下：


2.准备好各种安装包之后，要解压 tar zxvf *，然后进入这个目录

# 先进入nginx源码目录

cd nginx-http2/nginxsource/nginx-1.9.14

# 配置Nginx源码

# --prefix：配置nginx最终安装目录，可以修改为你想要的目录

# --with-http_ssl_module 跟 --with-http_v2_module 必带，因为 HTTP2.0采用 HTTPS ，HTTPS 基于 SSL/TLS

sudo ./configure --prefix=/home/kingdom/env/nginx-http2/nginx-http2 --with-http_ssl_module --with-pcre=../pcre-8.38--with-openssl=../openssl-1.0.2g--with-zlib=../zlib-1.2.8--with-http_v2_module

> 输入管理员密码然后回车

>make&&makeinstall

这样就安装完成了

3.生成SSL/TLS安全证书

要配置HTTP 2.0就需要启动HTTPS，那就需要生成一个SSL/TLS证书，而OpenSSL正好可以做这件事情，买不起证书但自己造一个来临时用还是可以的，直接执行下面命令：

# 切换到nginx的根目录（我这里是home/kingdom/env/nginx-http2/nginx-http2，请根据上一步编译时候指定的--prefix对应修改）

cd home/kingdom/env/nginx-http2/nginx-http2

cd conf

# 利用openssl命令生成公钥跟密钥

openssl genrsa -des3 -passout pass:x -outcert.pass.key2048

openssl rsa -passin pass:x -incert.pass.key-outcert.key

openssl req -new-keycert.key-outcert.csr

openssl x509 -req -days365-incert.csr -signkey server.key-outcert.crt

# 将crt跟key合并生成pem文件

cat cert.crt cert.key> cert.pem

# 删除掉我们不需要用到的文件

rm -rf cert.crt cert.csr

4.修改Nginx配置

server {

      listen      7104;

      server_name  localhost;

      location / {

           root  html;

           index  index.html index.htm;

           #return 301 https://$host$remote_port$request_uri;

}

         error_page  500 502 503 504  /50x.html;

      location = /50x.html {

      root  html;

     }

}

server {

listen 443 ssl http2 default_server;

server_name  localhost;

ssl_certificate      cert.pem;

ssl_certificate_key  cert.key;

ssl_session_cache    shared:SSL:1m;

ssl_session_timeout  5m;

# 我用原始的下面这段启动报错了，所以注释掉改用了后面那段

#ssl_ciphers  HIGH:!aNULL:!MD5;

ssl_ciphers 'CHACHA20:EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH:ECDHE-RSA-AES128-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA128:DHE-RSA-AES128-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA128:ECDHE-RSA-AES128-SHA384:ECDHE-RSA-AES128-SHA128:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES128-SHA128:DHE-RSA-AES128-SHA128:DHE-RSA-AES128-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA384:AES128-GCM-SHA128:AES128-SHA128:AES128-SHA128:AES128-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4;';

ssl_prefer_server_ciphers  on;

location / {

root  html;

index  index.html index.htm;

}

}

这里我们设置了两种端口，一个是https的默认端口443，一个是http的端口7104，之所以这样设置就是为了后面比较两者的效率

5.性能比较

这里我们使用一个简单的例子来完成比较，一个简单的index页面，加载500张图片的效率，如下图所示


看下请求速度：

http2.0


http1.1




可以看出两者时间上面差了近两秒，当然这里仅仅是通过img的src进行加载的，如果有ajax请求的请情况，我们可以通过chrome自带的检测工具，如下图，http://192.168.14.183:7104/请求数已经到达6个，这是ajax将会被阻塞，而http2，至始至终只有一个请求数，可以实现无阻塞。




如何识别http2呢？

在chrome的请求信息中我们可以看出，如下图，这些都是http2特有的标示。




这里要注意的一点，在配置nginx，conf中https的配置后，一定要ps -ef|grep nginx 来查看进程数，然后kill -9 ；杀掉进程，之后在启动nginx，这里的nginx -s reload 是不起作用的。

作者：maclery
链接：https://www.jianshu.com/p/04a9c447e809
來源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。
