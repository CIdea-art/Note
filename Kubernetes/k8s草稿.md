简介

微服务部署困难



概念

Pod，调度资源的最小单位

云平台上的容器化主机应用



架构

Control Plane（控制平面）。

- 为集群做出全局决策。比如资源的调度。

- 检测和响应集群事件。

![](resources\kubernetes-cluster-architecture.svg)

cloud-controller-manager

etcd：一致且高可用的键值存储，用作 Kubernetes 所有集群数据的后台数据库。

kube-api-server（API 服务器）：是 Kubernetes 控制平面的前端。负责公开了 Kubernetes API，负责处理接受请求的工作。

kube-scheduler：Pods注册管理，负责监视新创建的、未指定运行节点（node）的 Pods， 并选择节点来让 Pod 在上面运行。

kube-controller-manager：负责运行[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)进程。



Node（工作机器）

kubelet：节点Pod的监控。接收一组通过各类机制提供给它的 PodSpec，确保这些 PodSpec 中描述的容器处于运行状态且健康。

Pod：

CRI

kube-proxy：网络代理组件。可选，可用三方代替。



入门

搭建方式：二进制；命令行；

kuberts：封装k8sAPI的指令工具

自我修复：



进阶



组件实战



运维



项目实战









# Namespace

命名空间

隔离资源

命令行创建

```bash
kubectl get ns
kubectl create ns $NAME
kubectl delete ns $NAME
kubectl get pod -n $NS
```

配置文件创建

```yaml
# 怎么判断哪个版本新，字符串的比较方法？
apiVersion: v1
# 类型
kind: Namespace
metadata:
  name: ${NS_NAME}
```

然后应用

```bash
kubectl apply -f $CONF
kubectl delete -f $CONF
# -f, ????
```

# Pod

豌豆荚，运行多个容器

```bash
kubectl get pod [-A] [-owide]
```



```bash
kubectl run $NAME --image=$IMAGE
kubectl delete pod $NAME
# -n, 指定命名空间，未指定则默认操作default
```



```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: ${NAME}
  name: ${NAME}
  namespace: ${NS_NAME}
spec:
  containers:
  - image: ${IMAGE}
    name: ${NAME}
```



```bash
# 描述详细信息
kubectl describe pod $NAME
# Pod日志
kubectl logs [-f] $NAME
```



```bash
# bash交互
kubectl exec -it $NAME -- /bin/bash
```



