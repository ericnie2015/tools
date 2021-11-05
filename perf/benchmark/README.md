> 本文作为**Flomesh服务网格性能测试**一文的补充，详细讲述了如何进行Flomesh性能环境的搭建，欢迎感兴趣的同学阅读。

## 环境准备

### 需要4台服务器，规格可以参考下表
  | 序号 |  阿里云型号   |  CPU   | 内存 |        角色        |
  | :--: | :-----------: | :----: | :--: | :----------------: |
  |  1   | ecs.c6.large  | 2 vCPU |  4G  |     k3s master     |
  |  2   | ecs.g6.xlarge | 4 vCPU | 16G  |     k3s node1      |
  |  3   | ecs.g6.xlarge | 4 vCPU | 16G  |     k3s node2      |
  |  4   | ecs.c6.xlarge | 4 vCPU |  8G  | 测试机，运行fortio |



## k3s master节点

### 安装k3s master节点，在`master`节点上执行

```sh
export INSTALL_K3S_VERSION=v1.19.13+k3s1
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644
```

### 获取TOKEN，以便后续用来将node节点加入集群

```sh
root@k3s-master:~# cat /var/lib/rancher/k3s/server/node-token
K10d28ace1cc9e0f7b1f12641ed32c69bf6ae4ed21b796bb597181ce27e52b7cfb5::server:cd178f253f1ff4798e509c89efdaab1d
```

> 请记录此**TOKEN**，后续在创建node节点加入集群时会用到。

### 记得将master标记为不可调度节点

```sh
kubectl cordon k3s-master
```



## k3s node节点

### 安装k3s-agent，并加入集群

#### 在两个`node`节点上分别执行

```sh
export INSTALL_K3S_VERSION=v1.19.13+k3s1
curl -sfL https://get.k3s.io | K3S_URL=https://<master_IP>:6443 K3S_TOKEN=<join_token> sh -
```

请将`<master_IP>`和`<join_token>`替换成真实的值。

## 部署Prometheus
在`master`节点上执行以下步骤

### 下载kube-prometheus，请参照[Compability Guide](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.7#compatibility)下载合适的版本

```sh
cd
git clone -b release-0.7 https://github.com/prometheus-operator/kube-prometheus.git
```

### 安装kube-prometheus

```sh
cd ~/kube-prometheus
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```

等待namespace `monitoring`下的所有POD变为running状态。

### 对外暴露Grafana控制台和Prometheus，为了简化，仅通过port forwarding来实现

```sh
kubectl --namespace monitoring --address 0.0.0.0 port-forward svc/grafana 3000 &
kubectl --namespace monitoring --address 0.0.0.0 port-forward svc/prometheus-k8s 9090 &
```

如果是在云主机上，不要忘了打开相应的安全组端口。

## 部署Flomesh服务网格

在`master`节点上执行以下步骤

### 下载测试部署文件

```sh
cd
git clone https://github.com/flomesh-io/tools.git
```

### 安装cert-manager

```sh
cd ~/tools/perf/benchmark
kubectl apply -f cert-manager.yaml
```

等待namespace `cert-manager`下的POD全部变为running状态，然后进行下一步。


### 安装pipy-operator

```sh
kubectl apply -f pipy-operator.yaml
```

等待namespace `flomesh`下的pod变成running状态后再执行下一步。

   

## 部署bookinfo demo

### 配置proxy-profile

```sh
kubectl apply -f proxy-profile.yaml
```


### 为default namespace添加label

```sh
kubectl label ns default flomesh.io/inject=true
```


### 安装Fortio Server

- 部署fortio server：

```sh
kubectl apply -f fortio-server.yaml
```

- 等待fortio-server服务的POD变成RUNNING状态：

```sh
❯ kubectl get po
NAME                             READY   STATUS    RESTARTS   AGE
fortio-server-866889f5c9-g2st9   2/2     Running   0          5m55s
```

- 将服务扩容到100个实例

```sh
kubectl scale --replicas=100 deploy fortio-server
```

- 等待扩容完毕，取决于所用的硬件，可能需要10分钟或者更久。

```sh
root@k3s-master:~/tools/perf/benchmark# kubectl get deploy 
NAME              READY     UP-TO-DATE   AVAILABLE   AGE
fortio-server     100/100   100          100         20m
```

等待READY变成100/100，请检查pod的log，确认reviews服务已经启动。



## 测试客户端节点

在`测试客户端`节点上执行以下操作

### 安装fortio

- 下载并安装

```sh
wget https://github.com/fortio/fortio/releases/download/v1.17.0/fortio_1.17.0_amd64.deb
dpkg -i fortio_1.17.0_amd64.deb 
```

- 测试是否安装成功

```sh
fortio version
```

### 下载测试脚本

```sh
sudo apt-get update
sudo apt-get install git -y
cd
git clone https://github.com/flomesh-io/tools.git 
```

### 安装Python

- 安装python 3.8

```sh
sudo apt-get update
sudo apt-get install software-properties-common -y
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get install python3.8 -y
```

- 验证pyhton安装是否成功

```sh
root@test-client:~# python3 --version
Python 3.8.10
```

### 安装pipenv

```sh
pip install pipenv
```

### 准备Python环境

```sh
cd ~/tools/perf/benchmark
pipenv --three
pipenv shell
pipenv install
```



## 运行测试

在`测试客户端`节点上执行以下操作，整个测试约需要**45**分钟左右。

```sh
cd ~/tools/perf/benchmark
./benchmark.sh
```

最终在`~/tools/perf/benchmark`下会生成lantency-p50.png，lantency-p90.png和lantency-p99.png，以及每步所生成的json数据。

