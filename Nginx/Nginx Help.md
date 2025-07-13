## 命令

展示版本号和配置

```
nginx -V
```

*检测配置文件是否正常* 

```
nginx -t -c conf/nginx.conf
```

修改配置后平滑重启

```
nginx -s reload -c conf/nginx.conf
```

优雅关闭Nginx，会在执行完当前的任务后再退出

```
nginx -s quit
```

强制终止Nginx，不管当前是否有任务在执行

```
nginx -s stop
```

开放代理的端口

```
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-ports
```
开机自启

```shell
systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
```

## 配置

### 监听

```
server {         
	listen	80;         
	server_name localhost;
}
```

### 代理

```nginx
server {
	# 示例
	location / { 
        root  html; 
        # 配置一下index的地址，最后加上index.ftl
        index index.html index.htm index.jsp index.ftl; 
        proxy_set_header Host $host; 
        proxy_set_header X-Real-IP $remote_addr; 
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
        # 请求交给名为nginx_boot的upstream上
        # 若地址以'/'结尾，则不会带上location地址
        proxy_pass http://nginx_boot;
  	}
	
	# 静态
	location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|txt|js|css){ 
  		root  /soft/nginx/static_resources; 
  		expires 7d; 
	}
} 

```

location匹配规则，其实就是组合加正则

- `~`代表匹配时区分大小写

### 负载

```
upstream nginx_boot{ 
	# 30s内检查心跳发送两次包，未回复就代表该机器宕机，请求分发权重比为1:2
	server 192.168.0.000:8080 weight=100 max_fails=2 fail_timeout=30s;  
	server 192.168.0.000:8090 weight=200 max_fails=2 fail_timeout=30s; 
	# 这里的IP请配置成你WEB服务所在的机器IP
} 
```

### 静态资源压缩

`ngx_http_gzip_module`（内置）、`ngx_http_gzip_static_module`、`ngx_http_gunzip_module`

`ngx_http_gzip_module`

```
http{
    # 开启压缩机制
    gzip on;
    # 指定会被压缩的文件类型(也可自己配置其他类型)
    gzip_types text/plain application/javascript text/css application/xml text/javascript image/jpeg image/gif image/png;
    # 设置压缩级别，越高资源消耗越大，但压缩效果越好
    gzip_comp_level 5;
    # 在头部中添加Vary: Accept-Encoding（建议开启）
    gzip_vary on;
    # 处理压缩请求的缓冲区数量和大小
    gzip_buffers 16 8k;
    # 对于不支持压缩功能的客户端请求不开启压缩机制
    gzip_disable "MSIE [1-6]\."; # 低版本的IE浏览器不支持压缩
    # 设置压缩响应所支持的HTTP最低版本
    gzip_http_version 1.1;
    # 设置触发压缩的最小阈值
    gzip_min_length 2k;
    # 关闭对后端服务器的响应结果进行压缩
    gzip_proxied off;
}
```

- gzip_proxied

  - `off`：关闭`Nginx`对后台服务器的响应结果进行压缩。

  - `expired`：如果响应头中包含`Expires`信息，则开启压缩。

  - `no-cache`：如果响应头中包含`Cache-Control:no-cache`信息，则开启压缩。

  - `no-store`：如果响应头中包含`Cache-Control:no-store`信息，则开启压缩。

  - `private`：如果响应头中包含`Cache-Control:private`信息，则开启压缩。

  - `no_last_modified`：如果响应头中不包含`Last-Modified`信息，则开启压缩。

  - `no_etag`：如果响应头中不包含`ETag`信息，则开启压缩。

  - `auth`：如果响应头中包含`Authorization`信息，则开启压缩。

  - `any`：无条件对后端的响应结果开启压缩机制。

### 缓冲区

**用来解决两个连接之间速度不匹配造成的问题**

- `proxy_buffering`：是否启用缓冲机制，默认为`on`关闭状态。
- `client_body_buffer_size`：设置缓冲客户端请求数据的内存大小。
- `proxy_buffers`：为每个请求/连接设置缓冲区的数量和大小，默认`4 4k/8k`。
- `proxy_buffer_size`：设置用于存储响应头的缓冲区大小。
- `proxy_busy_buffers_size`：在后端数据没有完全接收完成时，`Nginx`可以将`busy`状态的缓冲返回给客户端，该参数用来设置`busy`状态的`buffer`具体有多大，默认为`proxy_buffer_size*2`。
- `proxy_temp_path`：当内存缓冲区存满时，可以将数据临时存放到磁盘，该参数是设置存储缓冲数据的目录。
- `path`是临时目录的路径。
  - 语法：`proxy_temp_path path;` path是临时目录的路径
- `proxy_temp_file_write_size`：设置每次写数据到临时文件的大小限制。
- `proxy_max_temp_file_size`：设置临时的缓冲目录中允许存储的最大容量。

非缓冲参数项：

- `proxy_connect_timeout`：设置与后端服务器建立连接时的超时时间。
- `proxy_read_timeout`：设置从后端服务器读取响应数据的超时时间。
- `proxy_send_timeout`：设置向后端服务器传输请求数据的超时时间。

```
http{  
    proxy_buffering on;  
    proxy_connect_timeout 10;  
    proxy_read_timeout 120;  
    proxy_send_timeout 10;  
    client_body_buffer_size 512k;  
    proxy_buffers 4 64k;  
    proxy_buffer_size 16k;  
    proxy_busy_buffers_size 128k;  
    proxy_temp_file_write_size 128k;  
    proxy_temp_path /soft/nginx/temp_buffer;  
}  
```

### 缓存机制

属于代理缓存的一种。

**proxy_cache_path**：代理缓存的路径。

```
proxy_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];
```

- `path`：缓存的路径地址。
- `levels`：缓存存储的层次结构，最多允许三层目录。
- `use_temp_path`：是否使用临时目录。
- `keys_zone`：指定一个共享内存空间来存储热点Key(1M可存储8000个Key)。
- `inactive`：设置缓存多长时间未被访问后删除（默认是十分钟）。
- `max_size`：允许缓存的最大存储空间，超出后会基于LRU算法移除缓存，Nginx会创建一个Cache manager的进程移除数据，也可以通过purge方式。
- `manager_files`：manager进程每次移除缓存文件数量的上限。
- `manager_sleep`：manager进程每次移除缓存文件的时间上限。
- `manager_threshold`：manager进程每次移除缓存后的间隔时间。
- `loader_files`：重启Nginx载入缓存时，每次加载的个数，默认100。
- `loader_sleep`：每次载入时，允许的最大时间上限，默认200ms。
- `loader_threshold`：一次载入后，停顿的时间间隔，默认50ms。
- `purger`：是否开启purge方式移除数据。
- `purger_files`：每次移除缓存文件时的数量。
- `purger_sleep`：每次移除时，允许消耗的最大时间。
- `purger_threshold`：每次移除完成后，停顿的间隔时间。

> purger是收费的，可安装`ngx_cache_purge`模块

**proxy_cache**：开启或关闭代理缓存，开启时需要指定一个共享内存区域。

```
proxy_cache zone | off;
```

zone为内存区域的名称，即上面中keys_zone设置的名称。

**「proxy_cache_key」**：定义如何生成缓存的键。

语法：

```
proxy_cache_key string;
```

string为生成Key的规则，如`$scheme$proxy_host$request_uri`。

**「proxy_cache_valid」**：缓存生效的状态码与过期时间。

语法：

```
proxy_cache_valid [code ...] time;
```

code为状态码，time为有效时间，可以根据状态码设置不同的缓存时间。

例如：`proxy_cache_valid 200 302 30m;`

**「proxy_cache_min_uses」**：设置资源被请求多少次后被缓存。

语法：

```
proxy_cache_min_uses number;
```

number为次数，默认为1。

**「proxy_cache_use_stale」**：当后端出现异常时，是否允许Nginx返回缓存作为响应。

语法：

```
proxy_cache_use_stale error;
```

error为错误类型，可配置`timeout|invalid_header|updating|http_500...`。

**「proxy_cache_lock」**：对于相同的请求，是否开启锁机制，只允许一个请求发往后端。

语法：

```
proxy_cache_lock on | off;
```

**「proxy_cache_lock_timeout」**：配置锁超时机制，超出规定时间后会释放请求。

```
proxy_cache_lock_timeout time;
```

**「proxy_cache_methods」**：设置对于那些HTTP方法开启缓存。

语法：

```
proxy_cache_methods method;
```

method为请求方法类型，如GET、HEAD等。

**「proxy_no_cache」**：定义不存储缓存的条件，符合时不会保存。

语法：

```
proxy_no_cache string...;
```

string为条件，例如`$cookie_nocache $arg_nocache $arg_comment;`

**「proxy_cache_bypass」**：定义不读取缓存的条件，符合时不会从缓存中读取。

语法：

```
proxy_cache_bypass string...;
```

和上面`proxy_no_cache`的配置方法类似。

**「add_header」**：往响应头中添加字段信息。

语法：

```
add_header fieldName fieldValue;
```

**「$upstream_cache_status」**：记录了缓存是否命中的信息，存在多种情况：

- `MISS`：请求未命中缓存。
- `HIT`：请求命中缓存。
- `EXPIRED`：请求命中缓存但缓存已过期。
- `STALE`：请求命中了陈旧缓存。
- `REVALIDDATED`：Nginx验证陈旧缓存依然有效。
- `UPDATING`：命中的缓存内容陈旧，但正在更新缓存。
- `BYPASS`：响应结果是从原始服务器获取的。

```
http{  
    # 设置缓存的目录，并且内存中缓存区名为hot_cache，大小为128m，  
    # 三天未被访问过的缓存自动清楚，磁盘中缓存的最大容量为2GB。  
    proxy_cache_path /soft/nginx/cache levels=1:2 keys_zone=hot_cache:128m inactive=3d max_size=2g;  
      
    server{  
        location / {  
            # 使用名为hot_cache的缓存空间  
            proxy_cache hot_cache;  
            # 对于200、206、304、301、302状态码的数据缓存1天  
            proxy_cache_valid 200 206 304 301 302 1d;  
            # 对于其他状态的数据缓存30分钟  
            proxy_cache_valid any 30m;  
            # 定义生成缓存键的规则（请求的url+参数作为key）  
            proxy_cache_key $host$uri$is_args$args;  
            # 资源至少被重复访问三次后再加入缓存  
            proxy_cache_min_uses 3;  
            # 出现重复请求时，只让一个去后端读数据，其他的从缓存中读取  
            proxy_cache_lock on;  
            # 上面的锁超时时间为3s，超过3s未获取数据，其他请求直接去后端  
            proxy_cache_lock_timeout 3s;  
            # 对于请求参数或cookie中声明了不缓存的数据，不再加入缓存  
            proxy_no_cache $cookie_nocache $arg_nocache $arg_comment;  
            # 在响应头中添加一个缓存是否命中的状态（便于调试）  
            add_header Cache-status $upstream_cache_status;  
        }  
    }  
} 
```

### 黑白名单

直接写入配置

```
allow xxx.xxx.xxx.xxx; # 允许指定的IP访问，可以用于实现白名单。  
deny xxx.xxx.xxx.xxx; # 禁止指定的IP访问，可以用于实现黑名单。 
```

或写入额外的配置文件并导入

```
http{  
    # 屏蔽该文件中的所有IP  
    include /soft/nginx/IP/BlocksIP.conf;   
 	server{  
    	location xxx {  
        	# 某一系列接口只开放给白名单中的IP  
        	include /soft/nginx/IP/blockip.conf;   
    	}  
 	}  
}
```

http和server的范围不同

### 跨域

> 产生原因：
>
> 产生跨域问题的主要原因就在于 **「同源策略」** ，为了保证用户信息安全，防止恶意网站窃取数据，同源策略是必须的，否则`cookie`可以共享。由于`http`无状态协议通常会借助`cookie`来实现有状态的信息记录，例如用户的身份/密码等，因此一旦`cookie`被共享，那么会导致用户的身份信息被盗取。
>
> 同源策略主要是指三点相同，**「协议+域名+端口」** 相同的两个请求，则可以被看做是同源的，但如果其中任意一点存在不同，则代表是两个不同源的请求，同源策略会**限制了不同源之间的资源交互**。

```
location / {  
    # 允许跨域的请求，可以自定义变量$http_origin，*表示所有 
    add_header 'Access-Control-Allow-Origin' *;  
    # 允许携带cookie请求  
    add_header 'Access-Control-Allow-Credentials' 'true';  
    # 允许跨域请求的方法：GET,POST,OPTIONS,PUT  
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT';  
    # 允许请求时携带的头部信息，*表示所有  
    add_header 'Access-Control-Allow-Headers' *;  
    # 允许发送按段获取资源的请求  
    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';  
    # 一定要有！！！否则Post请求无法进行跨域！  
    # 在发送Post跨域请求前，会以Options方式发送预检请求，服务器接受时才会正式请求  
    if ($request_method = 'OPTIONS') {  
        add_header 'Access-Control-Max-Age' 1728000;  
        add_header 'Content-Type' 'text/plain; charset=utf-8';  
        add_header 'Content-Length' 0;  
        # 对于Options方式的请求返回204，表示接受跨域请求  
        return 204;  
    }  
}  
```

### 防盗链

> 盗链即是指外部网站引入当前网站的资源对外展示

Nginx根据`Referer`判断是否为本站的资源引用请求，如果不是则不允许访问。

> `Referer`，该字段主要描述了当前请求是从哪儿发出的

```
valid_referers none | blocked | server_names | string ...;
```

- `none`：表示接受没有`Referer`字段的`HTTP`请求访问。
- `blocked`：表示允许`http://`或`https//`以外的请求访问。
- `server_names`：资源的白名单，这里可以指定允许访问的域名。
- `string`：可自定义字符串，支配通配符、正则表达式写法。

```
# 在动静分离的location中开启防盗链机制  
location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|txt|js|css){  
    # 最后面的值在上线前可配置为允许的域名地址  
    valid_referers blocked 192.168.12.129;  
    if ($invalid_referer) {  
        # 可以配置成返回一张禁止盗取的图片  
        # rewrite   ^/ http://xx.xx.com/NO.jpg;  
        # 也可直接返回403  
        return   403;  
    }  
      
    root   /soft/nginx/static_resources;  
    expires 7d;  
}  
```

> 也有专门的三方模块`ngx_http_accesskey_module`

但是防盗链机制也无法解决爬虫伪造`referers`信息的这种方式抓取数据。

### 大文件

- client_max_body_size，请求体允许的最大体积
- client_header_timeout，发送请求头的超时时间
- client_body_timeout，读取请求体的超时时间
- proxy_read_timeout，请求被后盾服务器读取时，Nginx等待的最长时间
- proxy_send_timeout，设置后盾项Nginx返回响应时的超时时间

上述配置仅是作为代理层需要配置的，因为最终客户端传输文件还是直接与后端进行交互，这里只是把作为网关层的`Nginx`配置调高一点，调到能够“容纳大文件”传输的程度。当然，`Nginx`中也可以作为文件服务器使用，但需要用到一个专门的第三方模块`nginx-upload-module`，如果项目中文件上传的作用处不多，那么建议可以通过`Nginx`搭建，毕竟可以节省一台文件服务器资源。但如若文件上传/下载较为频繁，那么还是建议额外搭建文件服务器，并将上传/下载功能交由后端处理。

### SSL证书

`HTTPS`为了确保通信安全，服务端需配置对应的数字证书。当项目使用`Nginx`作为网关时，那么证书在`Nginx`中也需要配置。

完整的数字证书文件有三个

- `.crt`：数字证书文件，`.crt`是`.pem`的拓展文件，因此有些人下载后可能没有。
- `.key`：服务器的私钥文件，及非对称加密的私钥，用于解密公钥传输的数据。
- `.pem`：`Base64-encoded`编码格式的源证书文本文件，可自行根需求修改拓展名。

```nginx
server {  
    # 监听HTTPS默认的443端口  
    listen 443;  
    # 配置自己项目的域名  
    server_name www.xxx.com;  
    # 打开SSL加密传输  
    ssl on;  
    # 输入域名后，首页文件所在的目录  
    root html;  
    # 配置首页的文件名  
    index index.html index.htm index.jsp index.ftl;  
    # 配置自己下载的数字证书  
    ssl_certificate  certificate/xxx.pem;  
    # 配置自己下载的服务器私钥  
    ssl_certificate_key certificate/xxx.key;  
    # 停止通信时，加密会话的有效期，在该时间段内不需要重新交换密钥  
    ssl_session_timeout 5m;  
    # TLS握手时，服务器采用的密码套件  
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;  
    # 服务器支持的TLS版本  
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  
    # 开启由服务器决定采用的密码套件  
    ssl_prefer_server_ciphers on;  
  
    location / {  
        ....  
    }  
}  
  
# ---------HTTP请求转HTTPS-------------  
server {  
    # 监听HTTP默认的80端口  
    listen 80;  
    # 如果80端口出现访问该域名的请求  
    server_name www.xxx.com;  
    # 将请求改写为HTTPS（这里写你配置了HTTPS的域名）  
    rewrite ^(.*)$ https://www.xxx.com;  
}  
```

### 限流

```nginx
# 基于ip地址的限制
#1）zone=iplimit 引用limit_rep_zone中的zone变量
#2）burst=2  设置一个大小为2的缓冲区域，当大量请求到来，请求数量超过限流频率时，将其放入缓冲区域
#3）nodelay   缓冲区满了以后，直接返回503异常
limit_req zone=iplimit burst=2 nodelay;
#基于服务器级别做限流
limit_req zone=serverlimit burst=2 nodelay;
#基于ip地址的链接数量做限流  最多保持100个链接
limit_conn zone=perip 100;
#基于服务器的连接数做限流 最多保持100个链接
limit_conn zone=perserver 1;
#配置request的异常返回504（默认为503）
limit_req_status 504;
limit_conn_status 504;

#前100m不限制速度
limit_rate_affer 100m;
#限制速度为256k
limit_rate 256k;
```

# 参考文献

[Nginx 实战：Centos7下Nginx各种方式安装 | 弟弟快看-教程 (ddkk.com)](https://www.ddkk.com/zhuanlan/server/nginx/2/3.html)
