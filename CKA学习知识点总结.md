# CKA（Certified Kubernetes Administrator）学习知识点总结

> 基于 CNCF / Linux Foundation 官方考纲整理，适用于 Kubernetes v1.34 ~ v1.35 环境。  
> 考试形式：**纯实操**（命令行完成任务），**2 小时**，约 **17–25 道题**，及格线 **66%**。

---

## 目录

- [一、考试概览](#一考试概览)
- [二、五大考纲域与权重](#二五大考纲域与权重)
- [三、集群架构、安装与配置（25%）](#三集群架构安装与配置25)
- [四、工作负载与调度（15%）](#四工作负载与调度15)
- [五、服务与网络（20%）](#五服务与网络20)
- [六、存储（10%）](#六存储10)
- [七、故障排查（30%）](#七故障排查30)
- [八、必背命令速查](#八必背命令速查)
- [九、YAML 模板要点](#九yaml-模板要点)
- [十、学习路线与备考建议](#十学习路线与备考建议)
- [十一、考前检查清单](#十一考前检查清单)

---

## 一、考试概览

| 项目 | 说明 |
|------|------|
| 考试类型 | 在线监考、性能实操（Performance-based） |
| 时长 | 120 分钟 |
| 及格分数 | 66% |
| 题目数量 | 约 17–25 道（每次略有不同） |
| K8s 版本 | 随 K8s 小版本更新（约发布后 4–8 周对齐，当前约 v1.34/v1.35） |
| 操作系统 | Ubuntu Linux 终端 |
| 允许查阅 | [kubernetes.io/docs](https://kubernetes.io/docs)、[kubernetes.io/blog](https://kubernetes.io/blog)、[github.com/kubernetes](https://github.com/kubernetes) |
| 证书有效期 | 2 年 |
| 费用 | $445（含一次免费重考） |
| 语言 | 英语、日语、**简体中文** |

**核心特点：**

- 没有选择题，全部在真实/模拟集群上操作
- 每题独立计分，**先做会的、再回头攻坚**
- 官方文档是唯一允许的参考资料，熟练检索文档是核心竞争力

---

## 二、五大考纲域与权重

| 域 | 权重 | 核心能力 |
|----|------|----------|
| **故障排查 Troubleshooting** | **30%** | 排查节点、控制面、Pod、网络、日志 |
| **集群架构、安装与配置** | **25%** | RBAC、kubeadm、升级、etcd 备份恢复、HA、Helm/Kustomize |
| **服务与网络** | **20%** | Service、Ingress、Gateway API、NetworkPolicy、CoreDNS |
| **工作负载与调度** | **15%** | Deployment、ConfigMap/Secret、调度约束、静态 Pod |
| **存储** | **10%** | PV/PVC、StorageClass、访问模式、回收策略 |

> **备考优先级：** 故障排查 + 集群架构 = **55%**，是过关的关键。

```
故障排查 ██████████████████████████████ 30%
集群架构 █████████████████████████ 25%
服务网络 ████████████████████ 20%
工作负载 ███████████████ 15%
存储     ██████████ 10%
```

---

## 三、集群架构、安装与配置（25%）

### 3.1 Kubernetes 架构基础

**控制平面（Control Plane）组件：**

| 组件 | 职责 |
|------|------|
| **kube-apiserver** | 集群 API 入口，唯一与 etcd 通信的组件 |
| **etcd** | 分布式 KV 存储，保存集群所有状态 |
| **kube-scheduler** | 为 Pod 选择合适 Node |
| **kube-controller-manager** | 运行各类控制器（Deployment、Node 等） |
| **cloud-controller-manager** | 与云厂商集成（考试环境通常不涉及） |

**工作节点（Worker Node）组件：**

| 组件 | 职责 |
|------|------|
| **kubelet** | 管理本节点 Pod 生命周期，向 API Server 注册 |
| **kube-proxy** | 维护 Service 网络规则（iptables/IPVS） |
| **容器运行时** | containerd / CRI-O（通过 CRI 接口与 kubelet 交互） |

**重要概念：**

- 控制面组件在 kubeadm 集群中通常以 **Static Pod** 形式运行（manifest 位于 `/etc/kubernetes/manifests/`）
- kubelet 是 **systemd 服务**，不是 Pod
- 扩展接口：**CRI**（容器运行时）、**CNI**（网络）、**CSI**（存储）

---

### 3.2 RBAC（基于角色的访问控制）

**四个核心对象：**

| 对象 | 作用域 | 说明 |
|------|--------|------|
| **Role** | Namespace | 命名空间内权限 |
| **ClusterRole** | Cluster | 集群级权限 |
| **RoleBinding** | Namespace | 将 Role/ClusterRole 绑定到主体 |
| **ClusterRoleBinding** | Cluster | 将 ClusterRole 绑定到主体（集群范围生效） |

**主体（Subject）类型：** User、Group、**ServiceAccount**

**关键规则：**

- `ClusterRole` + `RoleBinding` → 权限**仅限该 Namespace**
- `ClusterRole` + `ClusterRoleBinding` → 权限**全集群生效**
- 权限由 **verbs**（get/list/watch/create/update/patch/delete）+ **resources** 组成

**常用命令：**

```bash
# 创建 Role
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev

# 创建 RoleBinding（绑定 ServiceAccount）
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --serviceaccount=dev:my-sa \
  -n dev

# 创建 ClusterRole
kubectl create clusterrole node-reader --verb=get,list --resource=nodes

# 创建 ClusterRoleBinding
kubectl create clusterrolebinding read-nodes \
  --clusterrole=node-reader \
  --serviceaccount=dev:my-sa

# 验证权限（考试高频）
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:my-sa
kubectl auth can-i list nodes --as=system:serviceaccount:dev:my-sa
kubectl auth can-i create deployments --as=john -n prod
```

---

### 3.3 kubeadm 集群安装

**控制平面节点：**

```bash
# 初始化（需提前关闭 swap、加载内核模块、配置 sysctl）
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# 配置 kubeconfig
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 安装 CNI 插件（如 Calico）
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

**工作节点加入：**

```bash
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# 忘记 join 命令时重新生成
kubeadm token create --print-join-command
```

**节点前置条件（可能考修复）：**

```bash
swapoff -a                                    # 关闭 swap
modprobe overlay && modprobe br_netfilter     # 加载内核模块
# sysctl: net.bridge.bridge-nf-call-iptables = 1
systemctl status containerd                   # 容器运行时正常
```

---

### 3.4 高可用（HA）控制平面

**两种 etcd 拓扑：**

| 模式 | 说明 |
|------|------|
| **Stacked etcd** | etcd 与控制面同节点（常见，kubeadm 默认） |
| **External etcd** | etcd 独立部署（更高隔离性） |

**HA 要点：**

- 多个控制平面节点 + 负载均衡器
- `kubeadm init --control-plane-endpoint=<LB-IP>:6443`
- 额外控制面节点：`kubeadm join ... --control-plane --certificate-key <key>`

---

### 3.5 kubeadm 集群升级（高频考点）

**控制平面节点升级顺序：**

```bash
# 1. 升级 kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=<version>
sudo apt-mark hold kubeadm

# 2. 查看升级计划
sudo kubeadm upgrade plan

# 3. 执行升级（控制面专用）
sudo kubeadm upgrade apply v1.35.0

# 4. 升级 kubelet 和 kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=<version> kubectl=<version>
sudo apt-mark hold kubelet kubectl

# 5. 重启 kubelet
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

**工作节点升级顺序：**

```bash
# 1. 在控制面 drain 节点
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. SSH 到工作节点，升级 kubeadm/kubelet/kubectl（版本与控制面一致）
sudo apt-get install -y kubeadm=<version> kubelet=<version> kubectl=<version>

# 3. 执行节点升级（工作节点专用）
sudo kubeadm upgrade node

# 4. 重启 kubelet
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# 5. 在控制面 uncordon
kubectl uncordon <node-name>
```

> **记忆要点：** 控制面用 `kubeadm upgrade apply`，工作节点用 `kubeadm upgrade node`。

---

### 3.6 etcd 备份与恢复（几乎必考）

**证书路径来源：** 查看 `/etc/kubernetes/manifests/etcd.yaml` 中的 `--cert-file`、`--key-file`、`--trusted-ca-file`

**备份：**

```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**验证备份：**

```bash
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table
```

**恢复：**

```bash
# 1. 恢复到新目录
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# 2. 修改 /etc/kubernetes/manifests/etcd.yaml
#    --data-dir 改为 /var/lib/etcd-restored
#    hostPath.path 同步修改

# 3. 等待 static pod 自动重启（kubectl 可能短暂不可用，属正常）
```

**证书管理：**

```bash
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
```

---

### 3.7 Helm 与 Kustomize

**Helm（包管理器）：**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-nginx bitnami/nginx -n default
helm list -A
helm upgrade my-nginx bitnami/nginx --set replicaCount=3
helm uninstall my-nginx
```

**Kustomize（内置 kubectl）：**

```bash
kubectl apply -k ./overlays/production
kubectl kustomize ./base    # 预览渲染结果
```

**快速生成 YAML（考试技巧）：**

```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run test --image=busybox --dry-run=client -o yaml > pod.yaml
```

---

### 3.8 CRD 与 Operator

- **CRD（CustomResourceDefinition）**：扩展 Kubernetes API，定义自定义资源类型
- **Operator**：基于 CRD + Controller 模式，自动化管理复杂应用
- 考试要求：**理解概念**，能识别 CRD 资源，可能涉及安装 Helm Chart 形式的 Operator

```bash
kubectl get crd
kubectl get <custom-resource> -A
```

---

## 四、工作负载与调度（15%）

### 4.1 核心工作负载类型

| 类型 | 用途 | 关键特性 |
|------|------|----------|
| **Pod** | 最小调度单元 | 一个或多个容器共享网络/存储 |
| **ReplicaSet** | 维持 Pod 副本数 | 通常由 Deployment 管理，不直接创建 |
| **Deployment** | 无状态应用 | 滚动更新、回滚、扩缩容 |
| **DaemonSet** | 每节点一个 Pod | 日志采集、监控 Agent |
| **StatefulSet** | 有状态应用 | 稳定网络标识、有序部署 |
| **Job** | 一次性任务 | 运行至完成 |
| **CronJob** | 定时任务 | 基于 Cron 表达式 |

---

### 4.2 Deployment 滚动更新与回滚

```bash
kubectl create deployment webapp --image=nginx:1.26 --replicas=3

# 更新镜像（注意：第二个参数是容器名，不是 Deployment 名）
kubectl set image deployment/webapp nginx=nginx:1.27

kubectl rollout status deployment/webapp
kubectl rollout history deployment/webapp
kubectl rollout undo deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=2
kubectl rollout pause deployment/webapp
kubectl rollout resume deployment/webapp
```

**滚动更新策略：**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 更新期间最多超出期望副本数
      maxUnavailable: 0 # 更新期间最多不可用副本数（0 = 零停机）
```

| 策略 | 行为 |
|------|------|
| `RollingUpdate` | 逐步替换（默认） |
| `Recreate` | 先删后建（有停机） |

---

### 4.3 ConfigMap 与 Secret

**创建：**

```bash
kubectl create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=info

kubectl create secret generic db-creds \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=changeme

kubectl create configmap nginx-conf --from-file=nginx.conf
```

**三种注入方式：**

1. **环境变量（全部键）：** `envFrom` + `configMapRef` / `secretRef`
2. **环境变量（单个键）：** `env` + `valueFrom.configMapKeyRef`
3. **挂载为文件：** `volumes.configMap` / `volumes.secret`

**注意：**

- Secret 默认 Base64 编码，**不是加密**
- ConfigMap 挂载为目录时会**覆盖**整个目录，单文件用 `subPath`
- 修改 ConfigMap/Secret 后，挂载为文件的 Pod 会自动更新；环境变量方式需重启 Pod

---

### 4.4 资源限制与配额

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"       # 250 millicores = 0.25 CPU
  limits:
    memory: "128Mi"   # 超出 → OOMKilled
    cpu: "500m"       # 超出 → CPU 节流（throttle）
```

| 概念 | 说明 |
|------|------|
| **requests** | 调度依据，决定 Pod 分配到哪个 Node |
| **limits** | 运行时上限 |
| **LimitRange** | 命名空间内 Pod/Container 默认与约束 |
| **ResourceQuota** | 命名空间资源总量上限 |

---

### 4.5 自动扩缩容（HPA）

```bash
kubectl autoscale deployment webapp --min=2 --max=10 --cpu-percent=80
kubectl get hpa
```

- 需要 **metrics-server** 提供 CPU/内存指标
- CKA 考察深度较浅，了解基本用法即可

---

### 4.6 Pod 调度

**nodeSelector（最简单）：**

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**Node Affinity（更灵活）：**

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["zone-a"]
```

**Taints 与 Tolerations：**

```bash
# 给节点打污点
kubectl taint nodes node1 key=value:NoSchedule

# 移除污点（末尾加 -）
kubectl taint nodes node1 key=value:NoSchedule-
```

| Effect | 行为 |
|--------|------|
| `NoSchedule` | 不调度新 Pod（已有 Pod 不受影响） |
| `PreferNoSchedule` | 尽量不调度 |
| `NoExecute` | 不调度 + 驱逐已有 Pod |

**Pod Priority 与 Preemption：**

- 高优先级 Pod 可抢占低优先级 Pod 的资源
- `PriorityClass` 定义优先级等级

---

### 4.7 静态 Pod（Static Pod）

- 由 **kubelet 直接管理**，不经过 API Server
- manifest 目录默认：`/etc/kubernetes/manifests/`
- 控制面组件（apiserver、scheduler、controller-manager、etcd）均为 Static Pod
- `kubectl get pods` 中名称带节点后缀（如 `kube-apiserver-node1`）
- **无法通过 kubectl delete 删除**，需删除 manifest 文件

```bash
# 查找 static pod 路径
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# 创建静态 Pod
sudo vi /etc/kubernetes/manifests/my-static-pod.yaml
```

---

### 4.8 节点维护

```bash
kubectl cordon node1          # 标记不可调度
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data  # 驱逐 Pod
kubectl uncordon node1        # 恢复调度
```

---

## 五、服务与网络（20%）

### 5.1 Service 类型

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| **ClusterIP** | 集群内部虚拟 IP（默认） | 集群内服务互访 |
| **NodePort** | 在每个 Node 上开放端口（30000–32767） | 开发测试、无 LB 环境 |
| **LoadBalancer** | 云厂商负载均衡器 + NodePort | 生产外部访问 |
| **ExternalName** | CNAME 到外部 DNS | 代理外部服务 |

**端口概念：**

- `port`：Service 端口（客户端访问）
- `targetPort`：Pod 容器端口
- `nodePort`：Node 端口（仅 NodePort/LoadBalancer）

```bash
kubectl expose deployment webapp --port=80 --target-port=8080
kubectl expose deployment webapp --port=80 --target-port=8080 --type=NodePort
```

**Endpoints：**

```bash
kubectl get endpoints <service-name>
# Endpoints 为空 → Service selector 与 Pod labels 不匹配
```

---

### 5.2 DNS 与 CoreDNS

**DNS 命名规则：**

| 类型 | 格式 |
|------|------|
| Service | `<service>.<namespace>.svc.cluster.local` |
| Pod | `<pod-ip-dashed>.<namespace>.pod.cluster.local` |
| 同 Namespace 短名 | `<service>` |
| 跨 Namespace | `<service>.<namespace>` |

**排查命令：**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get configmap coredns -n kube-system -o yaml
kubectl run test-dns --image=busybox:1.36 --rm -it -- nslookup kubernetes.default
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

### 5.3 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx          # 必须指定 IngressClass
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix           # Exact | Prefix | ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

**要点：**

- Ingress 资源本身不处理流量，需要 **Ingress Controller**（如 nginx-ingress）
- `pathType: Prefix` 是最常用类型
- v1.22+ 使用 `ingressClassName` 替代旧 annotation

---

### 5.4 Gateway API（v1.35 新增重点）

Gateway API 是 Ingress 的下一代替代方案：

| 资源 | 角色 |
|------|------|
| **GatewayClass** | 定义 Gateway 实现类型 |
| **Gateway** | 基础设施层，定义监听器（端口/协议） |
| **HTTPRoute** | 应用层路由规则 |
| **GRPCRoute / TCPRoute / UDPRoute** | 其他协议路由 |

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "myapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 80
```

---

### 5.5 NetworkPolicy（网络策略）

**核心规则：**

- 一旦 Pod 被 NetworkPolicy 选中，**默认拒绝**所有未明确允许的流量
- 需要集群 CNI 支持 NetworkPolicy（Calico、Cilium 等，Flannel 不支持）
- 添加 Egress 规则时，**必须允许 DNS（UDP 53）**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - ports:                          # 允许 DNS（必加！）
    - protocol: UDP
      port: 53
```

**选择器逻辑（易错）：**

```yaml
# AND：同一 from 条目中 namespaceSelector + podSelector 同时满足
ingress:
- from:
  - namespaceSelector:
      matchLabels: { env: prod }
    podSelector:
      matchLabels: { app: frontend }

# OR：多个 from 条目，满足任一即可
ingress:
- from:
  - namespaceSelector:
      matchLabels: { env: prod }
- from:
  - podSelector:
      matchLabels: { app: frontend }
```

---

### 5.6 CNI 插件

| 插件 | NetworkPolicy | 说明 |
|------|---------------|------|
| **Calico** | ✅ | 考试最常用，推荐默认选择 |
| **Flannel** | ❌ | 简单 overlay，不支持 NetworkPolicy |
| **Cilium** | ✅ | eBPF 实现，功能强大 |
| **Weave Net** | ✅ | 支持 NetworkPolicy |

```bash
ls /etc/cni/net.d/
kubectl get pods -n kube-system | grep calico
```

---

### 5.7 Pod 间连通性

```bash
kubectl get pods -o wide
kubectl exec pod-a -- wget -qO- --timeout=2 http://<pod-b-ip>:8080
kubectl exec pod-a -- curl http://<service-name>.<namespace>.svc.cluster.local
```

---

## 六、存储（10%）

### 6.1 核心概念

| 对象 | 作用域 | 说明 |
|------|--------|------|
| **PersistentVolume (PV)** | Cluster | 集群中的存储资源 |
| **PersistentVolumeClaim (PVC)** | Namespace | 用户对存储的请求 |
| **StorageClass** | Cluster | 动态供应存储的模板 |

**绑定条件（三者必须同时满足）：**

1. PVC 请求的 `storage` ≤ PV 的 `capacity`
2. `accessModes` 兼容
3. `storageClassName` **完全一致**（含空字符串的情况）

---

### 6.2 访问模式（Access Modes）

| 模式 | 缩写 | 说明 |
|------|------|------|
| ReadWriteOnce | RWO | 单节点读写（最常用） |
| ReadOnlyMany | ROX | 多节点只读 |
| ReadWriteMany | RWX | 多节点读写（hostPath 不支持） |
| ReadWriteOncePod | RWOP | 单 Pod 读写（v1.29+） |

---

### 6.3 回收策略（Reclaim Policy）

| 策略 | 行为 |
|------|------|
| **Retain** | 删除 PVC 后 PV 保留，数据不丢，需手动清理 |
| **Delete** | 删除 PVC 后自动删除 PV 及底层存储（云 StorageClass 默认） |
| ~~Recycle~~ | 已废弃，不用记 |

---

### 6.4 Volume Mode

| 模式 | 说明 |
|------|------|
| **Filesystem** | 挂载为目录（默认） |
| **Block** | 原始块设备 |

---

### 6.5 静态 vs 动态供应

**静态供应（手动创建 PV）：**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/my-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: manual
```

**动态供应（StorageClass + PVC 自动创建 PV）：**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner   # 本地存储示例
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer     # 延迟绑定到 Pod 调度后
```

**volumeBindingMode：**

| 模式 | 说明 |
|------|------|
| **Immediate** | PVC 创建后立即绑定 PV |
| **WaitForFirstConsumer** | 等到 Pod 调度后再绑定（避免跨节点问题） |

---

### 6.6 Pod 挂载 PVC

```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

**ConfigMap / Secret 作为 Volume：**

```yaml
volumes:
- name: config-vol
  configMap:
    name: app-config
- name: secret-vol
  secret:
    secretName: db-creds
```

---

### 6.7 常见陷阱

- PVC 一直 `Pending`：检查 accessModes、storageClassName、容量
- `storageClassName: ""` 只匹配无 StorageClass 的 PV
- hostPath 不适合生产，但考试常用
- 删除顺序：先删 Pod → 再删 PVC → 最后处理 PV

---

## 七、故障排查（30%）

> 占比最高，建议建立**固定排查流程**，避免随机试错浪费时间。

### 7.1 通用排查流程

```
1. kubectl get nodes          → 节点是否 Ready？
2. kubectl get pods -A        → 哪些 Pod 异常？
3. kubectl describe <resource> → 查看 Events（最关键）
4. kubectl logs / --previous  → 查看容器日志
5. kubectl get events --sort-by='.lastTimestamp'
```

---

### 7.2 Pod 状态排查

| 状态 | 常见原因 | 排查方向 |
|------|----------|----------|
| **Pending** | 调度失败 | PVC 未绑定、资源不足、NodeSelector/Affinity 不匹配、Taint 无 Toleration |
| **CrashLoopBackOff** | 容器启动即崩溃 | `kubectl logs --previous`、命令/入口错误、缺少 ConfigMap/Secret |
| **ImagePullBackOff** | 镜像拉取失败 | 镜像名拼写、私有仓库缺少 imagePullSecrets |
| **Error / Failed** | 运行错误 | describe events + logs |
| **OOMKilled** | 内存超限 | 检查 limits，增大 memory limit |
| **CreateContainerConfigError** | 配置错误 | ConfigMap/Secret 不存在或 key 错误 |

```bash
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl logs <pod> -c <container-name>
kubectl exec <pod> -- env
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

### 7.3 节点故障排查

```bash
kubectl get nodes
kubectl describe node <node-name>

# SSH 到节点
sudo systemctl status kubelet
sudo systemctl restart kubelet
sudo journalctl -u kubelet --no-pager | tail -50
sudo journalctl -u containerd --no-pager | tail -50

# 检查证书
sudo kubeadm certs check-expiration

# NotReady 常见原因：kubelet 停止、容器运行时故障、磁盘压力、网络问题
```

---

### 7.4 控制平面组件故障

**Static Pod 排查：**

```bash
kubectl get pods -n kube-system
ls /etc/kubernetes/manifests/

# 查看各组件日志
kubectl logs -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system etcd-<node>
kubectl logs -n kube-system kube-scheduler-<node>
kubectl logs -n kube-system kube-controller-manager-<node>
```

**etcd 健康检查：**

```bash
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**常见修复：**

| 问题 | 修复 |
|------|------|
| kubelet 未运行 | `systemctl start kubelet` |
| manifest YAML 错误 | 编辑 `/etc/kubernetes/manifests/` 下文件 |
| 证书过期 | `kubeadm certs renew all` + 重启组件 |
| etcd data-dir 错误 | 修正 etcd.yaml 中 `--data-dir` |

---

### 7.5 网络故障排查

```bash
# 1. Service 是否存在，selector 是否正确
kubectl get svc <service>
kubectl describe svc <service>

# 2. Endpoints 是否有后端 Pod
kubectl get endpoints <service>

# 3. Pod 端口是否监听
kubectl exec <pod> -- wget -qO- localhost:<port>

# 4. DNS 是否正常
kubectl run test --image=busybox:1.36 --rm -it -- nslookup <service>

# 5. kube-proxy 是否运行
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# 6. NetworkPolicy 是否阻断
kubectl get networkpolicy -n <namespace>

# 7. CoreDNS 是否正常
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

**最常见网络问题：**

1. Service `selector` 与 Pod `labels` 不匹配（拼写错误）
2. Service `targetPort` 与容器 `containerPort` 不一致
3. NetworkPolicy 阻止流量且未放行 DNS
4. CoreDNS Pod 未运行
5. kube-proxy 异常

---

### 7.6 日志位置汇总

| 组件 | 日志查看方式 |
|------|-------------|
| kubelet | `journalctl -u kubelet` |
| containerd | `journalctl -u containerd` |
| kube-apiserver | `kubectl logs -n kube-system kube-apiserver-<node>` |
| etcd | `kubectl logs -n kube-system etcd-<node>` |
| 应用容器 | `kubectl logs <pod>` / `--previous` |

---

### 7.7 资源监控

```bash
kubectl top nodes          # 需要 metrics-server
kubectl top pods
kubectl describe node <node>   # 查看 Allocated resources
kubectl get events --sort-by='.lastTimestamp'
```

---

## 八、必背命令速查

### 8.1 kubectl 别名与自动补全

```bash
alias k=kubectl
complete -F __start_kubectl k
export dryrun='--dry-run=client -o yaml'    # 考试常用：k run nginx --image=nginx $dryrun
```

### 8.2 资源操作

```bash
kubectl get/describe/create/apply/delete/edit <resource>
kubectl get all -n <namespace>
kubectl get <resource> -o yaml/json/wide
kubectl label/annotate <resource> key=value
kubectl patch <resource> ...
```

### 8.3 集群信息

```bash
kubectl cluster-info
kubectl config get-contexts
kubectl config use-context <context>
kubectl api-resources
kubectl explain <resource>.<field>
```

### 8.4 故障排查

```bash
kubectl auth can-i <verb> <resource> --as=<user> -n <ns>
kubectl debug node/<node> -it --image=busybox     # 节点调试
kubectl cp <pod>:/path /local/path
```

---

## 九、YAML 模板要点

### 9.1 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: my-app
spec:
  containers:
  - name: main
    image: nginx:1.27
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
  restartPolicy: Always    # Always | OnFailure | Never
```

### 9.2 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.27
```

### 9.3 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

### 9.4 RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 十、学习路线与备考建议

### 10.1 推荐学习阶段（4–6 周）

| 阶段 | 时间 | 内容 |
|------|------|------|
| **第一阶段** | 第 1–2 周 | K8s 架构、Pod/Deployment/Service 基础、kubectl 熟练 |
| **第二阶段** | 第 2–3 周 | RBAC、Storage、NetworkPolicy、Ingress |
| **第三阶段** | 第 3–4 周 | kubeadm 安装/升级、etcd 备份恢复、Static Pod |
| **第四阶段** | 第 4–5 周 | 故障排查专项、模拟题、限时练习 |
| **第五阶段** | 考前 1 周 | 查漏补缺、熟记 etcd/upgrade 流程、刷模拟考 |

### 10.2 推荐练习环境

- **killercoda.com** — 免费浏览器 K8s 实验环境
- **Play with Kubernetes** — 在线多节点集群
- **Minikube / Kind / kubeadm** — 本地搭建
- **Killer Shell CKA Mock** — 模拟考试（强烈推荐）

### 10.3 官方文档速查路径

| 主题 | 文档路径 |
|------|----------|
| kubectl 速查 | `/docs/reference/kubectl/quick-reference/` |
| kubeadm 升级 | `/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/` |
| etcd 备份 | `/docs/tasks/administer-cluster/configure-upgrade-etcd/` |
| RBAC | `/docs/reference/access-authn-authz/rbac/` |
| NetworkPolicy | `/docs/concepts/services-networking/network-policies/` |
| Ingress | `/docs/concepts/services-networking/ingress/` |
| PV/PVC | `/docs/concepts/storage/persistent-volumes/` |

### 10.4 考试时间分配策略

| 题型 | 建议用时 |
|------|----------|
| 创建 Pod/Deployment | 2–3 分钟 |
| RBAC | 3–5 分钟 |
| PV/PVC/StorageClass | 4–6 分钟 |
| NetworkPolicy | 5–8 分钟 |
| Ingress/Gateway | 4–6 分钟 |
| Static Pod | 3–4 分钟 |
| 节点 drain/cordon | 2–3 分钟 |
| 故障排查 | 5–8 分钟 |
| **etcd 备份恢复** | **8–10 分钟** |
| **kubeadm 升级** | **8–10 分钟** |

**两趟策略：**

1. **第一趟（约 80 分钟）：** 按顺序做题，超过 8 分钟未解出的题先标记跳过
2. **第二趟（约 40 分钟）：** 回头做标记题，确保简单题不丢分

### 10.5 常见失分点

- etcd 备份时证书路径写错（必须从 etcd.yaml 中复制）
- NetworkPolicy 忘记放行 DNS（UDP 53）
- Deployment 更新时容器名写错
- PV/PVC 的 storageClassName 不匹配
- 静态 Pod 用 kubectl delete 而非删除 manifest
- kubeadm 升级顺序错误（应先 drain 工作节点）
- Service selector 与 Pod label 拼写不一致

---

## 十一、考前检查清单

### 技能自检

- [ ] 能在 5 分钟内完成 RBAC 四件套创建与验证
- [ ] 能在 10 分钟内完成 etcd 备份与恢复（含修改 manifest）
- [ ] 能在 10 分钟内完成 kubeadm 集群升级（控制面 + 工作节点）
- [ ] 能独立编写 NetworkPolicy（含 DNS egress）
- [ ] 能排查 Pending / CrashLoopBackOff / ImagePullBackOff 的 Pod
- [ ] 能排查 Service 无 Endpoints 的网络问题
- [ ] 能创建 PV + PVC + Pod 完整存储链路
- [ ] 能创建 Static Pod 并理解其生命周期
- [ ] 能在 kubernetes.io/docs 中 30 秒内找到所需 YAML 模板
- [ ] 熟练使用 `--dry-run=client -o yaml` 生成 YAML

### 考试当天

- [ ] 提前 15 分钟进入 PSI 监考环境
- [ ] 确认浏览器只能访问允许的 3 个网站
- [ ] 设置 kubectl 别名：`alias k=kubectl`
- [ ] 在记事本预先写好 etcd 常用命令模板
- [ ] 每题完成后验证结果（get/describe/auth can-i）
- [ ] 不会的题目先标记，不要死磕

---

## 参考资源

- [CNCF CKA 认证页面](https://www.cncf.io/training/certification/cka/)
- [Linux Foundation CKA 详情](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [Kubernetes 官方文档](https://kubernetes.io/docs/home/)
- [kubectl 速查手册](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [CNCF Curriculum 开源考纲](https://github.com/cncf/curriculum)

---

*最后更新：2026 年 6 月 | 适用 Kubernetes v1.34 ~ v1.35*
