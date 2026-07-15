# Rancher 与 Lens（FreeLens）管理 Kubernetes 集群教程

> 配套文档：  
> - [本机搭建K8s集群教程.md](./本机搭建K8s集群教程.md)  
> - [CKA学习知识点总结.md](./CKA学习知识点总结.md)  
>  
> 适用环境：已有可用的 Kubernetes 集群（推荐 Multipass + kubeadm 方案 C），Windows 10/11 本机安装桌面管理工具。  
> 目标：用 **Rancher** 做多集群运维平台，用 **Lens / FreeLens** 做日常可视化操作，并了解业界其他常用工具。

---

## 目录

- [一、工具定位：kubectl / Lens / Rancher 各自干什么](#一工具定位kubectl--lens--rancher-各自干什么)
- [二、硬件与软件要求](#二硬件与软件要求)
- [三、安装 Rancher（单节点 Docker，适合学习）](#三安装-rancher单节点-docker适合学习)
- [四、把现有 kubeadm 集群导入 Rancher](#四把现有-kubeadm-集群导入-rancher)
- [五、Windows 安装 Lens / FreeLens](#五windows-安装-lens--freelens)
- [六、用 Lens / FreeLens 连接现有集群](#六用-lens--freelens-连接现有集群)
- [七、Rancher 与 Lens 日常能做什么](#七rancher-与-lens-日常能做什么)
- [八、业界其他常用与前沿工具](#八业界其他常用与前沿工具)
- [九、选型建议（备考 / 开发 / 运维）](#九选型建议备考--开发--运维)
- [十、常见问题](#十常见问题)

---

## 一、工具定位：kubectl / Lens / Rancher 各自干什么

三者并不互斥，可同时使用：

```
┌─────────────────────────────────────────────────────────────┐
│  你（工程师）                                                │
│    ├─ kubectl          命令行，CKA 考试主力                   │
│    ├─ Lens / FreeLens  桌面 IDE，看资源、日志、事件           │
│    └─ Rancher UI       多集群平台：建集群、用户、权限、应用商店 │
└─────────────────────────────────────────────────────────────┘
              │ kubeconfig / API Server
              ↓
        Kubernetes 集群（你的 Multipass kubeadm 等）
```

| 工具 | 类型 | 核心作用 | 典型场景 |
|------|------|----------|----------|
| **kubectl** | CLI | 精确操作、脚本化、考试必备 | 排查故障、YAML 演练、CI/CD |
| **Lens / FreeLens** | 桌面客户端（IDE） | 图形化查看工作负载、日志、Shell、端口转发 | 日常开发调试、快速定位问题 |
| **Rancher** | 多集群管理平台（Server） | 统一纳管多集群、用户/RBAC、创建/升级集群、应用目录 | 团队运维、多环境治理 |

### 1.1 Rancher 是什么

**Rancher**（现属 SUSE）是开源的 **Kubernetes 多集群管理平台**。

| 能力 | 说明 |
|------|------|
| **导入现有集群** | 用 Agent 把已有 kubeadm / EKS / ACK 等纳入同一套 UI |
| **创建集群** | RKE2 / K3s 等一键或半自动建集群（生产常用） |
| **统一认证与权限** | 对接 LDAP/OIDC，细粒度项目/命名空间权限 |
| **应用目录** | Catalog / Helm Charts 一键装组件 |
| **监控与告警** | 可集成监控套件（需额外资源） |

**不是**替代 kubectl：底层仍然是标准 K8s API。Rancher 是「在 UI 上管很多集群」的控制台。

> **学习环境注意：** 官方强烈建议 **Docker 单节点安装仅用于开发测试**；生产应使用 Helm 部署到 **高可用 Kubernetes 集群**。

### 1.2 Lens 是什么

**Lens**（Mirantis）是 **Kubernetes 桌面 IDE**，自动读取本机 `kubeconfig`，左侧列出命名空间、Pod、Deploy、Service 等，可一键看日志、进入容器 Shell、做端口转发。

| 产品 | 许可 | 建议 |
|------|------|------|
| **Lens Desktop（官方）** | 商业产品（有试用/订阅） | 公司采购或个人愿意付费 |
| **OpenLens** | 社区旧版分支 | **基本不再维护，不推荐新装** |
| **FreeLens** | 开源、活跃维护 | **个人学习首选免费替代品** |

下文 **Windows 安装以 FreeLens 为主**（免费用），并说明官方 Lens 如何安装；操作方式与 Lens 高度相似。

### 1.3 和你现有环境的关系

若已按《本机搭建K8s集群教程》搭好 Multipass 三节点：

| 方式 | 用途 |
|------|------|
| WSL / master 里 `kubectl` | CKA 实操练习 |
| **FreeLens / Lens** | 本机可视化看节点、Pod、事件 |
| **Rancher** | 体验多集群平台、导入你的 kubeadm 集群 |

Rancher 容器本身较吃内存，**本地学习可先装 Lens；机器资源够（≥16GB）再装 Rancher**。

---

## 二、硬件与软件要求

| 项目 | Rancher（Docker 单节点） | Lens / FreeLens（Windows） |
|------|--------------------------|----------------------------|
| CPU | ≥ 2 核（推荐 4） | 普通 PC 即可 |
| 内存 | **≥ 4GB 专供 Rancher**（推荐预留 8GB） | 约 200MB～1GB |
| 磁盘 | ≥ 20GB | 约 200MB 安装包 |
| 系统 | Linux 主机 + Docker，或 Windows Docker Desktop / WSL2 | Windows 10/11 |
| 网络 | 能拉 `rancher/rancher` 镜像；能访问下游集群 API（6443） | 能访问集群 API Server |

> 你的 kubeadm 集群 master 静态 IP 若为 `10.13.31.10`，Windows / WSL 需能访问 `https://10.13.31.10:6443`（此前已配 k8snet）。

---

## 三、安装 Rancher（单节点 Docker，适合学习）

### 3.1 安装位置选择

| 方案 | 说明 | 推荐度 |
|------|------|--------|
| **A. 单独 Multipass VM 跑 Rancher** | 不跟业务集群抢资源 | ⭐⭐⭐⭐⭐ |
| **B. Windows Docker Desktop** | 本机直接 `docker run` | ⭐⭐⭐（注意 Hyper-V / 端口占用） |
| **C. 装在 k8s-master 上** | 简单但不推荐，抢 master 资源 | ⭐ |

下文以 **方案 A** 为主（与教程方案 C 一致）。

### 3.2 创建一台专供 Rancher 的虚拟机

**PowerShell：**

```powershell
multipass launch 24.04 --name rancher --cpus 2 --memory 4G --disk 40G
multipass shell rancher
```

若希望固定 IP（与集群互通），可参照主教程 **6.3**，把 Rancher VM 也挂到 `k8snet`，例如分配 `10.13.31.20`。

### 3.3 安装 Docker（在 rancher VM 内）

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
```

退出重新登录或执行 `newgrp docker`，验证：

```bash
docker version
```

### 3.4 国内环境：准备拉镜像

默认镜像：`rancher/rancher`（Docker Hub）。若超时，可用 DaoCloud 等加速，例如：

```bash
# 示例：先拉再标记（地址以你能访问的镜像站为准）
sudo docker pull docker.m.daocloud.io/rancher/rancher:v2.9.3
sudo docker tag docker.m.daocloud.io/rancher/rancher:v2.9.3 rancher/rancher:v2.9.3
```

> 版本号请到 [Rancher Releases](https://github.com/rancher/rancher/releases) 选稳定版；学习环境也可尝试 `rancher/rancher:latest`（不够稳定）。

### 3.5 启动 Rancher 容器

```bash
sudo mkdir -p /opt/rancher
sudo docker run -d --restart=unless-stopped \
  --name rancher \
  -p 80:80 -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:v2.9.3
```

| 参数 | 含义 |
|------|------|
| `-p 80:80 -p 443:443` | Web UI 端口 |
| `-v /opt/rancher:...` | 持久化 Rancher 数据（勿省略） |
| `--privileged` | 容器内运行本地 K8s 所需 |
| `--restart=unless-stopped` | 重启后自动起来 |

查看日志（首次启动可能要几分钟）：

```bash
sudo docker logs -f rancher
```

看到类似 Bootstrap / listening 相关日志且不再疯狂报错，再访问 UI。

### 3.6 首次登录

1. 浏览器打开：`https://<RANCHER_VM_IP>`  
   - 若 Rancher 在 Multipass 默认网卡：`multipass info rancher` 看 IPv4  
   - 若配置了静态 IP：`https://10.13.31.20`
2. 浏览器会提示自签名证书，选择继续访问（学习环境正常）。
3. 查看初始管理员密码：

```bash
sudo docker logs rancher 2>&1 | grep "Bootstrap Password"
# 或
sudo docker exec rancher cat /var/lib/rancher/bootstrap.passwd
```

4. 登录后设置新管理员密码，并填写 Rancher Server URL（通常填你访问用的 `https://IP`）。

### 3.7 Windows Docker Desktop 备选（可选）

若不想再开 VM，在 **Windows PowerShell**（Docker Desktop 已运行）：

```powershell
docker pull rancher/rancher:v2.9.3
docker run -d --restart=unless-stopped `
  --name rancher `
  -p 80:80 -p 443:443 `
  -v ${env:USERPROFILE}\rancher-data:/var/lib/rancher `
  --privileged `
  rancher/rancher:v2.9.3
```

浏览器访问：`https://localhost`。  
注意：需能访问下游集群 `10.13.31.10:6443`（Docker Desktop 与 Hyper-V 网络时要注意连通性）。

### 3.8 停止与清理

```bash
sudo docker stop rancher
sudo docker start rancher

# 彻底删除（数据一并清掉请删除 /opt/rancher）
sudo docker rm -f rancher
```

---

## 四、把现有 kubeadm 集群导入 Rancher

前提：Rancher UI 已能打开，且你的 **k8s-master API 对 Rancher 可达**（同网段 `10.13.31.0/24` 最优）。

### 4.1 在 UI 中导入

1. Rancher 首页 → **Cluster Management** → **Import Existing**（或「导入已有」）
2. 选择 Generic / Custom（通用 Kubernetes）
3. 给集群起名，例如 `cka-lab`
4. 页面会生成类似 `kubectl apply -f https://xxxx` 的命令

### 4.2 在业务集群执行注册命令

在能执行管理员权限的环境运行（推荐 **k8s-master** 或已配好 admin kubeconfig 的 WSL）：

```bash
# 复制 Rancher UI 给出的完整命令，例如：
curl --insecure -sfL https://<RANCHER_IP>/v3/import/xxxxxx.yaml | kubectl apply -f -
```

国内若无法访问 Rancher 的临时 URL，可在 Rancher UI 下载 YAML，拷到 master 上 `kubectl apply -f`。

### 4.3 验证

```bash
kubectl get pods -n cattle-system
# cattle-cluster-agent / cattle-node-agent 等变为 Running
```

Rancher UI 中该集群状态变为 **Active**。

### 4.4 导入成功后可以做什么

| 操作 | 说明 |
|------|------|
| 切命名空间看工作负载 | 等价于 kubectl get deploy/pod |
| kubectl Shell（部分版本） | 在浏览器里敲命令 |
| 项目管理 / 成员 | 体验多用户权限模型 |
| 安装监控等 | 可选，学习环境可暂不装 |

> **CKA 备考：** Rancher 不是考试内容。考试仍以 **kubectl + 官方文档** 为准。Rancher 用于加深「生产里人怎么管集群」的理解。

---

## 五、Windows 安装 Lens / FreeLens

### 5.1 推荐：安装 FreeLens（免费、开源）

**方式一：winget（推荐）**

```powershell
winget install freelens
```

**方式二：GitHub 安装包**

1. 打开 [FreeLens Releases](https://github.com/freelensapp/freelens/releases)
2. 下载 `Freelens-*-windows-amd64.msi` 或 `.exe`
3. 双击安装，完成从开始菜单启动

### 5.2 可选：安装官方 Lens Desktop

1. 打开 [https://k8slens.dev](https://k8slens.dev) 下载 Windows 安装包
2. 运行 `Lens-Setup-*.exe`
3. 默认路径类似：`%LOCALAPPDATA%\Programs\Lens`
4. 按提示注册 / 试用（商业许可策略以官网为准）

### 5.3 不要装 OpenLens（除非你清楚风险）

OpenLens 曾为免费开源分支，**当前维护停滞**，新环境请优先 **FreeLens** 或官方 Lens。

---

## 六、用 Lens / FreeLens 连接现有集群

### 6.1 准备 Windows 侧 kubeconfig

你的 Multipass 集群 kubeconfig 在 master：`~/.kube/config`。  
**不要手动复制粘贴大段 base64**，用传输命令：

**PowerShell：**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.kube"
multipass transfer k8s-master:/home/ubuntu/.kube/config "$env:USERPROFILE\.kube\config"
```

确认文件完整且含 `client-key-data`：

```powershell
Select-String -Path "$env:USERPROFILE\.kube\config" -Pattern "client-key-data|server:"
```

`server` 应为：

```yaml
server: https://10.13.31.10:6443
```

若是 `127.0.0.1`，用编辑器改为 `10.13.31.10`。

### 6.2 网络连通性

```powershell
Test-NetConnection 10.13.31.10 -Port 6443
# TcpTestSucceeded : True
```

失败时检查：Hyper-V `k8snet`、主机 `vEthernet (k8snet)` 是否有 `10.13.31.1`、防火墙是否放行。

### 6.3 打开 FreeLens / Lens

1. 启动应用，一般会自动发现 `%USERPROFILE%\.kube\config`
2. 左侧出现集群（context 名常为 `kubernetes-admin@kubernetes`）
3. 点击集群 → 可浏览 Nodes / Workloads / Network / Config / Events

| 功能 | 用途 |
|------|------|
| **Pods** | 看 Ready、重启次数、所属节点 |
| **Logs** | 实时日志，替代反复 `kubectl logs` |
| **Shell** | 进容器调试 |
| **Port Forward** | UI 转发到本机端口 |
| **Events** | ImagePullBackOff 等一眼可见 |

### 6.4 同时使用 WSL 的 kubectl 与 Windows Lens

| 环境 | kubeconfig |
|------|------------|
| WSL | `~/.kube/config`（Linux） |
| Windows Lens | `%USERPROFILE%\.kube\config` |

两边各放一份完整 config（`multipass transfer` 两次或文件复制），互不影响。

---

## 七、Rancher 与 Lens 日常能做什么

### 7.1 用 Lens 完成一次「可视化小练」

对照你教程里 `dev` 命名空间的 `web` Deployment：

1. 在 FreeLens 切到集群 → Namespace: `dev`
2. 打开 Deployments → 查看 `web` 副本数、镜像
3. 打开 Pods → 进某个 Pod 看 Logs
4. Services → 看 NodePort / ClusterIP
5. 对比：同一时间用 `kubectl get pods -n dev -o wide`，理解 UI 与 API 等价

### 7.2 用 Rancher 完成一次「平台小练」

1. 导入集群成功后进入集群
2. 创建命名空间 / 部署工作负载（可用表单，不必只写 YAML）
3. 查看集群节点资源使用（若启用了监控）
4. 体会：同一集群可被 **kubectl + Lens + Rancher** 同时管理

### 7.3 明确边界

| 能做 | 不能替代 |
|------|----------|
| 看得更清楚、改得更快 | **CKA 考试场景**（纯终端 + 文档） |
| 多集群权限与交付 | `kubeadm`、证书、etcd 底层排查仍要 CLI |

---

## 八、业界其他常用与前沿工具

按用途分组，便于选型：

### 8.1 桌面可视化 / IDE

| 工具 | 特点 | 适合谁 |
|------|------|--------|
| **FreeLens** | 开源活跃，体验接近经典 Lens | 个人免费首选 |
| **Lens Desktop** | 官方、生态与支持完善 | 企业采购 |
| **Headlamp** | CNCF 相关项目，浏览器或桌面版 Kubernetes UI | 想要更「官方感」的 Dashboard |
| **Aptakube** | 现代 UI，多集群体验好 | 偏好精致桌面客户端 |
| **k9s** | **终端 TUI**，键盘流，极快 | 习惯 CLI、服务器无图形界面 |

```bash
# k9s 示例（WSL / Linux）
# 安装后直接进入交互界面
k9s
```

### 8.2 Web Dashboard / 控制台

| 工具 | 特点 |
|------|------|
| **Kubernetes Dashboard** | 官方 Web UI，权限需严格收紧 |
| **Rancher** | 多集群治理平台，超出单一 Dashboard |
| **Portainer** | 容器与 Swarm / K8s 混合场景常用 |

### 8.3 GitOps / CD（前沿且已成业界主流）

| 工具 | 特点 |
|------|------|
| **Argo CD** | GitOps 声明式交付，应用状态与 Git 对齐 |
| **Flux** | CNCF 毕业项目，GitOps 持续同步 |
| **Tekton / Jenkins X** | 流水线与云原生 CI |

> 「集群管起来」之外，现代团队更强调 **Git 即真相**：改库 → Flux/Argo 自动同步到集群。

### 8.4 可观测与策略

| 工具 | 领域 |
|------|------|
| **Prometheus + Grafana** | 指标监控 |
| **Loki / EFK** | 日志 |
| **OpenTelemetry** | 统一可观测标准 |
| **Kyverno / OPA Gatekeeper** | 策略即代码、准入控制 |
| **Falco** | 运行时安全 |

### 8.5 AI / 智能化运维（新兴方向）

| 方向 | 示例 |
|------|------|
| 对话式运维 | 基于 LLM 的集群助手（辅助写 YAML、解释 Events） |
| 异常建议 | 部分商业平台对 CrashLoop / OOM 给出修复建议 |
| 注意 | **考试与生产关键操作仍需人工确认**，AI 输出必须复核 |

### 8.6 快速对照

| 你想… | 优先试 |
|-------|--------|
| 本机图形看 Pod/日志 | **FreeLens** |
| 团队多集群 + 权限 | **Rancher** |
| 纯键盘终端 | **k9s** |
| 声明式持续交付 | **Argo CD / Flux** |
| CKA 过考 | **kubectl + 官方文档**（工具仅辅助） |

---

## 九、选型建议（备考 / 开发 / 运维）

```
CKA 备考
  └─ 主练 kubectl；可装 FreeLens 加快理解，但考试不要依赖

日常开发调试
  └─ FreeLens / Aptakube / k9s

公司多套集群运维
  └─ Rancher（或云厂商控制台：ACK / EKS / GKE）+ GitOps

学习路径建议
  1. 先跑通 kubectl（主教程七、八节）
  2. 再装 FreeLens 对照资源
  3. 机器有余力再装 Rancher 导一次自建集群
```

---

## 十、常见问题

### 10.1 Rancher 拉镜像 / 启动失败

```bash
sudo docker logs rancher | tail -100
```

| 现象 | 处理 |
|------|------|
| Image pull timeout | 配置 Docker Hub 镜像加速或手动 pull + tag |
| 端口被占用 | 改 `-p 8080:80 -p 8443:443` 并从新端口访问 |
| 内存不足 | 给 VM 加到 6～8GB，或停掉不用的 Kind/Minikube |

### 10.2 Lens 连不上集群

| 检查 | 命令 / 动作 |
|------|-------------|
| kubeconfig 是否完整 | 必须有 `client-key-data` |
| server 地址 | 应为 `https://10.13.31.10:6443` |
| 网络 | `Test-NetConnection 10.13.31.10 -Port 6443` |
| 证书过期 | master 上 `sudo kubeadm certs check-expiration` |

### 10.3 Rancher 导入后 Agent 一直 Pending

```bash
kubectl describe pod -n cattle-system
kubectl get nodes
```

常见原因：节点资源不足、镜像拉不到、CNI 未 Ready。处理思路与排查业务 Pod 相同（describe → Events）。

### 10.4 Rancher 与 Lens 同时管一个集群安全吗

可以。两者都是 **API Server 的普通客户端**（持不同或相同凭证）。生产环境应：

- 给 UI 工具单独账号 + RBAC 最小权限  
- 勿把 `cluster-admin` kubeconfig 拷到多台不受控电脑  

### 10.5 卸载 FreeLens / Lens

Windows「设置 → 应用」卸载即可；kubeconfig **不会**被删除。

---

## 附录 A：命令速查

```powershell
# FreeLens
winget install freelens

# 导出 kubeconfig 到 Windows（供 Lens）
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.kube"
multipass transfer k8s-master:/home/ubuntu/.kube/config "$env:USERPROFILE\.kube\config"
Test-NetConnection 10.13.31.10 -Port 6443
```

```bash
# Rancher（在 rancher VM）
sudo docker run -d --restart=unless-stopped --name rancher \
  -p 80:80 -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:v2.9.3

sudo docker logs rancher 2>&1 | grep "Bootstrap Password"
```

---

## 附录 B：官方文档链接

| 项目 | 链接 |
|------|------|
| Rancher 安装总览 | https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade |
| Rancher Docker 单节点 | https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/other-installation-methods/rancher-on-a-single-node-with-docker |
| Lens 安装 | https://docs.k8slens.dev/getting-started/install-lens/ |
| FreeLens | https://github.com/freelensapp/freelens |
| Headlamp | https://headlamp.dev/ |
| Argo CD | https://argo-cd.readthedocs.io/ |
| Flux | https://fluxcd.io/ |
| k9s | https://k9scli.io/ |

---

**总结：**  
- **Rancher** = 多集群「运维平台」——建群、纳管、权限、应用目录。  
- **Lens / FreeLens** = 本机「图形化 kubectl」——看资源、日志、Shell。  
- 备考仍以 **kubectl** 为主；工具用来提效和理解生产实践。  
- 业界另有 **k9s、Headlamp、Argo CD/Flux、Kyverno** 等，可按阶段扩展学习。
