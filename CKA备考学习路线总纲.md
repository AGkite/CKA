# CKA 备考学习路线总纲

> 本文是整套 CKA 备考教程的「导航 + 学习安排」。  
> 适用环境：已按 [第一部分-本机搭建K8s集群教程.md](./第一部分-本机搭建K8s集群教程.md) 搭好 Multipass + kubeadm 三节点集群。  
> 知识点索引：[CKA学习知识点总结.md](./CKA学习知识点总结.md)

---

## 一、这套教程怎么用

整套教程按 **CKA 官方考纲五大域** 拆成 6 个部分，循序渐进，全部默认「零基础」讲解：**先讲概念 → 再讲命令和每个参数的含义 → 最后给动手练习和考试查文档技巧**。

| 部分 | 文档 | 对应考纲域 | 权重 |
|------|------|------------|------|
| 第一部分 | [第一部分-本机搭建K8s集群教程.md](./第一部分-本机搭建K8s集群教程.md) | 环境搭建（前置） | — |
| 第二部分 | [第二部分-工作负载与调度.md](./第二部分-工作负载与调度.md) | 工作负载与调度 | 15% |
| 第三部分 | [第三部分-服务与网络.md](./第三部分-服务与网络.md) | 服务与网络 | 20% |
| 第四部分 | [第四部分-存储.md](./第四部分-存储.md) | 存储 | 10% |
| 第五部分 | [第五部分-集群架构安装与配置.md](./第五部分-集群架构安装与配置.md) | 集群架构、安装与配置 | 25% |
| 第六部分 | [第六部分-故障排查.md](./第六部分-故障排查.md) | 故障排查 | 30% |
| 第七部分 | [第七部分-Rancher与Lens管理K8s集群教程.md](./第七部分-Rancher与Lens管理K8s集群教程.md) | 可视化（非考点） | — |

> **学习顺序建议：** 二 → 三 → 四 → 五 → 六。  
> 工作负载是一切的基础；网络、存储是常见题型；集群架构与故障排查合计 **55%**，是过关关键，放在有基础后主攻。

---

## 二、考试速览（先建立目标感）

| 项目 | 内容 |
|------|------|
| 形式 | 纯实操，浏览器里连真实集群敲命令 |
| 时长 | 120 分钟 |
| 题量 | 约 17–25 题，每题独立计分 |
| 及格 | 66% |
| 允许查文档 | 仅 `kubernetes.io/docs`、`kubernetes.io/blog`、`github.com/kubernetes` |
| 环境 | Ubuntu 终端 + kubeadm 集群 |

**三条核心策略：**

1. **会做的先做**，超过 8 分钟没解出的题标记跳过，第二趟再回头。
2. **每题做完必验证**（`kubectl get` / `describe` / `auth can-i`）。
3. **熟练查官方文档**：几乎所有 YAML 都能在文档里找到模板复制修改，不必背全。

---

## 三、五大域权重与主攻优先级

```
故障排查 ██████████████████████████████ 30%   ← 第六部分
集群架构 █████████████████████████ 25%          ← 第五部分
服务网络 ████████████████████ 20%               ← 第三部分
工作负载 ███████████████ 15%                    ← 第二部分
存储     ██████████ 10%                         ← 第四部分
```

| 优先级 | 域 | 理由 |
|--------|----|----|
| ⭐⭐⭐⭐⭐ | 故障排查 | 占比最高，且需要前面所有知识 |
| ⭐⭐⭐⭐⭐ | 集群架构（etcd 备份、升级、RBAC） | 单题分值高，流程固定，练熟就是稳分 |
| ⭐⭐⭐⭐ | 服务与网络 | NetworkPolicy、Service 排错高频 |
| ⭐⭐⭐ | 工作负载 | 基础，必须先掌握 |
| ⭐⭐⭐ | 存储 | 占比小但题型固定，容易拿分 |

---

## 四、6 周学习安排（可按自身节奏压缩/拉长）

> 假设你已完成第一部分（集群搭好、`kubectl get nodes` 三节点 Ready）。

### 第 1 周：工作负载与调度（第二部分）

| 天 | 目标 |
|----|------|
| 1–2 | Pod / Deployment / ReplicaSet 概念，`kubectl run/create/get/describe` |
| 3 | ConfigMap / Secret 三种注入方式 |
| 4 | 资源 requests/limits、探针、多容器 Pod |
| 5 | 调度：nodeSelector、affinity、taint/toleration |
| 6 | DaemonSet / Job / CronJob / 静态 Pod |
| 7 | 复盘 + `--dry-run=client -o yaml` 生成 YAML 练习 |

**周目标：** 能 3 分钟内用命令生成并改好一个 Deployment。

### 第 2 周：服务与网络（第三部分）

| 天 | 目标 |
|----|------|
| 1 | Service 四种类型、port/targetPort/nodePort |
| 2 | Endpoints、selector 与 label 匹配排错 |
| 3 | CoreDNS、集群 DNS 命名规则 |
| 4 | Ingress + IngressController 概念与 YAML |
| 5 | NetworkPolicy（Ingress/Egress，放行 DNS） |
| 6 | Gateway API 概念了解 |
| 7 | 复盘 + Service/NetworkPolicy 综合练习 |

### 第 3 周：存储 + RBAC（第四部分 + 第五部分前半）

| 天 | 目标 |
|----|------|
| 1–2 | PV / PVC / StorageClass、访问模式、回收策略 |
| 3 | 静态供应 vs 动态供应、volumeBindingMode |
| 4 | Pod 挂载 PVC、ConfigMap/Secret 卷 |
| 5–6 | RBAC 四件套 + `auth can-i` 验证 |
| 7 | 复盘 |

### 第 4 周：集群架构与运维（第五部分）

| 天 | 目标 |
|----|------|
| 1 | K8s 架构、控制面/工作节点组件、静态 Pod |
| 2–3 | **etcd 备份与恢复**（反复练到闭眼能敲） |
| 4–5 | **kubeadm 升级**（控制面 + 工作节点） |
| 6 | 证书续期、HA 概念、Helm/Kustomize |
| 7 | 复盘：etcd + upgrade 各限时练 3 遍 |

### 第 5 周：故障排查专项（第六部分）

| 天 | 目标 |
|----|------|
| 1 | 通用排查流程、Pod 状态排查 |
| 2 | 节点 NotReady 排查（kubelet/containerd） |
| 3 | 控制面组件故障（apiserver/etcd/manifest） |
| 4 | 网络故障排查（Service/DNS/kube-proxy） |
| 5 | 综合故障场景演练 |
| 6–7 | 限时模拟题（killercoda / Killer Shell） |

### 第 6 周（考前）：查漏补缺 + 模拟考

| 天 | 目标 |
|----|------|
| 1–3 | Killer Shell 官方模拟（报名后免费送 2 次） |
| 4–5 | 弱项专项复练 |
| 6 | 熟记 etcd/upgrade 模板、别名、时间分配 |
| 7 | 休息，检查考试环境（PSI、摄像头、证件） |

---

## 五、每份教程的统一结构

为方便学习，第二～六部分都遵循同一套结构：

1. **本章考什么**：对应考纲域与常见题型
2. **概念详解**：零基础解释每个术语
3. **命令与参数逐项说明**：每个命令做什么、每个 flag 什么意思
4. **动手练习**：在你现有集群上直接跑
5. **考试查文档指南**：这类题在 `kubernetes.io/docs` 的哪一页
6. **易错点与自检清单**

---

## 六、贯穿全程的通用技巧

### 6.1 必配的 kubectl 提速

```bash
alias k=kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k
export do='--dry-run=client -o yaml'   # k run nginx --image=nginx $do
export now='--force --grace-period=0'  # 立即删除 Pod
```

| 别名/变量 | 作用 |
|-----------|------|
| `k` | 少敲 6 个字符，一天下来省很多时间 |
| `completion` | Tab 自动补全资源名 |
| `$do` | 快速生成 YAML 模板 |
| `$now` | 强制立即删除，不等优雅终止 |

### 6.2 生成 YAML 而不是手写

考试几乎不用从零手写 YAML：

```bash
# 生成 Deployment 骨架再改
kubectl create deployment web --image=nginx $do > web.yaml

# 生成 Pod 骨架
kubectl run test --image=busybox $do > pod.yaml
```

### 6.3 查文档比背更重要

考试允许开 `kubernetes.io/docs`。把这些页面收藏（各部分教程会重复强调）：

| 主题 | 文档路径 |
|------|----------|
| kubectl 速查 | `/docs/reference/kubectl/quick-reference/` |
| etcd 备份 | `/docs/tasks/administer-cluster/configure-upgrade-etcd/` |
| kubeadm 升级 | `/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/` |
| RBAC | `/docs/reference/access-authn-authz/rbac/` |
| NetworkPolicy | `/docs/concepts/services-networking/network-policies/` |
| PV/PVC | `/docs/concepts/storage/persistent-volumes/` |
| Ingress | `/docs/concepts/services-networking/ingress/` |

### 6.4 用自己集群做「练习靶场」

你已有的 `dev` 命名空间和 `web` Deployment 就是练手对象。建议再建几个命名空间隔离练习：

```bash
kubectl create ns cka-workload
kubectl create ns cka-network
kubectl create ns cka-storage
```

> 国内环境注意：业务镜像用国内源（如 `docker.m.daocloud.io/library/nginx:1.27`），或按第一部分给三台节点配好 containerd 镜像加速，避免练习时反复 `ImagePullBackOff`。

---

## 七、准备就绪自检

开始第二部分前，确认：

- [ ] `kubectl get nodes` 三节点均 Ready
- [ ] `kubectl get pods -A` 控制面与 Calico 均 Running
- [ ] 会用 `kubectl get/describe/logs`
- [ ] 配好 `alias k=kubectl` 和 `$do`
- [ ] 能访问 `kubernetes.io/docs`

准备好后，从 [第二部分-工作负载与调度.md](./第二部分-工作负载与调度.md) 开始。

---

*配套：第一部分（搭建）→ 第二~六部分（考纲）→ 知识点总结（速查）*
