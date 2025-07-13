# 命令

## 详情

```bash
docker info
```

## 镜像-拉取

```bash
docker pull NAME[:tags]
```

## 镜像-列表

```basj
docker images
```

## 镜像-运行

若本地没有镜像，则会尝试拉取

```bash
docker run [OPTIONS] NAME[:tags] [cmd命令]
# --restart=always	重启
# -p 主机端口:容器端口	映射端口
# -d后台
# --name命名
# -it交互模式
# -e 变量名=值		设置环境变量
# --volumes
# --link单向连接
# -v, --宿主机路径:容器内路径	挂载
```

## 镜像-删除

```bash
docker rmi [-f] NAME[:tags]
```

## 容器-修改配置

```bash
docker update [OPTIONS] CONTAINER
# OPTIONS参照run
```

## 容器-列表

默认运行中

```bash
docker ps [-a]
# -a全部
```

## 容器-元数据

如虚拟ip 

```shell
docker inspect CONTAINER
```

## 容器-停止

```bash
docker stop [-f] IMAGE
# -f强制
```

## 容器-删除

```bash
docker rm [-f] 容器id
```

## 容器-执行内部指令

```shell
docker exec [-it] 容器id 指令
# 指令例如linux的/bin/bash
```

## 容器-暂停/解除

```bash
docker pause 容器id
docker unpause 容器id
```

## 容器-启动/重启

```bash
docker start
docker restart
```

## 网络

```shell
# 查看网络
docker network ls
# 创建
docker network create -d bridge network名称
# 连接
docker network connect network名称 容器名称
```

## 镜像-提交

```bash
docker commit [-OPTIONS] CONTAINER [REPOSITORY[:TAG]]
# -a	作者
# -c	
# -m	提交信息
# -p	提交时暂停容器
```

下次可直接启动这个保存的镜像

但依旧需要启动端口映射等参数，一键化需参照后续的dockerfile内容

## 镜像-保存

```bash
docker save [OPTIONS] IMAGE [IMAGE...]
# 默认的话输出到STDOUT
# Options:
#   -o, 输出到文件
```

## 镜像-加载

```bash
docker load [OPTIONS]
# -i, --input string		读取文件
# -q, 
```

## 镜像-改名

```bash
docker tag IMAGE NEWNAME
```

## 镜像-推送

推送的名字有要求

```bash
docker push REPO/IMAGE:TAG
```

## 其它

### 日志

```bash
docker logs IMAGE
```

### CP复制

```bash
docker cp [IMAGE:]PATH1 [IMAGE:]PATH2
```

# 参考文献

[Docker命令实战 (yuque.com)](https://www.yuque.com/leifengyang/oncloud/ox16bw)
