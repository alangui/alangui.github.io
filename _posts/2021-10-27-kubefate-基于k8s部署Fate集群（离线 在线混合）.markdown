---
layout:     post
title:      "快速搭建生产环境联邦学习集群"
subtitle:   "「KubeFATE」基于k8s部署Fate集群（在线+离线混合模式）"
date:       2021-10-27 12:00:00
author:     "小桂子"
header-img: "img/post-bg-linux.jpg"
tags:
- k8s
- KubeFATE
- 联邦学习
---

# 每台机器环境准备
```
0、所有机器处于同一局域网中（二层网络互通），如果是机器处于不通局域网中（三层网络互通）或者是有公有云机器，需要依照不同的网络环境搭建k8s集群
1、机器配置建议最低4C8G（一个完整包含所有组件的fate节点需要内存大约4G）
2、各节点时间同步
3、各节点域名解析 dns or hosts
4、关闭防火墙
5、安装kuberbete的主节点cup核心大于等于2
6、关闭swap，注释swap分区
swapoff -a
vi /etc/fstab
注释最后一行
7、使用root用户部署
8、所有机器主机名不能一样（建议提前永久修改所有机器主机名）
```

# 部署清单
```
1、机器配置(IP cpu 内存 挂载磁盘 主机名)
10.0.19.188 4C8G 50G master
10.0.19.189 4C8G 50G node-1
10.0.19.190 4C8G 50G node-2

2、k8s集群节点规划
master作为主节点、node-1、node-2作为工作节点

3、离线资源清单
文件名                      | 描述
-------------------------------------------------------------
docker-20.10.8.tgz          | docker离线安装包
docker.service              | 注册docker service文件
kube-flannel.yml            | k8s集群flannel网络插件资源清单
deploy.yaml                 | k8s安装ingress插件资源清单
recommended.yaml            | k8s安装dashboard插件资源清单
kubefate-k8s-v1.6.0.tar.gz  | 基于k8s部署fate集群资源清单
k8s-core.gz                 | k8s集群核心组件离线镜像文件
k8s-dashboard.gz            | k8s dashboard插件离线镜像文件
k8s-ingress.gz              | k8s ingreess插件离线镜像文件
kubefate-server.gz          | kubefate server服务离线镜像文件
fate.gz                     | fate节点所有组件离线镜像文件
```

# 每台机器安装docker
```
1、解压
tar -xvf docker-20.10.8.tgz

2、将解压出来的docker文件内容移动到 /usr/bin/ 目录下
cp docker/* /usr/bin/

3、将docker注册为service
cp docker.service /etc/systemd/system/

4、添加文件权限并启动docker 
chmod +x /etc/systemd/system/docker.service

5、重载unit配置文件 
systemctl daemon-reload

6、启动Docker 
systemctl start docker

7、设置开机自启
systemctl enable docker.service
 
8、验证
#查看docker状态
systemctl status docker
#查看Docker版本
docker -v

9、修改docker使用的cgroupDriver与k8s的kubelet一致
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

10、重载
systemctl daemon-reload
systemctl restart docker
docker info|grep "Cgroup Driver"
```

# 搭建k8s集群
## 主节点部署
```
1、配置yum仓库
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

2、安装k8s核心组件
# 安装指定版本
yum install -y kubelet-1.19.2 kubeadm-1.19.2 kubectl-1.19.2
rpm -ql kubelet
# 配置kubelet
vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
# 启动kubelet
systemctl start kubelet (失败，因为初始化未完成，缺少配置文件)
systemctl status kubelet
systemctl stop kubelet
# 设置kubelet开机启动
systemctl enable kubelet

3、初始化k8s集群
kubeadm init \
--image-repository registry.aliyuncs.com/google_containers \
--service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=Swap

#解释
指定镜像仓库registry.aliyuncs.com/google_containers，因为kubeadm默认从官网k8s.grc.io下载所需镜像，国内无法访问指定apiserver组件地址
主节点ip地址指定service的ip网段
指定pod的ip网段

4、拷贝配置
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

5、运行成功后保存最后的内容，此内容需要在其它节点加入Kubernetes集群时执行
kubeadm join 172.17.0.4:6443 --token 2zgii2.5albs909s5n4e923 \
    --discovery-token-ca-cert-hash sha256:64a352148fbd6db8e3db9a3ccec8f44a9c042513402c5996188a5fd804bdbad2
    
6、查看节点以及pod信息
kubectl get -h
kubectl get node
kubectl get componentstatus    
kubectl get ns
kubectl get pods -n kube-system
kubectl get pod --all-namespaces
# 解释
node节点为NotReady，因为corednspod没有启动，缺少网络pod
如果kubectl get cs命令检测组件的运行状态时有异常出现这种情况，是/etc/kubernetes/manifests/下的kube-controller-manager.yaml和kube-scheduler.yaml设置的默认端口是0导致的，解决方式是注释掉对应的port即可
cd /etc/kubernetes/manifests/
vim kube-controller-manager.yaml
vim kube-scheduler.yaml
systemctl restart kubelet.service
kubectl get cs

7、添加flannel网络组件
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
依照网络情况稍等一会儿（拉取网络插件耗时）再次查看pod和node
直到集群状态正常
```
## 依此添加机器节点至集群
```
1、配置yum仓库
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

2、安装k8s核心组件
# 安装指定版本
yum install -y kubelet-1.19.2 kubeadm-1.19.2 kubectl-1.19.2
rpm -ql kubelet
# 配置kubelet
vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
# 设置kubelet开机启动
systemctl enable kubelet

3、加入主节点
kubeadm join 172.17.0.4:6443 --token 2zgii2.5albs909s5n4e923 \
    --discovery-token-ca-cert-hash sha256:64a352148fbd6db8e3db9a3ccec8f44a9c042513402c5996188a5fd804bdbad2
    
4、在master节点查看node信息
kubectl get pods -n kube-system -o wide
```
## 如果需要重新搭建k8s流程
```
1、kubelet重新初始化
systemctl stop kubelet
rm -f /etc/kubernetes/manifests/*
rm -rf /var/lib/etcd/*
kubeadm reset

2、卸载k8s核心组件
yum remove kubelet-1.19.2 kubeadm-1.19.2 kubectl-1.19.2

3、重新搭建
```

# k8s集群安装ingress
```
1、关于版本说明
ingress-nginx v1.0 适用于Kubernetes版本 v1.19+（包括 v1.19）
Kubernetes-v1.22+需要使用ingress-nginx>=1.0，因为networking.k8s.io/v1beta已经移除

2、部署ingress-nginx
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/baremetal/deploy.yaml

3、由于墙的原因，修改源配置
sed -i 's@k8s.gcr.io/ingress-nginx/controller:v1.0.0\(.*\)@willdockerhub/ingress-nginx-controller:v1.0.0@' deploy.yaml
sed -i 's@k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0\(.*\)$@hzde0128/kube-webhook-certgen:v1.0@' deploy.yaml

4、修改部署配置（一行修改，二行新增）
...
    spec:
      hostNetwork: true # 新增
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: willdockerhub/ingress-nginx-controller:v1.0.0
          imagePullPolicy: IfNotPresent
          lifecycle:
...

...
# Source: ingress-nginx/templates/controller-deployment.yaml
apiVersion: apps/v1
#kind: Deployment   # 注释
kind: DaemonSet     # 新增
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.1
...

...
args:
  - /nginx-ingress-controller
  - --publish-service=$(POD_NAMESPACE)/ingress-nginx-dev-v1-test-controller
  - --election-id=ingress-controller-leader
  - --controller-class=k8s.io/ingress-nginx
  - --configmap=$(POD_NAMESPACE)/ingress-nginx-dev-v1-test-controller
  - --validating-webhook=:8443
  - --validating-webhook-certificate=/usr/local/certificates/cert
  - --validating-webhook-key=/usr/local/certificates/key
  - --watch-ingress-without-class=true  # 新增
...

5、部署
kubectl apply -f deploy.yaml

6、验证
# 检查 pod 
kubectl  get  pods -n ingress-nginx  -o wide 
# 检查service
kubectl get svc -n ingress-nginx
# 检查端口
netstat  -pntl |grep 443
# demo
https://help.aliyun.com/document_detail/86536.html
```

# k8s集群安装dashboard
```
1、安装
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
kubectl apply -f recommended.yaml
kubectl get pods -n kubernetes-dashboard
kubectl get svc -n kubernetes-dashboard
kubectl patch svc kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}' -n kubernetes-dashboard

2、浏览器访问
https://nodeIp:nodePort

3、使用kubefate-admin的token
参考下文，KubeFATE部署好之后，获取kubefate-admin Token
kubectl get secret -n kube-fate
kubectl describe secret kubefate-admin-token-t9vd6 -n kube-fate
```

# 联邦集群搭建
## 官方部署参考
```
https://github.com/FederatedAI/KubeFATE/wiki/%E4%BD%BF%E7%94%A8KubeFATE%E5%9C%A8Kubernetes%E4%B8%8A%E9%83%A8%E7%BD%B2FATE%E9%9B%86%E7%BE%A4#exchange%E6%A8%A1%E5%BC%8F
```
## 部署KubeFATE
```
1、下载KubeFATE安装包
wget https://github.com/FederatedAI/KubeFATE/releases/download/v1.6.0/kubefate-k8s-v1.6.0.tar.gz

2、解压之后就可以使用
mkdir -p kubefate
tar -zxvf kubefate-k8s-v1.6.0.tar.gz -C kubefate
chmod +x ./kubefate
cd kubefate
mv ./kubefate /usr/bin

3、安装KubeFATE server
kubectl apply -f rbac-config.yaml
kubectl apply -f kubefate.yaml

4、配置hosts文件(可以登陆k8s dashboard查看kube-fate空间 pod状态以及ingress的信息)
ip = kubefate ingress入口地址：Endpoints
echo "ip kubefate.net"  >> /etc/hosts
解释
KubeFATE命令行是KubeFATE server的API调用实现，KubeFATE命令行与KubeFATE server的通信是通过Ingress暴露的URL kubefate.net来实现

5、验证KubeFATE成功
kubefate version
```
## 建立工作目录
```
# 建议一个Fate集群建一个目录（称为工作目录），放所有Fate节点yaml文件
mkdir hn-jr

# 复制执行kubefate指令所要的认证文件至工作目录
cp config.yaml hn-jr/

# 进入工作目录
cd hn-jr
```
## 使用KubeFate部署两个工作节点（party）
1、新建namespace
```
kubectl create namespace fate-9000-jr
kubectl create namespace fate-8000-jr
```
2、新建fate工作节点yaml文件
```
vi fate-9000-jr.yaml
```
```
name: fate-9000-jr
namespace: fate-9000-jr
chartName: fate
chartVersion: v1.6.0
partyId: 9000
registry: ""
imageTag: ""
pullPolicy:
imagePullSecrets:
- name: myregistrykey
persistence: false
istio:
  enabled: false
modules:
  - rollsite
  - clustermanager
  - nodemanager
  - mysql
  - python
  - fateboard
  - client

backend: eggroll

rollsite:
  exchange:
    ip: 10.0.19.189
    port: 30000
```
```
vi fate-8000-jr.yaml
```
```
name: fate-8000-jr
namespace: fate-8000-jr
chartName: fate
chartVersion: v1.6.0
partyId: 8000
registry: ""
imageTag: ""
pullPolicy:
imagePullSecrets:
- name: myregistrykey
persistence: false
istio:
  enabled: false
modules:
  - rollsite
  - clustermanager
  - nodemanager
  - mysql
  - python
  - fateboard
  - client

backend: eggroll

rollsite:
  exchange:
    ip: 10.0.19.189
    port: 30000
```
3、部署工作节点
```
kubefate cluster install -f fate-9000-jr.yaml
kubefate cluster install -f fate-8000-jr.yaml

yaml文件配置解释
1. 工作节点采用exchange通信模式（星型网络拓扑结构）
2. exchange节点rollsite组件服务提供类型NodePort service
3. 工作节点rollsite exchange ip配置可以是k8s集群任意一台节点机器ip
```
4、使用dashboard查看运行情况
## 使用KubeFATE部署FATE集群-通信路由节点（exchange）
1、新建namespace
```
kubectl create namespace fate-exchange-jr
```
2、新建exchange节点yaml文件（可以参考官方配置）
```
vi fate-exchange-jr.yaml
```
```
name: fate-exchange-jr
namespace: fate-exchange-jr
chartName: fate-exchange
chartVersion: v1.6.0
partyId: 1
registry: ""
imageTag: "1.6.0-release"
pullPolicy:
imagePullSecrets:
- name: myregistrykey
persistence: false
istio:
  enabled: false
modules:
  - rollsite

rollsite:
  type: NodePort
  nodePort: 30000
  partyList:
  - partyId: 9000
    partyIp: 10.106.224.144
    partyPort: 9370
  - partyId: 8000
    partyIp: 10.103.34.210
    partyPort: 9370
```
3、部署exchange节点
```
kubefate cluster install -f fate-exchange-jr.yaml

yaml文件配置解释
1. partyList是各个fate节点路由表信息配置
2. partyIp是k8s集群内fate节点rollsite组件pod代理service VIP
3. partyPort是k8s集群内fate节点rollsite组件对外提供通信的默认端口
```
4、使用dashboard查看运行情况
## 部署在线推理服务节点（fate-serving）
1、新建namespace
```
kubectl create namespace fate-8000-serving-jr
kubectl create namespace fate-9000-serving-jr
```
2、分别新建fate-8000-jr、fate-9000-jr节点对应的serving yaml文件
```
vi fate-8000-serving-jr.yaml
```
```
name: fate-8000-serving-jr
namespace: fate-8000-serving-jr
chartName: fate-serving
chartVersion: v2.0.0
partyId: 8000
registry: ""
pullPolicy:
persistence: false
istio:
  enabled: false
modules:
  - servingProxy
  - servingRedis
  - servingServer

servingProxy:
  nodePort: 30001
  ingerssHost: 8000.serving-proxy.kubefate.net
  partyList:
  - partyId: 9000
    partyIp: 10.0.19.189
    partyPort: 30002
  nodeSelector: {}

servingServer:
  fateflow:
    ip: 10.111.122.171
    port: 9380
  subPath: ""
  existingClaim: ""
  storageClass: "serving-server"
  accessMode: ReadWriteOnce
  size: 1Gi
  nodeSelector: {}

servingRedis:
  password: fate_dev
  nodeSelector: {}
  subPath: ""
  existingClaim: ""
  storageClass: "serving-redis"
  accessMode: ReadWriteOnce
  size: 1Gi

```
```
vi fate-9000-serving-jr.yaml
```
```
name: fate-9000-serving-jr
namespace: fate-9000-serving-jr
chartName: fate-serving
chartVersion: v2.0.0
partyId: 9000
registry: ""
pullPolicy:
persistence: false
istio:
  enabled: false
modules:
  - servingProxy
  - servingRedis
  - servingServer

servingProxy:
  nodePort: 30002
  ingerssHost: 9000.serving-proxy.kubefate.net
  partyList:
  - partyId: 8000
    partyIp: 10.0.19.189
    partyPort: 30001
  nodeSelector: {}

servingServer:
  fateflow:
    ip: 10.101.85.211
    port: 9380
  subPath: ""
  existingClaim: ""
  storageClass: "serving-server"
  accessMode: ReadWriteOnce
  size: 1Gi
  nodeSelector: {}

servingRedis:
  password: fate_dev
  nodeSelector: {}
  subPath: ""
  existingClaim: ""
  storageClass: "serving-redis"
  accessMode: ReadWriteOnce
  size: 1Gi
```
3、部署fate-serving
```
kubefate cluster install -f fate-8000-serving-jr.yaml
kubefate cluster install -f fate-9000-serving-jr.yaml

解释
1. 各个fate-serving采用集群直连通信模式（网状网络拓扑模式）
2. partyList是其它Fate-serving servingProxy路由清单，彼此通过Service NodePort类型通信
3、在线推理服务servingServer组件需要配置对应fate节点调度组件的通信ip和port, 通信方式采用Service VIP方式，fateFlow配置Fate节点Services(fateflow-client)的vip信息（建议通过dashboard查看其具体vip地址）
```
4、使用dashboard查看运行情况

5、更新Fate节点对应的serving配置
```
vi fate-8000-jr.yaml
```
```
name: fate-8000-jr
namespace: fate-8000-jr
chartName: fate
chartVersion: v1.6.0
partyId: 8000
registry: ""
imageTag: ""
pullPolicy:
imagePullSecrets:
- name: myregistrykey
persistence: false
istio:
  enabled: false
modules:
  - rollsite
  - clustermanager
  - nodemanager
  - mysql
  - python
  - fateboard
  - client

backend: eggroll

rollsite:
  exchange:
    ip: 10.0.19.189
    port: 30000

servingIp: 10.102.206.50
servingPort: 8000
```
```
vi fate-9000-jr.yaml
```
```
name: fate-9000-jr
namespace: fate-9000-jr
chartName: fate
chartVersion: v1.6.0
partyId: 9000
registry: ""
imageTag: ""
pullPolicy:
imagePullSecrets:
- name: myregistrykey
persistence: false
istio:
  enabled: false
modules:
  - rollsite
  - clustermanager
  - nodemanager
  - mysql
  - python
  - fateboard
  - client

backend: eggroll

rollsite:
  exchange:
    ip: 10.0.19.189
    port: 30000

servingIp: 10.106.205.209
servingPort: 8000
```
6、update部署
```
kubefate cluster update -f fate-8000-jr.yaml
kubefate cluster update -f fate-9000-jr.yaml

解释
1. 在线推理服务servingServer组件只对内部servingProxy和对应的Fate节点提供端口服务(8000端口)，外部fate如果需要进行联合在线推理，必须通过servingProxy组件通信
2. 更新Fate就是为了增加Fate访问其servingServer的Ip和端口，一对Fate与Serving的通信基于VIP地址
```
## 验证（使用k8s dashboard操作）
1、使用fate-8000-jr做为guest, 使用fate-9000-jr做为host
2、进入fate-9000-jr python容器（提交host方训练数据）
```
cd fate_flow
vi examples/upload_host.json
```
```
{
  "file": "examples/data/breast_hetero_host.csv",
  "head": 1,
  "partition": 10,
  "work_mode": 1,
  "namespace": "fate_flow_test_breast",
  "table_name": "breast"
}
```
```
python fate_flow_client.py -f upload -c examples/upload_host.json
```
3、进入fate-8000-jr python容器

提交host方训练数据
```
cd fate_flow
vi examples/upload_guest.json
```
```
{
  "file": "examples/data/breast_hetero_guest.csv",
  "head": 1,
  "partition": 10,
  "work_mode": 1,
  "namespace": "fate_flow_test_breast",
  "table_name": "breast"
}
```
```
python fate_flow_client.py -f upload -c examples/upload_guest.json
```
guest和host联合训练
```
vi examples/test_hetero_lr_job_conf.json
```
```
{
    "initiator": {
        "role": "guest",
        "party_id": 8000
    },
    "job_parameters": {
        "work_mode": 1
    },
    "role": {
        "guest": [8000],
        "host": [9000],
        "arbiter": [9000]
    },
    "role_parameters": {
        "guest": {
            "args": {
                "data": {
                    "train_data": [{"name": "breast", "namespace": "fate_flow_test_breast"}]
                }
            },
            "dataio_0":{
                "with_label": [true],
                "label_name": ["y"],
                "label_type": ["int"],
                "output_format": ["dense"]
            }
        },
        "host": {
            "args": {
                "data": {
                    "train_data": [{"name": "breast", "namespace": "fate_flow_test_breast"}]
                }
            },
             "dataio_0":{
                "with_label": [false],
                "output_format": ["dense"]
            }
        }
    },
    "algorithm_parameters": {
        "hetero_lr_0": {
            "penalty": "L2",
            "optimizer": "rmsprop",
            "alpha": 0.01,
            "max_iter": 3,
            "batch_size": 320,
            "learning_rate": 0.15,
            "init_param": {
                "init_method": "random_uniform"
            }
        }
    }
}
```
```
vi examples/test_hetero_lr_job_dsl.json
```
```
{
    "components" : {
        "dataio_0": {
            "module": "DataIO",
            "input": {
                "data": {
                    "data": [
                        "args.train_data"
                    ]
                }
            },
            "output": {
                "data": ["train"],
                "model": ["dataio"]
            },
            "need_deploy": true
         },
        "hetero_feature_binning_0": {
            "module": "HeteroFeatureBinning",
            "input": {
                "data": {
                    "data": [
                        "dataio_0.train"
                    ]
                }
            },
            "output": {
                "data": ["train"],
                "model": ["hetero_feature_binning"]
            }
        },
        "hetero_feature_selection_0": {
            "module": "HeteroFeatureSelection",
            "input": {
                "data": {
                    "data": [
                        "hetero_feature_binning_0.train"
                    ]
                },
                "isometric_model": [
                    "hetero_feature_binning_0.hetero_feature_binning"
                ]
            },
            "output": {
                "data": ["train"],
                "model": ["selected"]
            }
        },
        "hetero_lr_0": {
            "module": "HeteroLR",
            "input": {
                "data": {
                    "train_data": ["hetero_feature_selection_0.train"]
                }
            },
            "output": {
                "data": ["train"],
                "model": ["hetero_lr"]
            }
        },
        "evaluation_0": {
            "module": "Evaluation",
            "input": {
                "data": {
                    "data": ["hetero_lr_0.train"]
                }
            },
            "output": {
                "data": ["evaluate"]
            }
        }
    }
}
```
```
python fate_flow_client.py -f submit_job -d examples/test_hetero_lr_job_dsl.json -c examples/test_hetero_lr_job_conf.json
```
查看训练结果
```
python fate_flow_client.py -f query_task -j ${jobId} | grep f_status
```
注册预测模型接口
```
vi examples/publish_load_model.json
```
```
{
    "initiator": {
        "party_id": "8000",
        "role": "guest"
    },
    "role": {
        "guest": ["8000"],
        "host": ["9000"],
        "arbiter": ["9000"]
    },
    "job_parameters": {
        "work_mode": 1,
        "model_id": "arbiter-10000#guest-9999#host-10000#model",
        "model_version": "202003060553168191842"
    }
}
```
```
python fate_flow_client.py -f load -c examples/publish_load_model.json
vi examples/bind_model_service.json
```
```
{
    "service_id": "test",
    "initiator": {
        "party_id": "8000",
        "role": "guest"
    },
    "role": {
        "guest": ["8000"],
        "host": ["9000"],
        "arbiter": ["9000"]
    },
    "job_parameters": {
        "work_mode": 1,
        "model_id": "arbiter-8000#guest-7000#host-8000#model",
        "model_version": "202110210648038681602"
    }
}

```
```
python fate_flow_client.py -f bind -c examples/bind_model_service.json
```
测试在线推理预测
```
Fate-Serving服务入口就是servingProxy的8059端口，其对外暴露的端口有类型为NodePort的service和ingress
通过fateboard找到访问servingProxy的ip和port
```
```
 curl -X POST -H 'Content-Type: application/json' -i 'http://nodeIp:nodePort/federation/v1/inference' --data '{
  "head": {
    "serviceId": "test"
  },
  "body": {
    "featureData": {
      "x0": 0.254879,
      "x1": -1.046633,
      "x2": 0.209656,
      "x3": 0.074214,
      "x4": -0.441366,
      "x5": -0.377645,
      "x6": -0.485934,
      "x7": 0.347072,
      "x8": -0.287570,
      "x9": -0.733474
    },
    "sendToRemoteFeatureData": {
      "id": "123"
    }
  }
}'
```
