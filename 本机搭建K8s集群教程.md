# 本机搭建 Kubernetes 集群教程（Windows + WSL2）

> 面向 CKA 备考的动手环境搭建指南。  
> 适用环境：Windows 10/11 + WSL2（Ubuntu）+ Docker Desktop。  
> 配套文档：  
> - [CKA学习知识点总结.md](./CKA学习知识点总结.md)  
> - [Rancher与Lens管理K8s集群教程.md](./Rancher与Lens管理K8s集群教程.md)（图形化/多集群管理）

---

## 目录

- [一、方案选择](#一方案选择)
- [二、硬件与软件要求](#二硬件与软件要求)
- [三、基础环境准备](#三基础环境准备)
- [四、方案 A：Minikube 快速入门（推荐新手第一步）](#四方案-aminikube-快速入门推荐新手第一步)
- [五、方案 B：Kind 多节点集群（轻量练习）](#五方案-bkind-多节点集群轻量练习)
- [六、方案 C：Multipass + kubeadm（最接近 CKA 考试）](#六方案-cmultipass--kubeadm最接近-cka-考试)
- [七、集群搭好后的第一个练习](#七集群搭好后的第一个练习)
- [八、日常管理命令](#八日常管理命令)
- [九、常见问题排查](#九常见问题排查)
- [十、按 CKA 考纲的练习路线](#十按-cka-考纲的练习路线)

---

## 一、方案选择

| 方案 | 难度 | 资源占用 | 接近 CKA 程度 | 适合阶段 |
|------|------|----------|---------------|----------|
| **A. Minikube** | ⭐ 简单 | 低（~2 CPU / 4GB） | ⭐⭐ | 第 1 周：熟悉 kubectl、Pod、Deployment |
| **B. Kind** | ⭐⭐ 中等 | 低（~4 CPU / 6GB） | ⭐⭐ | 多节点调度、NetworkPolicy 练习 |
| **C. Multipass + kubeadm** | ⭐⭐⭐ 较难 | 高（~8 CPU / 12GB） | ⭐⭐⭐⭐⭐ | 第 2 周起：etcd、升级、RBAC、故障排查 |

**建议路径：**

```
第 1 周  → 方案 A（Minikube）跑通基础操作
第 2 周  → 方案 B（Kind）练习多节点与网络
第 3 周起 → 方案 C（kubeadm）专项攻克 CKA 高频题
```

> CKA 考试环境是 **Ubuntu + kubeadm 集群**，方案 C 最值得投入时间，但不建议第一天就上 kubeadm——先用 Minikube 建立手感。

**不需要 VMware：** 三种方案均可在 WSL2 + Docker 或 Multipass 上完成，无需手动克隆三台 Ubuntu 虚拟机。

---

## 二、硬件与软件要求

### 最低配置

| 项目 | Minikube | Kind 多节点 | kubeadm 三节点 |
|------|----------|-------------|----------------|
| CPU | 2 核 | 4 核 | 8 核 |
| 内存 | 4 GB | 6 GB | 12 GB |
| 磁盘 | 20 GB 可用 | 30 GB 可用 | 50 GB 可用 |

### 必需软件

| 软件 | 用途 | 你本机状态 |
|------|------|-----------|
| WSL2 + Ubuntu | 提供 Linux 命令行，K8s 工具链运行在 Linux 上 | ✅ 已安装 |
| Docker Desktop | 容器运行时，Minikube/Kind 的底层依赖 | ✅ 已安装 |
| kubectl | 与 K8s API Server 通信的客户端 CLI | ✅ 已安装（建议升级到 v1.29+） |

### 核心概念速览（读教程前先看）

| 概念 | 含义 |
|------|------|
| **kubectl** | 你操作集群的命令行工具，发请求给 API Server |
| **节点（Node）** | 跑 Pod 的机器，分控制平面节点和工作节点 |
| **Pod** | K8s 最小调度单元，里面跑一个或多个容器 |
| **容器运行时** | 真正拉镜像、启容器的软件（containerd / Docker） |
| **CNI** | 容器网络插件，负责 Pod 之间互通（如 Calico） |
| **kubeadm** | 官方集群安装工具，CKA 考试用的就是它 |

---

## 三、基础环境准备

以下步骤在 **Windows PowerShell（管理员）** 和 **WSL2 Ubuntu 终端** 中分别执行。

---

### 3.1 确认 WSL2 与 Docker 联动

#### 这一步在做什么？

WSL2 让你在 Windows 里运行 Ubuntu；Docker Desktop 负责跑容器。Minikube/Kind 需要 Docker 作为底层驱动，因此必须让 **WSL 里的 Ubuntu 能调用 Docker**。

#### 命令与参数说明

**PowerShell：**

```powershell
wsl --status
```

| 参数/命令 | 说明 |
|-----------|------|
| `wsl --status` | 查看 WSL 全局状态，确认默认版本是否为 2 |
| `wsl -l -v` | `-l` 列出已安装发行版，`-v` 显示 WSL 版本（必须是 2） |

**预期结果：** 默认发行版为 `Ubuntu`，VERSION 为 `2`。WSL1 不支持 Docker 集成，必须用 WSL2。

**Docker Desktop 设置：**

打开 **Docker Desktop → Settings → Resources → WSL Integration**，勾选 `Ubuntu`，点 **Apply & Restart**。

| 设置项 | 含义 |
|--------|------|
| WSL Integration | 允许指定 WSL 发行版直接使用 Docker Desktop 的 Docker 引擎 |
| 勾选 Ubuntu | 让 WSL 终端里的 `docker` 命令生效，而不是报错「找不到 docker」 |

**WSL Ubuntu 终端验证：**

```bash
docker run hello-world
docker ps
```

| 命令 | 说明 |
|------|------|
| `docker run hello-world` | 拉取并运行测试镜像，验证 Docker 引擎可用 |
| `docker ps` | 列出当前运行中的容器（hello-world 跑完即退出，列表可能为空，不报错即可） |

---

### 3.2 在 WSL 中安装/升级 kubectl

#### 这一步在做什么？

Windows 上虽有 kubectl，但后续 Minikube/Kind 主要在 **WSL 终端**操作。在 WSL 内单独装一份，避免路径和版本不一致。

#### 命令与参数说明

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

| 部分 | 说明 |
|------|------|
| `curl -LO` | `-L` 跟随重定向，`-O` 按远程文件名保存到当前目录 |
| `$(curl ... stable.txt)` | 子命令替换，自动获取当前 **stable 稳定版** 版本号 |
| `linux/amd64` | WSL Ubuntu 在 x64 PC 上用 amd64 架构二进制 |

```bash
chmod +x kubectl          # 赋予可执行权限
sudo mv kubectl /usr/local/bin/   # 放入 PATH，任意目录可直接调用
kubectl version --client  # 只查看客户端版本（集群未建时无法看服务端）
```

| 参数 | 说明 |
|------|------|
| `--client` | 仅显示 kubectl 客户端版本，不连接集群 |

> **版本建议：** 尽量与目标 K8s 集群版本相差不超过一个小版本（如集群 v1.31，kubectl v1.30~v1.31 均可）。

---

### 3.3 配置 kubectl 别名（强烈建议）

#### 这一步在做什么？

CKA 考试和日常练习中 kubectl 使用频率极高。设置别名和自动补全，可显著提速。

#### 配置说明

在 WSL 的 `~/.bashrc` 末尾追加：

```bash
alias k=kubectl
complete -F __start_kubectl k 2>/dev/null
export do="--dry-run=client -o yaml"
```

| 配置 | 含义 |
|------|------|
| `alias k=kubectl` | 把 `k` 映射为 `kubectl`，如 `k get pods` |
| `complete -F __start_kubectl k` | 为 `k` 启用 Tab 自动补全（资源名、命名空间等） |
| `export do="--dry-run=client -o yaml"` | 考试常用技巧：生成 YAML 模板而不真正创建资源 |

**`$do` 用法示例（CKA 高频）：**

```bash
k run nginx --image=nginx $do > pod.yaml   # 生成 Pod YAML 到文件，再编辑 apply
```

| `$do` 展开参数 | 说明 |
|---------------|------|
| `--dry-run=client` | 只在本地模拟，不发给 API Server |
| `-o yaml` | 输出 YAML 格式，便于修改后 `kubectl apply -f` |

```bash
source ~/.bashrc   # 使当前终端立即生效
```

---

### 3.4 关闭 WSL 内 swap（kubeadm 方案必需）

#### 这一步在做什么？

kubeadm 官方要求 **关闭 swap**，否则 kubelet 会拒绝启动或集群初始化失败。Minikube/Kind 不强制，但方案 C 必须做。

#### 命令说明

```bash
sudo swapoff -a
```

| 说明 |
|------|
| 立即关闭所有 swap 分区，重启 WSL 后可能恢复 |

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

| 部分 | 说明 |
|------|------|
| `sed -i` | 直接修改文件 |
| `/ swap / s/^\(.*\)$/#\1/g` | 在 `/etc/fstab` 中找到 swap 行并在行首加 `#` 注释，防止重启后再启用 |

> Minikube/Kind 用户可跳过 3.4，方案 C 三台 VM 内各执行一次。

---

## 四、方案 A：Minikube 快速入门（推荐新手第一步）

> 10–15 分钟可跑起来。Minikube 会在 Docker 里创建一个「迷你 Linux 节点」，并在其上自动装好 K8s。

---

### 4.1 安装 Minikube

**在 WSL Ubuntu 中：**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

| 命令 | 说明 |
|------|------|
| `curl -LO .../latest/...` | 下载 Minikube 最新 Linux 版二进制 |
| `install ... /usr/local/bin/minikube` | 安装到系统 PATH，比 `mv` 会自动设置权限 |
| `minikube version` | 确认安装成功，显示 client 和可能的 cluster 版本 |

---

### 4.2 启动单节点集群

```bash
minikube start \
  --driver=docker \
  --cpus=2 \
  --memory=4096 \
  --kubernetes-version=v1.31.0
```

#### 参数详解

| 参数 | 含义 | 为何重要 |
|------|------|----------|
| `--driver=docker` | 用 Docker 容器模拟 K8s 节点 | 你已装 Docker Desktop，最省事；其他选项还有 `hyperv`、`kvm2` |
| `--cpus=2` | 给 Minikube 节点分配 2 个 CPU | 太少会导致 Pod 调度慢或 Pending |
| `--memory=4096` | 分配 4096 MB（4GB）内存 | 低于 2GB 常启动失败；可按本机内存调整 |
| `--kubernetes-version=v1.31.0` | 指定 K8s 版本 | 不指定则用 Minikube 默认版本；备考建议指定与考纲接近的版本 |

**启动后 Minikube 会自动：**

1. 在 Docker 里创建节点容器
2. 用 kubeadm 初始化集群
3. 安装默认 CNI 网络
4. 把 kubeconfig 写入 `~/.kube/config`，kubectl 可直接使用

**验证：**

```bash
minikube status
kubectl get nodes
```

| 命令 | 说明 |
|------|------|
| `minikube status` | 查看 host、kubelet、apiserver 状态是否 Running |
| `kubectl get nodes` | 应看到名为 `minikube` 的节点，STATUS 为 `Ready` |

**节点 STATUS 含义：**

| 状态 | 含义 |
|------|------|
| `Ready` | 节点健康，可调度 Pod |
| `NotReady` | kubelet 或网络异常，Pod 无法调度 |
| `SchedulingDisabled` | 节点被 cordon，不接受新 Pod |

---

### 4.3 启用常用插件

```bash
minikube addons enable metrics-server
minikube addons enable ingress
minikube addons enable dashboard
```

| 插件 | 作用 | CKA 关联 |
|------|------|----------|
| `metrics-server` | 采集 CPU/内存指标 | 支持 `kubectl top pods/nodes`、HPA |
| `ingress` | 部署 Ingress Controller | 练习 Ingress 资源 |
| `dashboard` | K8s Web 管理界面 | 可选，可视化查看资源 |

| 子命令 | 说明 |
|--------|------|
| `addons list` | 查看所有可用插件及状态 |
| `addons enable <name>` | 启用插件，会在集群内部署对应组件 |

---

### 4.4 验证集群

```bash
kubectl run nginx --image=nginx:1.27 --port=80
```

| 参数 | 说明 |
|------|------|
| `run nginx` | 创建名为 nginx 的 Pod（K8s 1.27+ 推荐用 `create` 或 YAML，但 run 仍可用于快速测试） |
| `--image=nginx:1.27` | 容器镜像及版本 |
| `--port=80` | 声明容器监听端口（记录用，不自动创建 Service） |

```bash
kubectl get pods -w
```

| 参数 | 说明 |
|------|------|
| `-w` / `--watch` | 持续监视，状态变化时实时刷新；看到 `Running` 后 `Ctrl+C` 退出 |

```bash
kubectl expose pod nginx --port=80 --type=NodePort
```

| 参数 | 说明 |
|------|------|
| `expose pod nginx` | 为 Pod 创建 Service，通过标签选择器关联 |
| `--port=80` | Service 对外暴露的端口 |
| `--type=NodePort` | 在每个节点上开放 30000–32767 范围内的端口，便于从集群外访问 |

```bash
kubectl get svc nginx
minikube service nginx --url
```

| 命令 | 说明 |
|------|------|
| `get svc nginx` | 查看 Service，注意 `PORT(S)` 列如 `80:31234/TCP`（31234 即 NodePort） |
| `minikube service nginx --url` | Minikube 专用：返回可从本机浏览器访问的 URL |

浏览器打开 URL，看到 Nginx 欢迎页 → 集群网络正常。

---

### 4.5 Minikube 常用操作

```bash
minikube stop          # 停止节点容器，保留集群数据和配置
minikube start         # 再次启动（沿用原配置）
minikube delete        # 彻底删除集群和所有数据
minikube ssh           # SSH 进入 minikube 节点内部（排查问题时用）
minikube ip            # 查看节点 IP（NodePort 访问时可能用到）
```

| 命令 | 使用场景 |
|------|----------|
| `stop` / `start` | 不用集群时 stop 省资源，下次 start 秒级恢复 |
| `delete` | 集群搞乱了，从零重建 |
| `ssh` | 查看节点上 kubelet 日志、网络配置等 |
| `ip` | 配合 NodePort 手动拼访问地址 `http://<ip>:<nodePort>` |

---

### 4.6 Minikube 多节点（可选）

```bash
minikube delete
minikube start --nodes=3 --driver=docker --cpus=4 --memory=6144
kubectl get nodes
```

| 参数 | 说明 |
|------|------|
| `--nodes=3` | 创建 1 个控制平面 + 2 个工作节点（共 3 节点） |
| `--cpus=4 --memory=6144` | 多节点需更多资源，按本机情况调整 |

可用于初步练习 **nodeSelector、污点容忍** 等多节点调度，但不如 Kind/kubeadm 真实。

---

## 五、方案 B：Kind 多节点集群（轻量练习）

> Kind = Kubernetes in Docker。每个「节点」是一个 Docker 容器，容器内跑 kubelet + 控制面/工作负载组件。

---

### 5.1 安装 Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

| 说明 |
|------|
| 版本号 `v0.24.0` 可改为 [Kind 发布页](https://github.com/kubernetes-sigs/kind/releases) 最新版 |
| Kind 版本与 K8s 版本有对应关系，一般最新 Kind 支持较新 K8s |

---

### 5.2 创建多节点集群配置文件

```bash
mkdir -p ~/k8s-lab && cd ~/k8s-lab
```

创建 `kind-multi-node.yaml`：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cka-lab
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=control-plane"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=worker,disk=ssd"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "role=worker,disk=hdd"
```

#### 配置文件字段说明

| 字段 | 含义 |
|------|------|
| `kind: Cluster` | 声明这是 Kind 集群配置 |
| `name: cka-lab` | 集群名称，也会成为 kubectl context 名的一部分（`kind-cka-lab`） |
| `role: control-plane` | 该节点跑 API Server、Scheduler 等控制面组件 |
| `role: worker` | 仅跑工作负载 Pod |
| `kubeadmConfigPatches` | 在 kubeadm 初始化/加入时注入额外配置 |
| `node-labels` | 给节点打标签，后续可用 `nodeSelector` / `nodeAffinity` 练习调度 |

> 这里故意给两个 worker 打了不同 `disk` 标签（`ssd` / `hdd`），方便练习「把 Pod 调度到特定节点」。

---

### 5.3 创建并验证集群

```bash
kind create cluster --config kind-multi-node.yaml
```

| 参数 | 说明 |
|------|------|
| `--config` | 指定集群定义文件；不指定则创建默认单节点集群 |

**Kind 创建时自动完成：**

1. 拉取 `kindest/node` 镜像（内含 K8s 组件）
2. 启动 Docker 容器作为节点
3. kubeadm init / join 组建集群
4. 写入 kubeconfig，context 名为 `kind-cka-lab`

```bash
kubectl cluster-info --context kind-cka-lab
kubectl get nodes -o wide
```

| 参数 | 说明 |
|------|------|
| `--context kind-cka-lab` | 指定操作哪个集群（多集群并存时避免误操作） |
| `-o wide` | 显示 NODE IP、OS 镜像等额外列 |

---

### 5.4 安装 Calico（NetworkPolicy 练习必需）

Kind 默认网络（kindnet）**不支持 NetworkPolicy**。要练网络策略必须换 Calico。

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl get pods -n kube-system -w
```

| 说明 |
|------|
| `apply -f <url>` | 直接从远程 YAML 部署资源 |
| Calico 会部署 DaemonSet 到每个节点，负责 Pod 网络和 NetworkPolicy  enforcement |
| `-n kube-system` | 系统组件命名空间 |
| 等待 calico-node、calico-kube-controllers 等 Pod 全部 `Running` |

---

### 5.5 Kind 常用操作

```bash
kind get clusters                              # 列出本机所有 Kind 集群
kubectl config get-contexts                    # 查看 kubectl 可切换的集群
kubectl config use-context kind-cka-lab        # 切换到 Kind 集群
kind delete cluster --name cka-lab             # 删除集群及对应 Docker 容器
```

| 命令 | 说明 |
|------|------|
| `kind get clusters` | 集群名列表 |
| `kubectl config use-context` | 切换当前默认集群 |
| `kind delete cluster --name` | `--name` 须与配置文件中 `name` 一致 |

---

## 六、方案 C：Multipass + kubeadm（最接近 CKA 考试）

> 考试环境 = **Ubuntu + kubeadm + containerd + Calico**。本方案用 Multipass 在 Windows 上快速创建真实 Ubuntu VM，完整手动走一遍官方安装流程。

---

### 6.1 架构规划

```
┌─────────────────────────────────────────────────────────┐
│  Windows 主机（vEthernet k8snet: 10.13.31.1/24）         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│  │ master   │ │ worker1  │ │ worker2  │                │
│  │ .10 静态 │ │ .11 静态 │ │ .12 静态 │  ← k8snet 互通  │
│  │ 2CPU/4GB │ │ 2CPU/3GB │ │ 2CPU/3GB │                │
│  └──────────┘ └──────────┘ └──────────┘                │
│         Multipass 轻量虚拟机（Ubuntu 24.04）              │
│         默认网卡：Default Switch（DHCP，上网用）           │
└─────────────────────────────────────────────────────────┘
```

| 角色 | 组件 |
|------|------|
| master | API Server、etcd、Scheduler、Controller Manager |
| worker | kubelet、kube-proxy、业务 Pod |

---

### 6.2 安装 Multipass

**PowerShell（管理员）：**

```powershell
winget install Canonical.Multipass
multipass version
```

| 说明 |
|------|
| Multipass 是 Canonical 出品的轻量 VM 管理工具，一条命令创建 Ubuntu 实例 |
| 比 VMware 手动装系统省事，底层在 Windows 上通常用 Hyper-V |

---

### 6.3 创建三台虚拟机（固定 IP，防重启 IP 漂移）

> **为什么需要固定 IP？**  
> Windows Hyper-V 的 Default Switch 在主机重启后可能给 VM 分配不同 IP（例如 `172.26.94.43` → 另一个地址），导致 kubeadm 证书、hosts、kubeconfig 全部失效。  
> **做法：** 保留 Default Switch 负责上网，另建 Internal 交换机 `k8snet`，用第二块网卡 + netplan 配置静态 IP，专供 K8s 节点互通。

| 网卡 | 作用 | IP 特点 |
|------|------|---------|
| 默认网卡（Default Switch） | VM 访问外网（apt、拉镜像） | DHCP，重启可能变化 |
| 第二网卡（k8snet） | 节点间通信、kubeadm、kubectl | **静态 IP，重启不变** |

> 若已有未配静态 IP 的实例（`multipass list` 只有一条 `172.x.x.x`），请先执行 [6.12 销毁与重建](#612-kubeadm-集群销毁与重建)，再按本节从头创建。

---

#### 6.3.1 创建 Hyper-V Internal 交换机（一次性）

1. 打开 **Hyper-V 管理器** → **虚拟交换机管理器**
2. 新建 → 类型选 **内部** → 名称填 `k8snet`（小写、单词，与后续 `--network name=` 一致）
3. PowerShell 确认 Multipass 能识别：

```powershell
multipass networks
```

| 输出 | 说明 |
|------|------|
| `k8snet` | 刚创建的 Internal 交换机，后续 `--network name=k8snet` 使用 |

4. 给 Windows 主机侧 vEthernet 配 IP（便于 Windows/WSL 访问 API Server）：

```powershell
# 查看 Hyper-V 虚拟网卡名（通常是 "vEthernet (k8snet)"）
Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "*Hyper-V*" }

# 设静态 IP（InterfaceAlias 按上一步实际名称替换）
New-NetIPAddress -InterfaceAlias "vEthernet (k8snet)" -IPAddress 10.13.31.1 -PrefixLength 24
```

| 说明 |
|------|
| `10.13.31.1/24` 是主机在 k8snet 上的地址，VM 静态 IP 与之同网段 |
| 若提示地址已存在，说明之前配过，可跳过 |

---

#### 6.3.2 IP 与 MAC 规划

| 节点 | 静态 IP | launch 时指定 MAC |
|------|---------|-------------------|
| k8s-master | `10.13.31.10/24` | `52:54:00:00:00:10` |
| k8s-worker1 | `10.13.31.11/24` | `52:54:00:00:00:11` |
| k8s-worker2 | `10.13.31.12/24` | `52:54:00:00:00:12` |

下文占位符对应关系：

| 占位符 | 固定值 |
|--------|--------|
| `<MASTER_IP>` | `10.13.31.10` |
| `<WORKER1_IP>` | `10.13.31.11` |
| `<WORKER2_IP>` | `10.13.31.12` |

> **重要：** kubeadm、hosts、join、WSL kubeconfig **一律使用上表静态 IP**，不要用 `multipass list` 里 Default Switch 的 DHCP 地址（如 `172.26.x.x`）。

---

#### 6.3.3 创建实例（带第二网卡）

**PowerShell：**

```powershell
multipass launch 24.04 --name k8s-master  --cpus 2 --memory 4G --disk 20G --network name=k8snet,mode=manual,mac=52:54:00:00:00:10
multipass launch 24.04 --name k8s-worker1 --cpus 2 --memory 3G --disk 20G --network name=k8snet,mode=manual,mac=52:54:00:00:00:11
multipass launch 24.04 --name k8s-worker2 --cpus 2 --memory 3G --disk 20G --network name=k8snet,mode=manual,mac=52:54:00:00:00:12
multipass list
```

| 参数 | 含义 |
|------|------|
| `launch 24.04` | 创建 Ubuntu 24.04 LTS 实例 |
| `--name k8s-master` | 实例名称，后续 `shell`、`info` 都用这个名字 |
| `--cpus 2` / `--memory 4G` / `--disk 20G` | CPU、内存、磁盘 |
| `--network name=k8snet,mode=manual,mac=...` | 附加第二网卡，连到 k8snet，MAC 与 6.3.2 表一致 |

---

#### 6.3.4 配置 netplan 静态 IP（每台各执行一次）

以 **k8s-master** 为例（worker 改 MAC 和 IP 即可）：

```powershell
multipass exec k8s-master -- sudo bash -c 'cat << EOF > /etc/netplan/10-k8s-static.yaml
network:
  version: 2
  ethernets:
    k8snet0:
      dhcp4: no
      match:
        macaddress: "52:54:00:00:00:10"
      addresses: [10.13.31.10/24]
EOF'
multipass exec k8s-master -- sudo netplan apply
```

**k8s-worker1：**

```powershell
multipass exec k8s-worker1 -- sudo bash -c 'cat << EOF > /etc/netplan/10-k8s-static.yaml
network:
  version: 2
  ethernets:
    k8snet0:
      dhcp4: no
      match:
        macaddress: "52:54:00:00:00:11"
      addresses: [10.13.31.11/24]
EOF'
multipass exec k8s-worker1 -- sudo netplan apply
```

**k8s-worker2：**

```powershell
multipass exec k8s-worker2 -- sudo bash -c 'cat << EOF > /etc/netplan/10-k8s-static.yaml
network:
  version: 2
  ethernets:
    k8snet0:
      dhcp4: no
      match:
        macaddress: "52:54:00:00:00:12"
      addresses: [10.13.31.12/24]
EOF'
multipass exec k8s-worker2 -- sudo netplan apply
```

| 说明 |
|------|
| `match.macaddress` 必须与 launch 时 `--network ... mac=` 一致 |
| 只给 k8snet 网卡设静态 IP，**不要**关闭默认网卡的 DHCP（否则 VM 无法上网） |

---

#### 6.3.5 验证网络

```powershell
multipass info k8s-master
ping 10.13.31.10
ping 10.13.31.11
ping 10.13.31.12
multipass exec k8s-master -- ip -br a
```

| 检查项 | 预期 |
|--------|------|
| `multipass info` | 显示 **两个 IPv4**：一条 `172.x.x.x`（Default Switch，会变）+ 一条 `10.13.31.10`（固定） |
| `ping 10.13.31.x` | 三台静态 IP 互通 |
| `ip -br a` | 除默认网卡外，k8snet 网卡有 `10.13.31.x/24` |

---

#### 6.3.6 重启验证（确认 IP 不再漂移）

```powershell
multipass stop k8s-master k8s-worker1 k8s-worker2
multipass start k8s-master k8s-worker1 k8s-worker2
ping 10.13.31.10
```

| 说明 |
|------|
| 重启后 `multipass list` 里的 `172.x.x.x` 可能变化，**不影响集群** |
| `10.13.31.10/11/12` 应保持不变；集群搭好后可用 `kubectl get nodes` 进一步验证 |

---

### 6.4 所有节点通用初始化脚本

进入 master：

```powershell
multipass shell k8s-master
```

以下在 **三台机器各执行一次**（hostname 按节点改）。

---

#### 6.4.1 设置主机名

```bash
sudo hostnamectl set-hostname k8s-master
```

| 说明 |
|------|
| 设置系统主机名，worker 改为 `k8s-worker1` / `k8s-worker2` |
| 便于日志和 `kubectl get nodes` 识别节点 |

---

#### 6.4.2 关闭 swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

| 原因 |
|------|
| kubelet 默认要求 swap 关闭，否则节点 NotReady 或 kubeadm init 报错 |

---

#### 6.4.3 加载内核模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

| 模块 | 作用 |
|------|------|
| `overlay` | 容器镜像分层存储（OverlayFS） |
| `br_netfilter` | 让 iptables 能过滤网桥流量，Service 网络依赖它 |

| 命令 | 说明 |
|------|------|
| `/etc/modules-load.d/k8s.conf` | 开机自动加载模块 |
| `modprobe` | 当前会话立即加载 |

---

#### 6.4.4 网络 sysctl 参数

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

| 参数 | 作用 |
|------|------|
| `bridge-nf-call-iptables = 1` | 网桥流量走 iptables 规则（kube-proxy Service 转发需要） |
| `bridge-nf-call-ip6tables = 1` | IPv6 同理 |
| `ip_forward = 1` | 允许内核转发包，Pod 跨节点通信需要 |

---

#### 6.4.5 安装 containerd

K8s 从 1.24 起**不再内置 Docker**，官方推荐 **containerd** 作为容器运行时（考试环境用它）。

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

安装 Docker 官方 apt 源（containerd 包在此源中）：

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io
```

| 说明 |
|------|
| 这里装的是 **containerd**，不是 Docker Engine；K8s 通过 CRI 接口调用 containerd |

**配置 containerd：**

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

| 配置 | 说明 |
|------|------|
| `config default` | 生成默认配置到 `/etc/containerd/config.toml` |
| `SystemdCgroup = true` | 使用 systemd 管理 cgroup，与 kubelet 一致（**必改**，否则 kubelet 启动失败） |
| `enable containerd` | 开机自启 |

---

#### 6.4.6 安装 kubeadm、kubelet、kubectl

```bash
K8S_VERSION=1.31
```

| 变量 | 说明 |
|------|------|
| `K8S_VERSION=1.31` | 主版本号，三台机器必须一致 |

添加 Kubernetes 官方 apt 源并安装：

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

| 组件 | 作用 |
|------|------|
| **kubeadm** | 集群初始化、加入、升级 |
| **kubelet** | 每个节点上的 K8s 代理，管理 Pod 生命周期 |
| **kubectl** | 命令行客户端（master 上管理集群用） |

| 命令 | 说明 |
|------|------|
| `apt-mark hold` | 锁定版本，防止 `apt upgrade` 意外升级导致集群版本不一致（**CKA 考点**） |
| `enable kubelet` | 开机自启；init 之前 kubelet 会反复重启属正常现象 |

---

### 6.5 配置 hosts（所有节点）

使用 [6.3.2](#632-ip-与-mac-规划) 中的**静态 IP**（不要用 `172.x.x.x`）：

```bash
cat <<EOF | sudo tee -a /etc/hosts
10.13.31.10  k8s-master
10.13.31.11  k8s-worker1
10.13.31.12  k8s-worker2
EOF
```

| 原因 |
|------|
| 节点间用主机名通信（证书 SAN、日志可读性）；考试环境也通常配 hosts |
| 静态 IP 保证 Windows 重启后 hosts 仍然有效 |

---

### 6.6 初始化控制平面（仅 master 节点）

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=10.13.31.10
```

#### 参数详解

| 参数 | 含义 |
|------|------|
| `--pod-network-cidr=192.168.0.0/16` | 分配给 Pod 的网段；**必须与 CNI 插件（Calico）配置一致** |
| `--apiserver-advertise-address=10.13.31.10` | API Server 对外公告的 IP；**必须用 k8snet 静态 IP**，worker 和 WSL kubectl 通过此地址连接 master |

**init 成功后会输出：**

1. `kubeadm join ...` 命令 → **务必保存**，worker 加入用
2. 提示配置 kubeconfig

**配置 kubectl（普通用户，非 root）：**

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

| 文件 | 说明 |
|------|------|
| `/etc/kubernetes/admin.conf` | 集群管理员 kubeconfig，含证书和 API Server 地址 |
| `$HOME/.kube/config` | kubectl 默认读取的配置路径 |

**若忘记 join 命令：**

```bash
kubeadm token create --print-join-command
```

| 参数 | 说明 |
|------|------|
| `--print-join-command` | 生成带新 token 的完整 join 命令，可直接复制到 worker 执行 |

**join 命令参数说明（worker 上会用到）：**

| 参数 | 含义 |
|------|------|
| `10.13.31.10:6443` | API Server 地址（k8snet 静态 IP），6443 是默认端口 |
| `--token` | /bootstrap 认证用的一次性令牌，默认 24 小时过期 |
| `--discovery-token-ca-cert-hash sha256:...` | 验证 master 身份，防止中间人攻击 |

---

### 6.7 安装 Calico 网络插件（master 节点）

kubeadm init **不会**自动装网络插件，不装则节点一直 `NotReady`。

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl get pods -n kube-system -w
```

| 说明 |
|------|
| Calico 的默认 Pod CIDR 为 `192.168.0.0/16`，与 init 时 `--pod-network-cidr` 对应 |
| 若 CIDR 不一致，Pod 网络会异常 |

**安装完成后节点变为 Ready 的条件：** calico-node Pod 在每个节点 Running。

---

### 6.8 工作节点加入集群

在 **k8s-worker1**、**k8s-worker2** 上执行（6.6 保存的命令）：

```bash
sudo kubeadm join 10.13.31.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

| 说明 |
|------|
| 必须加 `sudo`，join 会修改 kubelet 配置并重启 kubelet |
| 成功输出 `This node has joined the cluster` |

在 master 验证：

```bash
kubectl get nodes
```

---

### 6.9 验证集群（master 节点）

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

**预期：**

| 检查项 | 预期 |
|--------|------|
| 3 个节点 | STATUS 均为 `Ready` |
| kube-system Pod | calico-node、kube-proxy、coredns 等为 `Running` |

---

### 6.10 允许 master 调度 Pod（单控制平面实验环境）

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

| 概念 | 说明 |
|------|------|
| **Taint（污点）** | 节点上的「排斥规则」，没有对应 Toleration 的 Pod 不能调度上来 |
| 控制平面默认污点 | `node-role.kubernetes.io/control-plane:NoSchedule`，防止业务 Pod 占用 master |
| 命令末尾 `-` | 表示 **移除** 该污点 |
| 学习环境 | 只有一台 master 时，去掉污点可以让 Pod 也调度到 master，节省资源 |

---

### 6.11 从 Windows/WSL 使用 kubectl 管理 Multipass 集群

**方法一：在 master VM 内操作（最简单）**

```powershell
multipass shell k8s-master
kubectl get nodes
```

**方法二：把 kubeconfig 复制到 WSL**

PowerShell：

```powershell
multipass exec k8s-master -- cat /home/ubuntu/.kube/config > $env:USERPROFILE\.kube\config-multipass
```

| 部分 | 说明 |
|------|------|
| `multipass exec k8s-master --` | 在 VM 内执行命令而不进入交互 shell |
| `cat .../config` | 输出 kubeconfig 内容重定向到 Windows 文件 |

WSL 中：

```bash
mkdir -p ~/.kube
cp /mnt/c/Users/<你的Windows用户名>/.kube/config-multipass ~/.kube/config
# 若 server 是 127.0.0.1，改成 master 静态 IP
sed -i 's/127.0.0.1/10.13.31.10/g' ~/.kube/config
kubectl get nodes
```

| 步骤 | 原因 |
|------|------|
| 复制到 WSL | 在 WSL 终端用 kubectl 管理 VM 里的集群 |
| `sed` 替换 IP | VM 内 kubeconfig 的 server 可能是 `127.0.0.1`，WSL 访问需改成 master 的 **k8snet 静态 IP** `10.13.31.10` |
| 主机侧验证 | Windows 上 `ping 10.13.31.10` 通，WSL 里 kubectl 才能连上（依赖 [6.3.1](#631-创建-hyper-v-internal-交换机一次性) 给 vEthernet 配的 `10.13.31.1`） |

---

### 6.12 kubeadm 集群销毁与重建

环境乱了或要从头搭静态 IP 集群时，按顺序执行：

**1. 清理 WSL / Windows 上的 kubeconfig（可选）：**

```bash
# WSL
mv ~/.kube/config ~/.kube/config.bak 2>/dev/null
```

```powershell
# PowerShell
Remove-Item "$env:USERPROFILE\.kube\config-multipass" -ErrorAction SilentlyContinue
```

**2. 销毁 Multipass 实例：**

```powershell
multipass stop k8s-master k8s-worker1 k8s-worker2
multipass delete k8s-master k8s-worker1 k8s-worker2
multipass purge
multipass list
```

| 命令 | 说明 |
|------|------|
| `stop` | 先停止再删，避免残留进程 |
| `delete` | 标记删除实例 |
| `purge` | 真正释放磁盘空间，不可恢复 |

**3. 重建：** 从 [6.3](#63-创建三台虚拟机固定-ip防重启-ip-漂移) 起重新创建（Hyper-V 交换机 `k8snet` **不会被 purge 删除**，可复用）。

| 说明 |
|------|
| 若 VM 内已 init 过，也可在 purge 前执行 `multipass exec k8s-master -- sudo kubeadm reset -f`，但 `purge` 会直接删盘，通常不必 |

---

## 七、集群搭好后的第一个练习

无论哪种方案，按顺序完成以下 10 步，**每步都理解在做什么**。

---

### 练习 1：查看集群信息

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

| 命令 | 说明 |
|------|------|
| `cluster-info` | 显示 API Server、CoreDNS 等核心服务地址 |
| `get nodes -o wide` | 节点列表 + 内网 IP、容器运行时版本等 |

---

### 练习 2：创建 Namespace

```bash
kubectl create ns dev
```

| 概念 | 说明 |
|------|------|
| **Namespace** | 逻辑隔离单元，同名资源可在不同 ns 共存 |
| `dev` | 开发环境命名空间，后续资源都放这里 |

等价写法：`kubectl create namespace dev`

---

### 练习 3：创建 Deployment

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3 -n dev
```

| 参数 | 说明 |
|------|------|
| `deployment web` | 创建名为 web 的 Deployment |
| `--image=nginx:1.27` | Pod 中容器镜像 |
| `--replicas=3` | 维持 3 个 Pod 副本 |
| `-n dev` | 指定命名空间 |

**Deployment 做了什么：** 自动创建 ReplicaSet，ReplicaSet 再创建并维持 3 个 Pod。

验证：

```bash
kubectl get deploy,rs,pod -n dev
```

---

### 练习 4：扩缩容

```bash
kubectl scale deployment web --replicas=5 -n dev
```

| 参数 | 说明 |
|------|------|
| `scale deployment web` | 调整 Deployment 副本数 |
| `--replicas=5` | 从 3 扩到 5，K8s 会新起 2 个 Pod |

```bash
kubectl get pods -n dev
```

应看到 5 个 Running 的 Pod。

---

### 练习 5：暴露 Service

```bash
kubectl expose deployment web --port=80 --target-port=80 -n dev
```

| 参数 | 说明 |
|------|------|
| `expose deployment web` | 为 Deployment 下所有 Pod 创建 Service（通过 label selector 关联） |
| `--port=80` | Service 自身的端口（集群内访问用） |
| `--target-port=80` | 转发到 Pod 内容器的端口 |
| 默认 `--type=ClusterIP` | 仅集群内可访问 |

---

### 练习 6：查看 Endpoints

```bash
kubectl get endpoints web -n dev
```

| 概念 | 说明 |
|------|------|
| **Endpoints** | Service 后端实际 Pod IP:Port 列表，由 Endpoint Controller 自动维护 |
| 若 ENDPOINTS 为空 | Service 的 selector 与 Pod labels 不匹配（**CKA 网络排查高频考点**） |

---

### 练习 7：滚动更新

```bash
kubectl set image deployment/web nginx=nginx:1.26 -n dev
```

| 参数 | 说明 |
|------|------|
| `deployment/web` | 格式 `资源类型/名称` |
| `nginx=nginx:1.26` | **容器名=新镜像**（容器名来自 Pod spec，不一定是 deployment 名） |

```bash
kubectl rollout status deployment/web -n dev
```

| 命令 | 说明 |
|------|------|
| `rollout status` | 阻塞等待滚动更新完成 |

```bash
kubectl rollout history deployment/web -n dev
```

查看修订历史，每次镜像变更产生一个新 revision。

---

### 练习 8：回滚

```bash
kubectl rollout undo deployment/web -n dev
```

| 说明 |
|------|
| 回滚到**上一版本**镜像（nginx:1.27） |
| 指定版本：`kubectl rollout undo deployment/web --to-revision=2 -n dev` |

---

### 练习 9：创建 ConfigMap

```bash
kubectl create configmap web-config --from-literal=ENV=dev -n dev
```

| 参数 | 说明 |
|------|------|
| `configmap web-config` | 配置数据对象，存非敏感键值对 |
| `--from-literal=ENV=dev` | 键 ENV，值 dev；可多次使用添加多组键值 |

查看：

```bash
kubectl describe configmap web-config -n dev
```

后续可在 Pod 中通过 `envFrom` 或 volume 挂载使用（参见 CKA 知识点总结）。

---

### 练习 10：查看资源与事件

```bash
kubectl get all -n dev
kubectl get events -n dev --sort-by='.lastTimestamp'
```

| 命令 | 说明 |
|------|------|
| `get all` | 快捷查看 ns 内主要资源（pod、svc、deploy 等） |
| `get events --sort-by='.lastTimestamp'` | 按时间排序的事件，**排障第一参考** |

全部成功 → 可以开始按 [CKA学习知识点总结.md](./CKA学习知识点总结.md) 系统学习。

---

## 八、日常管理命令

### 8.1 切换不同集群上下文

```bash
kubectl config get-contexts
kubectl config use-context minikube
kubectl config current-context
```

| 概念 | 说明 |
|------|------|
| **context** | 集群 + 用户 + 命名空间的组合，决定 kubectl 连哪个集群 |
| `minikube` | Minikube 自动创建的 context 名 |
| `kind-cka-lab` | Kind 集群 context 名格式为 `kind-<集群名>` |

---

### 8.2 集群启停（节省资源）

| 方案 | 停止 | 启动 |
|------|------|------|
| Minikube | `minikube stop` | `minikube start` |
| Kind | 不使用时 `kind delete cluster` | 重新 `kind create cluster` |
| Multipass | `multipass stop k8s-master k8s-worker1 k8s-worker2` | `multipass start k8s-master ...` |

---

### 8.3 资源清理

```bash
kubectl delete ns dev
```

| 说明 |
|------|
| 删除 Namespace 会级联删除其下所有资源 |

```bash
minikube delete && minikube start
kind delete cluster --name cka-lab
```

---

## 九、常见问题排查

### 9.1 Minikube 启动失败

```bash
minikube delete
minikube start --driver=docker --cpus=2 --memory=4096 --force
minikube logs
```

| 参数 | 说明 |
|------|------|
| `--force` | 强制重建集群，忽略部分缓存 |

**常见原因：** Docker 未启动、WSL Integration 未开、内存不足。

---

### 9.2 Kind 节点 NotReady

```bash
kubectl describe node <node-name>
docker ps | grep kind
```

| 排查方向 |
|----------|
| `describe node` 看 Conditions 和 Events |
| Calico 是否安装、Pod CIDR 是否匹配 |

---

### 9.3 kubeadm init 失败

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -e
sudo systemctl status containerd
```

| 命令 | 说明 |
|------|------|
| `journalctl -u kubelet -e` | 查看 kubelet 日志，`-e` 跳到最新 |
| 常见错误 | swap 未关、containerd SystemdCgroup 未改、端口 6443 被占用 |

**重置后重来：**

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/etcd $HOME/.kube/config
```

| 说明 |
|------|
| `kubeadm reset` 清理 init/join 状态，可重新 init |

---

### 9.4 Worker 节点无法加入

```bash
kubeadm token list
kubeadm token create --print-join-command
ping k8s-master
ping 10.13.31.10
```

| 原因 |
|------|
| token 过期（24h）、防火墙阻断 6443、hosts/IP 配置错误 |
| join 地址用了 `172.x.x.x` 而非静态 IP `10.13.31.10` |

---

### 9.4.1 Windows 重启后 kubectl 连不上 / 集群异常

**现象：** `kubectl` 报 `connection refused`；或 `multipass list` 里 IP 变了，节点 NotReady。

**原因：** 未按 [6.3](#63-创建三台虚拟机固定-ip防重启-ip-漂移) 配置 k8snet 静态 IP，kubeadm 绑定了 Default Switch 的 DHCP 地址。

**处理：**

1. 确认静态 IP 是否还在：`ping 10.13.31.10`
2. 若 ping 不通 → 按 6.3 重建 VM 并配 netplan
3. 若 ping 通但 kubectl 失败 → 检查 WSL `~/.kube/config` 中 `server:` 是否为 `https://10.13.31.10:6443`
4. 若集群已 init 在旧 IP 上 → 建议 [6.12 销毁重建](#612-kubeadm-集群销毁与重建)，不要试图手工改证书

---

### 9.5 Pod 一直 Pending

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by='.lastTimestamp'
```

| 常见原因 |
|----------|
| CNI 未就绪、资源不足（CPU/Memory）、PVC 未绑定、nodeSelector 无匹配节点 |

---

### 9.6 CoreDNS 不工作

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl run test --image=busybox:1.36 --rm -it -- nslookup kubernetes.default
```

| 参数 | 说明 |
|------|------|
| `-l k8s-app=kube-dns` | 按 label 筛选 CoreDNS Pod |
| `--rm -it` | 交互运行，结束后自动删除 Pod |
| `nslookup kubernetes.default` | 测试集群 DNS 能否解析默认 Service |

---

## 十、按 CKA 考纲的练习路线

| 周次 | 环境 | 练习内容 |
|------|------|----------|
| 第 1 周 | Minikube | Pod、Deployment、Service、ConfigMap、Secret、kubectl 熟练 |
| 第 2 周 | Kind 多节点 | nodeSelector、Affinity、Taint/Toleration、NetworkPolicy |
| 第 3 周 | kubeadm 集群 | RBAC、Static Pod、PV/PVC/StorageClass |
| 第 4 周 | kubeadm 集群 | etcd 备份恢复、kubeadm 升级、证书续期 |
| 第 5 周 | kubeadm 集群 | 故障排查：改坏 kubelet、apiserver manifest、Service selector |
| 第 6 周 | 在线模拟 | [killercoda.com](https://killercoda.com) 或 Killer Shell Mock Exam |

### kubeadm 集群专项练习清单

- [ ] 手动创建 PV + PVC + Pod 挂载
- [ ] 编写 NetworkPolicy（含 DNS egress UDP 53）
- [ ] 创建 Role + RoleBinding + 用 `kubectl auth can-i` 验证
- [ ] 在 `/etc/kubernetes/manifests/` 创建 Static Pod
- [ ] `etcdctl snapshot save` 备份 + `snapshot restore` 恢复
- [ ] `kubeadm upgrade plan` + 升级一个 worker 节点
- [ ] `kubectl drain` / `cordon` / `uncordon` 节点维护
- [ ] 故意改错 Service selector，再排查修复

---

## 快速决策：我现在该用哪个？

```
只想今天就开始动手？
  └─→ 方案 A：Minikube（第四节）

想练 NetworkPolicy 和多节点调度？
  └─→ 方案 B：Kind（第五节）

认真备考 CKA，想练 etcd / kubeadm 升级？
  └─→ 方案 C：Multipass + kubeadm（第六节）
```

---

*最后更新：2026 年 6 月 | 适用 Windows 11 + WSL2 + Docker Desktop*
