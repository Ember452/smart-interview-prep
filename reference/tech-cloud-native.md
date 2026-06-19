# 云原生与平台工程知识库

> 面试高频问题与答案要点。覆盖容器/K8s/可观测性/平台工程方向。

---

## 1. 容器

### 1.1 Docker 核心原理
- **Docker vs 虚拟机** — Docker 共享宿主机操作系统内核，通过 namespace（隔离）和 cgroup（资源限制）实现容器化；VM 需要独立 Guest OS，占用更大、启动更慢。容器即进程，VM 即机器
- **namespace 六大隔离** — PID（进程隔离）、NET（网络栈独立）、IPC、MNT（文件系统挂载点独立）、UTS（主机名隔离）、USER（用户/组 ID 映射）
- **cgroup 资源限制** — cpu.shares（权重分配）/cpu.cfs_quota_us（硬限制）、memory.limit_in_bytes（硬限制，超过 OOM Kill）、blkio 限制磁盘 I/O

### 1.2 镜像优化
- **镜像分层与构建优化** — 每条 RUN 指令产生一层（合并 RUN 减少层数）；COPY 频繁变化的文件放后面（利用缓存）；多阶段构建（builder 阶段编译 → 最终镜像只 copy 二进制，镜像缩小 90%）；选 alpine/slim 基础镜像
- **安全基线** — 不以 root 运行（USER 指令）、最小化攻击面（不装无关工具）、使用受信任基础镜像（Docker Official Image）、扫描漏洞（Trivy/Snyk）、镜像签名（Docker Content Trust/Notary）

---

## 2. Kubernetes

### 2.1 核心组件
- **控制平面组件** — API Server（kubectl 入口/认证授权准入三关/REST）、etcd（KV 存储/cluster 唯一真相源）、Scheduler（为 Pod 选 Node：过滤 + 打分+ 预选优选）、Controller Manager（运行各类控制器：ReplicaSet/Deployment/Node 等，驱动实际状态→期望状态）
- **Node 组件** — kubelet（Pod 生命周期管理/CRI 调容器运行时/上报 Node/Pod 状态）、kube-proxy（Service 四层负载均衡/iptables 或 IPVS 规则）

### 2.2 调度与编排
- **Deployment vs StatefulSet vs DaemonSet** — Deployment：无状态/可随意替换（滚动更新/回滚）；StatefulSet：有状态（有序部署/删除/PVC 生命周期绑定 Pod 不会随 Pod 删除被回收/稳定网络标识 <Name>-<Ordinal>）；DaemonSet：每 Node 一个（日志采集/监控 Agent）
- **HPA vs VPA** — HPA 横向扩缩（指标：CPU/内存/自定义指标→调整 replicas 数量）；VPA 纵向调整资源 requests/limits（重启 Pod 生效，适合不太频繁变动场景）
- **Service 与网络** — ClusterIP（集群内）、NodePort（暴露 Node 固定端口，范围 30000-32767）、LoadBalancer（云厂商 LB 对接）、Ingress（七层路由/tls 终止/path-based 路由，由 Ingress Controller 如 Nginx/Traefik 实现）

### 2.3 故障排查
- **Pod 启动失败排查链** — `kubectl describe pod` 看 Events：ImagePullBackOff→镜像拉取失败（检查 imagePullSecrets/仓库连通性）；CrashLoopBackOff→容器反复崩溃（`kubectl logs --previous` 看前一次启动日志；检查资源限制/探针配置）；Pending→调度不上（看 Events 是否无满足 Node/资源不足/affinity/taint 冲突）
- **监控信号** — 四金信号：Latency（延迟）、Traffic（流量）、Errors（错误率）、Saturation（饱和度：CPU 节流/内存压力/goroutine 堆积）

---

## 3. 服务治理

- **服务发现对比** — 客户端发现（客户端从注册中心拉服务列表 + 客户端侧负载均衡，如 Eureka+Ribbon）；服务端发现（DNS/负载均衡器隐藏后端变化，如 K8s Service。客户端发现更灵活但耦合注册中心）
- **配置中心核心能力** — 配置集中管理（Nacos/Apollo/Consul）、动态刷新（@RefreshScope 或 Watch 机制，无需重启应用）、灰度发布（部分实例先用新配置）、版本回滚
- **熔断/限流/降级** — Sentinel/Hystrix 三态：Closed（正常通行）→失败超过阈值→ Open（快速失败，过一段时间）→ Half-Open（试探性放行一条请求，成功→恢复 Close，失败→继续 Open）。限流：滑动窗口/QPS/并发线程数。降级：高负载时关非核心功能保核心链路

---

## 4. 可观测性

- **三大支柱** — Logs（文本随时间有序记录事件，ELK/Loki）、Metrics（数值测量，Prometheus+TimeSeries DB 聚合/统计）、Traces（一次请求在多个服务中的完整路径/延迟分布，Jaeger/Zipkin/Tempo）。三者互补：Metrics 发现异常，Traces 定位瓶颈服务，Logs 查具体错误
- **Prometheus 核心概念** — Pull 模式（Server 主动拉取 Exporter/metrics endpoint）；PromQL 查询语言；Recording Rules（预计算频繁使用的聚合，减少查询延迟）；Alertmanager（抑制/分组/路由告警）
- **SLO/SLI/SLA** — SLI（服务级别指标：可用性/延迟/错误率，直接可测量）；SLO（服务级别目标：SLI ≤ 目标值，如可用性≥99.9%。内部驱动可靠性决策）；SLA（服务级别协议：SLO + 未达成后果，外部合同/商业承诺）
- **告警降噪** — 告警分组（同一故障产生的多条告警合并为一条通知）、告警抑制（高优先级抑制低优先级；基础设施故障抑制上层服务告警）、告警静默（计划维护期间不发送告警）

---

## 5. 平台工程

- **IaC vs ClickOps** — IaC 声明式配置（Terraform/Pulumi/Crossplane）版本化+可审计+可复现；ClickOps 在控制台手动点击不可复现、无审计记录、团队协作困难
- **GitOps 核心循环** — Git 源仓库为目标状态的唯一真相源 → Operator（Flux/ArgoCD）监控仓库变化 → 自动同步到集群 → 检测并自动回滚漂移
- **多环境管理** — 开发/测试/预发/生产环境一致性：同一套配置模板 + 环境差异化 Variables（数据库地址/规格/副本数）。安全：生产环境专属仓库/集群物理隔离、合入权限分离
- **成本治理** — FinOps 循环：Inform（按标签/命名空间分账归算各团队成本）→ Optimize（识别闲置资源/建议合适规格/Spot 实例降本）→ Operate（持续监控+异常预算告警）

---

## 6. Service Mesh 与 eBPF

- **Service Mesh 架构** — 数据平面（Sidecar Proxy 如 Envoy 随每个 Pod 部署，透明拦截所有进出流量）+ 控制平面（Istio/Linkerd 下发配置/证书/流量策略）。核心能力：mTLS 自动证书管理、流量管理（金丝雀发布/流量镜像/故障注入）、可观测性（自动采集指标/Traces 无需改代码）
- **Service Mesh 的代价** — 每个请求多一跳（Sidecar 延迟 1-5ms）、资源开销（每个 Pod 多 100-200MB 内存）、复杂度上升（调试困难/配置繁琐）。适合：微服务规模大/有严格安全要求/需要精细化流量管理。不适合：服务数量少/对延迟极敏感
- **eBPF 原理与应用** — 在 Linux 内核中安全运行沙箱程序，无需修改内核代码或内核模块。场景：网络（Cilium 替代 kube-proxy，无 Sidecar 的高性能 Service Mesh）、可观测性（系统调用/网络追踪零侵入）、安全（内核级安全策略 Falco/Tetragon）。优势：无侵入、高性能、内核级可见性