# Agent Sandbox 开发能力速成清单（Go 专家版）

> **前提：** 已熟练掌握 Go（接口、并发、错误处理、HTTP/gRPC、测试）  
> **目标：** 能在 3～6 周内独立开发/改造 Agent Sandbox 类系统（CRD + Controller + SDK 集成）  
> **策略：** 只学「沙箱开发用到的 20% K8s」，其余用 agent-sandbox 源码当教材

---

## 总览：你要补的不是 Go，是这三块

| 模块 | 占比 | 你要达到的水平 |
|------|------|----------------|
| **Kubernetes 使用层** | 30% | 能读 events、调 Pod/PVC/NetworkPolicy |
| **K8s 扩展开发（CRD + Controller）** | 50% | 能写 Reconcile、改 agent-sandbox 源码 |
| **Agent 沙箱业务语义** | 20% | 理解 Sandbox/WarmPool/Claim 与用户旅程 |

**每日建议投入：** 2～3 小时 → 约 4 周可开发 MVP；6 周可改 agent-sandbox 并提 PR 级别改动。

---

## 第 0 周（2～3 天）：环境与肌肉记忆

### 目标
本地能跑通 agent-sandbox，建立「改代码 → 部署 → 验证」闭环。

### 清单

- [ ] 安装工具链：`kubectl`、`kind`（或 minikube）、`docker`
- [ ] 克隆仓库：`git clone https://github.com/kubernetes-sigs/agent-sandbox.git`
- [ ] 阅读（只读，不深挖）：[README](https://github.com/kubernetes-sigs/agent-sandbox/blob/main/README.md)、[Getting Started](https://agent-sandbox.sigs.k8s.io/docs/getting_started/)
- [ ] KinD 起集群 + 安装 controller（release manifest 或 `make deploy-kind`）
- [ ] 跑通最小 Sandbox：

```yaml
apiVersion: agents.x-k8s.io/v1beta1
kind: Sandbox
metadata:
  name: hello-sandbox
spec:
  podTemplate:
    spec:
      containers:
      - name: main
        image: busybox:1.36
        command: ["sleep", "3600"]
```

- [ ] 熟练以下命令（每天敲直到不用想）：

```bash
kubectl get sandbox,pod,pvc,events -A
kubectl describe sandbox hello-sandbox
kubectl logs <pod>
kubectl exec -it <pod> -- sh
kubectl delete sandbox hello-sandbox
```

### 验收标准
- [ ] 能解释：apply Sandbox 后集群里多了哪些对象（Sandbox CR → Pod → 可选 PVC/Service）
- [ ] 能在 10 分钟内从零部署一个 Running 的 Sandbox

### 推荐资源
- [Kubernetes 官方教程 - 概念](https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/)（只学 Module 1～4：Pod、Deployment、Service、Namespace）
- agent-sandbox 示例：[examples/](https://github.com/kubernetes-sigs/agent-sandbox/tree/main/examples)

---

## 第 1 周：K8s 沙箱必备概念（不泛泛学 K8s）

### 目标
理解 agent-sandbox 依赖的 K8s 原语，能排查 Pod Pending/CrashLoop。

### 必学概念（按优先级）

| 优先级 | 概念 | 与 agent-sandbox 的关系 |
|--------|------|-------------------------|
| P0 | Pod / Container spec | Sandbox 最终变成 Pod |
| P0 | Namespace / RBAC | 多租户、Controller 权限 |
| P0 | PVC / StorageClass | 持久化、suspend/resume |
| P1 | Service（Headless） | 稳定网络身份 |
| P1 | ResourceQuota / LimitRange | 防资源打满 |
| P1 | NetworkPolicy | 沙箱出网隔离 |
| P2 | RuntimeClass | gVisor/Kata 强隔离 |
| P2 | Finalizer / OwnerReference | Controller 清理逻辑 |

### 动手练习

- [ ] **练习 1：** 手动 `kubectl run` 创建 Pod，挂 emptyDir 和 PVC，观察差异
- [ ] **练习 2：** 给 Namespace 加 `ResourceQuota`，看 Sandbox 创建失败时的 events
- [ ] **练习 3：** 写一条 deny-all egress 的 NetworkPolicy，验证 Pod 无法 curl 外网
- [ ] **练习 4：** 读 agent-sandbox 生成的 Pod YAML（`kubectl get pod -o yaml`），对照 Sandbox spec

### 验收标准
- [ ] Pod Pending 时，能用 `describe` + `events` 说出 3 种常见原因（调度、镜像、PVC）
- [ ] 能画出：`Sandbox spec` → `Pod spec` 的字段映射关系

### 可跳过（现阶段）
Deployment 滚动更新、Ingress 细节、Helm 模板语法、Service Mesh

---

## 第 2 周：CRD + Controller 开发（核心）

### 目标
理解 Reconcile 模式，能读懂并小改 agent-sandbox controller。

### 学习路径

#### Day 1～2：CRD 与 API 类型

- [ ] 阅读 `api/v1beta1/` 下 Go struct（Sandbox、SandboxTemplate 等）
- [ ] 对照 `config/crd/` 或 release 里的 CRD YAML
- [ ] 理解：`spec`（期望） vs `status`（实际）

**关键认知：** CRD 只是 schema；业务在 Controller。

#### Day 3～4：controller-runtime 模式

- [ ] 阅读 [controller-runtime 官方 Book（Quick Start）](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial)
- [ ] 或 kubebuilder tutorial 前几章（4～6 小时）
- [ ] 对照 agent-sandbox `controllers/` 目录，找 Reconcile 入口

**Reconcile 心智模型（背下来）：**

```
1. 获取对象
2. 已删除？→ 清理子资源（Pod/PVC）→ 去 Finalizer
3. 否则 ensure 子资源（Pod 存在且符合 spec）
4. 更新 status
5. 错误 → return Requeue
```

#### Day 5～7：读 agent-sandbox 源码

按顺序读（每文件记：输入、输出、副作用）：

- [ ] `controllers/sandbox_controller.go`（或等价路径）— 核心生命周期
- [ ] `internal/` 下 Pod/PVC 构建逻辑
- [ ] extensions：`SandboxWarmPool`、`SandboxClaim` controller（若装 extensions）
- [ ] `cmd/agent-sandbox-controller/main.go` — 启动、flag、并发配置

**边读边做笔记：**

| 问题 | 你的答案 |
|------|----------|
| 创建 Sandbox 时谁创建 Pod？ | |
| 删除 Sandbox 时 Pod 怎么删？ | |
| suspend 和 delete 区别？ | |
| status.Ready 何时变为 true？ | |

### 动手项目（最小 Controller）

- [ ] 用 kubebuilder 初始化项目：`SandboxLite`，spec 只有 `image` + `ttlSeconds`
- [ ] 实现 Reconcile：创建 Pod，TTL 到期 delete Pod
- [ ] 加 Finalizer：删 CR 时保证 Pod 被删
- [ ] 写 1 个 envtest 或 KinD e2e

**若时间紧：** 跳过自研，直接在 agent-sandbox 里加一个小 feature（见第 4 周）。

### 验收标准
- [ ] 能口头讲清 agent-sandbox 的 Reconcile 主路径（10 分钟）
- [ ] 能独立写一个「CR → Pod」的最小 Controller（100～200 行级）

### 推荐资源
- [Programming Kubernetes（O'Reilly）](https://www.oreilly.com/library/view/programming-kubernetes/9781492041430/) — 第 3～7 章
- [kubebuilder book](https://book.kubebuilder.io/)
- agent-sandbox 源码 + [DeepWiki](https://deepwiki.com/kubernetes-sigs/agent-sandbox)

---

## 第 3 周：Agent Sandbox 扩展能力 + SDK

### 目标
掌握 WarmPool/Claim 业务流，会用 SDK 从应用侧调沙箱。

### 清单

#### 扩展 CRD（生产常见）

- [ ] 部署 `extensions.yaml`
- [ ] 跑通 WarmPool + Claim 示例（`extensions/examples/`）
- [ ] 理解三者的业务关系：

```
SandboxTemplate（房型）
    ↑
SandboxWarmPool（预热的空房）
    ↑
SandboxClaim（用户领钥匙）
    → 绑定到一个 Sandbox
```

- [ ] 读 [configuration.md](https://github.com/kubernetes-sigs/agent-sandbox/blob/main/docs/configuration.md)，理解并发 flag

#### Python / Go SDK

- [ ] 部署 sandbox-router（SDK 前置）
- [ ] 跑 Python SDK `test_client.py`（Dev Mode / KinD）
- [ ] 理解四种连接模式：Gateway / Tunnel / In-Cluster / Direct

#### OpenClaw 业务用例

- [ ] 读 [OpenClaw use case 文档](https://agent-sandbox.sigs.k8s.io/docs/use-cases/openclaw/)
- [ ] 跑 `examples/openclaw-sandbox/`（理解 always-on 模式）

### 动手练习

- [ ] **练习 A：** 创建一个 WarmPool（size=3），连续 Claim 3 次，测分配延迟
- [ ] **练习 B：** 用 SDK `create_sandbox` → `commands.run` → `terminate` 跑通
- [ ] **练习 C：** 给 Sandbox 加 TTL，验证自动回收

### 验收标准
- [ ] 能向业务方解释 WarmPool 为何能降低冷启动（结合实测延迟）
- [ ] 能写一段 Go/Python 调用 SDK 完成「创建沙箱 → 执行命令 → 销毁」

---

## 第 4 周：生产级能力 + 实战项目

### 目标
具备改 production 代码的意识：幂等、测试、安全、观测。

### 必学生产话题（每个 2～4 小时）

| 话题 | 做什么 |
|------|--------|
| **幂等与并发** | 同一 Sandbox apply 两次，Pod 数量仍为 1 |
| **Finalizer** | 删 CR 卡在 Terminating 时怎么查 |
| **Status 更新** | 为何 patch status 要用 subresource |
| **Leader Election** | Controller 多副本为何不重复建 Pod |
| **Metrics** | 找 controller 暴露的 Prometheus 指标 |
| **Webhook（选学）** | Validating CR spec 非法字段 |

### 故障排查 Drill（必做）

人为制造故障并修复：

- [ ] 删 Pod 不删 Sandbox → Controller 是否重建？
- [ ] 镜像 pull 失败 → status 显示什么？
- [ ] PVC 无法绑定 → Sandbox 卡在哪？
- [ ] apiserver 限流 → 调 `--kube-api-qps` 观察

### 毕业项目（三选一）

**项目 1（推荐）：** 给 agent-sandbox 提一个小 PR  
- 例如：Sandbox status 增加可读 message、文档修正、小 bugfix  
- 读 [CONTRIBUTING.md](https://github.com/kubernetes-sigs/agent-sandbox/blob/main/CONTRIBUTING.md)

**项目 2：** 自研 MVP `SandboxLite` Operator  
- spec：`image`、`cpu`、`memory`、`ttl`  
- 实现：创建 Pod + TTL 清理 + Finalizer  
- 部署到 KinD + README

**项目 3：** 平台集成层（Go HTTP API）  
- `POST /sandboxes` → 创建 Sandbox CR  
- `POST /sandboxes/{id}/exec` → client-go exec  
- `DELETE /sandboxes/{id}`  
- 不接 CRD 也行，为后续演进打基础

### 验收标准（开发能力达标）

- [ ] 能在 1 天内实现一个可 demo 的沙箱 feature
- [ ] 能写 KinD e2e 或 envtest 覆盖 happy path
- [ ] 能列出自己 feature 的 5 条生产 DoD（幂等、泄漏、并发、安全、观测）

---

## 生产级 Definition of Done（每个 feature 必过）

开发任何沙箱相关功能时，用这张表自检：

| # | 检查项 | 通过标准 |
|---|--------|----------|
| 1 | 幂等 | 同一 spec 重复 apply，资源不 duplicated |
| 2 | 清理 | 删 CR 后 5 分钟内无残留 Pod/PVC |
| 3 | 并发 | 10 个对象同时 create/delete 无 panic |
| 4 | 失败可恢复 | 手动删 Pod 后 Controller 重建 |
| 5 | 状态准确 | status 反映 Ready/Failed/Pending |
| 6 | 安全默认 | 非 root、有 resource limit（NetworkPolicy 视场景） |
| 7 | 可观测 | 失败能在 logs/events/metrics 定位 |

---

## 每日学习模板（2～3 小时）

```
[30min] 概念：读文档/书的一节
[60min] 动手：kubectl 或改代码部署到 KinD
[30min] 源码：读 agent-sandbox 一个函数并做笔记
[30min] 复盘：写 3 句话总结 + 1 个未解决问题
```

**配合 AI 的正确姿势：**

- ✅ 让 AI 根据本文档生成 kubebuilder 脚手架、Reconcile 分函数、测试用例
- ✅ 把 `kubectl describe` / events 贴给 AI 诊断
- ❌ 不让 AI 替代你在 KinD 里验证
- ❌ 内网环境勿上传 kubeconfig、业务 YAML 到公网 AI

---

## 知识地图（Go 专家速查）

```
你已具备                          需新增
────────                          ──────
Go + 并发 + 测试        ──→       client-go / controller-runtime
HTTP API 设计           ──→       K8s API（REST 风格 CRUD）
状态机思维              ──→       Reconcile 循环
依赖注入/接口           ──→       Controller 接口与 mock client
```

**agent-sandbox 技术栈对照：**

| 项目使用 | 你要会 |
|----------|--------|
| Go 1.26 | ✅ 已有 |
| controller-runtime | 第 2 周重点 |
| client-go | 第 1～2 周 |
| CRD / apiextensions | 第 2 周 |
| Prometheus / OTEL | 第 4 周 |
| Python SDK | 第 3 周（调用侧） |

---

## 推荐阅读顺序（最短路径）

1. agent-sandbox README + architecture 图（1h）
2. K8s 官方 Pod/PVC/Namespace 教程（3h）
3. kubebuilder Quick Start（6h）
4. agent-sandbox `controllers/` 源码（8h）
5. extensions + SDK README（4h）
6. OpenClaw example + 业务视角文档（2h）

**总计精学约 24h + 动手 40h ≈ 3～4 周（业余）或 1.5～2 周（全职）**

---

## 里程碑时间表

| 时间 | 里程碑 | 标志 |
|------|--------|------|
| 第 3 天 | 环境通 | KinD + Sandbox Running |
| 第 7 天 | K8s 够用 | 独立排查 Pod 问题 |
| 第 14 天 | Controller 读懂 | 画出 Reconcile 流程图 |
| 第 21 天 | 扩展 + SDK | WarmPool/Claim + SDK 跑通 |
| 第 28 天 | 能开发 | 完成毕业项目之一 |
| 第 42 天 | 能改生产 | PR 或 MVP Operator 带测试 |

---

## 常见弯路（避开能省 1～2 周）

| 弯路 | 正确做法 |
|------|----------|
| 系统学完整 K8s 认证课 | 只学本文 P0/P1 概念 |
| 先写完美 CRD 设计 | 先 client-go 直管 Pod，再抽象 CRD |
| 一上来 clone 全功能 agent-sandbox | 先跑 examples，再读一个 controller |
| 不看 events 只看代码 | 每个 bug 先 `kubectl describe` |
| 跳过 Finalizer/TTL | 删资源泄漏是生产第一坑 |

---

## 下一步行动（今天就开始）

**Day 1 任务（按顺序勾选）：**

- [ ] 安装 kind + kubectl
- [ ] clone agent-sandbox
- [ ] KinD 创建集群
- [ ] apply release manifest（选最新 [release](https://github.com/kubernetes-sigs/agent-sandbox/releases)）
- [ ] apply 一个 busybox Sandbox
- [ ] `kubectl get sandbox,pod -w` 观察创建过程
- [ ] 阅读 `agent-sandbox-business-perspective.md` 巩固业务语境

完成 Day 1 后，从 **第 1 周清单** 继续。

---

## 相关文档

- 同目录：`agent-sandbox-business-perspective.md` — 业务视角
- 官方：https://agent-sandbox.sigs.k8s.io/docs/
- 源码：https://github.com/kubernetes-sigs/agent-sandbox
