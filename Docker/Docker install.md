# Centos

安装yum扩展工具

```shell
yum install yum-utils
```

添加yum仓库，更新配置

```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum update
```

安装

```shell
yum install docker-ce docker-ce-cli containerd.io
# 分别是docker服务器、docker命令行（类似MysqlCli连接工具）、容器化运行环境
```

开机启动

```bash
systemctl enable docker --now
# --now立即启动
```

配置阿里云镜像容器服务的镜像加速器

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://dn3p24fg.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```





yuque.com/leifengyang/oncloud/ox16bw

# 参考文献