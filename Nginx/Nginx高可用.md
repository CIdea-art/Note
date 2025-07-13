## 重启检测脚本

check_nginx_pid_restart.sh

```
#!/bin/sh  
# 通过ps指令查询后台的nginx进程数，并将其保存在变量nginx_number中  
nginx_number=`ps -C nginx --no-header | wc -l`  
# 判断后台是否还有Nginx进程在运行  
if [ $nginx_number -eq 0 ];then  
    # 如果后台查询不到`Nginx`进程存在，则执行重启指令  
    /soft/nginx/sbin/nginx -c /soft/nginx/conf/nginx.conf  
    # 重启后等待1s后，再次查询后台进程数  
    sleep 1  
    # 如果重启后依旧无法查询到nginx进程，异常需要人工排查  
    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then  
        # 将keepalived主机下线，将虚拟IP漂移给从机，从机上线接管Nginx服务  
        systemctl stop keepalived.service  
    fi  
fi  
```

需要更改编码格式，并赋予执行权限

```
vi check_nginx_pid_restart.sh  
:set fileformat=unix # 在vi命令里面执行，修改编码格式  
:set ff # 查看修改后的编码格式  
chmod +x check_nginx_pid_restart.sh  
```



## Keepalived

编辑配置文件`/keepalived/etc/keepalived/keepalived.conf`

主机

```
global_defs {  
  # 自带的邮件提醒服务，建议用独立的监控或第三方SMTP，也可选择配置邮件发送。  
  notification_email {  
    root@localhost  
  }  
  notification_email_from root@localhost  
  smtp_server localhost  
  smtp_connect_timeout 30  
  # 高可用集群主机身份标识(集群中主机身份标识名称不能重复，建议配置成本机IP)  
  router_id 192.168.12.129   
}  
  
# 定时运行的脚本文件配置  
vrrp_script check_nginx_pid_restart {  
  # 之前编写的nginx重启脚本的所在位置  
  script "/scripts/keepalived/check_nginx_pid_restart.sh"   
  # 每间隔3秒执行一次  
  interval 3  
  # 如果脚本中的条件成立，重启一次则权重-20  
  weight -20  
}  
  
# 定义虚拟路由，VI_1为虚拟路由的标示符（可自定义名称）  
vrrp_instance VI_1 {  
  # 当前节点的身份标识：用来决定主从（MASTER为主机，BACKUP为从机）  
  state MASTER  
  # 绑定虚拟IP的网络接口，根据自己的机器的网卡配置  
  interface ens33   
  # 虚拟路由的ID号，主从两个节点设置必须一样  
  virtual_router_id 121  
  # 填写本机IP  
  mcast_src_ip 192.168.12.129  
  # 节点权重优先级，主节点要比从节点优先级高  
  priority 100  
  # 优先级高的设置nopreempt，解决异常恢复后再次抢占造成的脑裂问题  
  nopreempt  
  # 组播信息发送间隔，两个节点设置必须一样，默认1s（类似于心跳检测）  
  advert_int 1  
  authentication {  
    auth_type PASS  
    auth_pass 1111  
  }  
  # 将track_script块加入instance配置块  
  track_script {  
    # 执行Nginx监控的脚本  
    check_nginx_pid_restart  
  }  

  virtual_ipaddress {  
    # 虚拟IP(VIP)，也可扩展，可配置多个。  
    192.168.12.111  
  }  
} 
```

备机

拷贝Ng脚本。修改`Keepalived`配置节点身份表示、主备标识、权重

```
global_defs {  
  # 高可用集群主机身份标识(集群中主机身份标识名称不能重复，建议配置成本机IP)  
  router_id 192.168.12.129
  ...
}
# 定义虚拟路由，VI_1为虚拟路由的标示符（可自定义名称）  
vrrp_instance VI_1 {  
  # 当前节点的身份标识：用来决定主从（MASTER为主机，BACKUP为从机）  
  state BACKUP
  # 节点权重优先级，主节点要比从节点优先级高  
  priority 90
  ...
} 
```

如果安装`keepalived`时，是自定义的安装位置，因此需要拷贝一些文件到系统目录中

> TODO：是否可用快捷方式代替

```
mkdir /etc/keepalived/  
cp /soft/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/  
cp /soft/keepalived/keepalived-2.2.4/keepalived/etc/init.d/keepalived /etc/init.d/  
cp /soft/keepalived/etc/sysconfig/keepalived /etc/sysconfig/  
```

将`keepalived`加入系统服务并设置开启自启动，然后测试启动是否正常：

```
chkconfig keepalived on  
systemctl daemon-reload  
systemctl enable keepalived.service  #开机自启
systemctl start keepalived.service  #启动
```

其他命令：

```bash
systemctl disable keepalived.service # 禁止开机自动启动  
systemctl restart keepalived.service # 重启keepalived  
systemctl stop keepalived.service # 停止keepalived  
tail -f /var/log/messages # 查看keepalived运行时日志  
```





参考资料

[Nginx从安装到高可用](https://mp.weixin.qq.com/s/nEjzYIxxazG8T_eR7i6wHA)