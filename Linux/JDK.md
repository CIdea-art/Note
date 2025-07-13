解压

如果是tar.gz文件将-xvf换成-xzvf

```shell
tar -xvf jdk-8u321-linux-x64.tar -C /sjj/install/
```

配置

```shell
vim /etc/profile
```

profile最后一行

```shell
# Java环境配置
export JAVA_HOME=/sjj/install/jdk1.8.0_321
export PATH=$PATH:$JAVA_HOME/bin
```

- 让配置生效

```shell
source /etc/profile
```

- 查看配置是否成功

```shell
java -version
```