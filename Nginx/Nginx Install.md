## 安装

### windows

[nginx：下载](https://nginx.org/en/download.html)

解压进入文件夹，启动并生成`nginx.pid`

```
nginx -c conf/nginx.conf
```

> 如果没指定配置，之后其它命令会报[error]CreateFile

打开任意浏览器，输入`localhost`，可以看到nginx的主页。

### linux

1、获取安装包

```
wget https://nginx.org/download/nginx-1.21.6.tar.gz
```

> 如果没有wget
>
> ```
> yum -y install wget
> ```

解压

```
tar -xvzf nginx-1.21.6.tar.gz
```

2、下载依赖包

```
yum install --downloadonly --downloaddir=/soft/nginx/ gcc-c++
yum install --downloadonly --downloaddir=/soft/nginx/ pcre pcre-devel4
yum install --downloadonly --downloaddir=/soft/nginx/ zlib zlib-devel
yum install --downloadonly --downloaddir=/soft/nginx/ openssl openssl-devel
```

3、安装依赖包

```
rpm -ivh --nodeps \*.rpm
```

4、环境初始化

```
./configure --prefix=${nginx_path}
```

5、编译并安装

```
make && make install
```

6、修改配置

```
vi conf/nginx.conf
```

7、启动

```
sbin/nginx -c conf/nginx.conf
```

8、检测运行

```
 ps aux | grep nginx
```

## 添加模块

添加

```
./configure --prefix=/soft/nginx/ --add-module=/soft/nginx/cache_purge/ngx_cache_purge-2.3/
```

重新编译

```
make
```

从生成的`objs`目录中，重新复制一个`Nginx`的启动文件到原来的位置：

```
cp objs/nginx /soft/nginx/sbin/nginx  
```

