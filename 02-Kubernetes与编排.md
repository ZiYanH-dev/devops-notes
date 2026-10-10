# Kubernetes（K8s）与编排

> **解决什么问题**：
> 	单个容器易于管理，
> 	**成百上千个**容器需要自动部署、扩容与容灾。

> **一句话**：
> 	K8s 是容器编排系统，
> 	自动完成一组==容器的调度、扩缩容与自愈。==

## 0. K8s 的软件类别与形态

**K8s 是基础设施（infrastructure）软件，
不是编程语言的库，也不是框架。**

- 它是多个组件组成的独立程序，运行在服务器集群上，控制整个集群的机器。
- 它不能 `import` 进业务代码，只能**部署后向其下发指令**。

| 对比   | 语言库/框架               | Kubernetes            |
| ---- | -------------------- | --------------------- |
| 形态   | 代码，`import`/`new` 进来 | 一个独立运行的服务集群           |
| 你做的事 | 用它的 API 写业务          | 部署并向其下发指令             |
| 归类   | Express、Django 这类库   | 操作系统、数据库、Nginx 这类基础设施 |

### 三种交互方式（全走 API Server）

K8s 交互总结：
- 所有操作的唯一入口是 kube-apiserver，
- 底层统一走 HTTPS REST API。

1. kubectl：命令行客户端，封装 HTTP 请求。
2. 各语言 SDK：编程库，封装 HTTP 请求。
3. 原生 HTTP 请求：curl 等工具直接调用 API。三者只是封装层不同，本质都是向 apiserver 发网络请求。集群内部组件（kubelet、控制器）同样通过这套 API 和集群交互。

无论哪种方式，最终发送给 K8s 的都是同一份内容，即一套资源描述。

```
kubectl CLI  ──┐
语言库(客户端) ──┼→ 发 HTTP 请求 → K8s API Server
YAML 文件    ──┘
```

| 方式               | 说明                              |
| ---------------- | ------------------------------- |
| **CLI（kubectl）** | 最常用，`kubectl apply -f xxx.yaml` |
| **配置文件（YAML）**   | 声明式描述"我想要什么状态"，主流方式             |
| **语言库发请求**       | 通过 HTTP 调 API Server，语言库只是封装为函数 |

> SDK/语言库不是连接 K8s 的唯一途径，CLI 和 YAML 更主流。

### 核心职责：调度与容器生命周期管理

调度是核心，但不止于此，它负责一大批容器的完整生命周期。

| K8s 做的事         | 说明              |
| --------------- | --------------- |
| **调度**          | 分配容器至集群内的具体机器   |
| **自动扩缩容**       | 流量增大时扩容，缩小时缩减   |
| **自愈**          | 容器故障时自动重建       |
| **服务发现 + 负载均衡** | 客户端找到并分发到这些容器   |
| **滚动更新 / 回滚**   | 更新不停机，异常时回滚至旧版本 |
| **配置/密钥管理**     | 注入配置和密钥         |

> **一句话**：
> 	- K8s 管理一大批容器的完整生命周期，包括放置、扩缩容、故障重启、更新回滚，全部自动。
> 	- 它调度的是容器运行时（Docker/containerd）起的容器，但"调度"只是它能力的第一层。

## 1. K8s 相对单机 Docker 的必要性

| 问题    | Docker 单机 | K8s 集群            |
| ----- | --------- | ----------------- |
| 容器故障  | 需要人手动重启   | **自动重建新容器**（自愈）   |
| 流量增大  | 手动加容器     | **自动扩缩容（HPA）**    |
| 多台服务器 | 每台各自管     | **统一调度**到任意节点     |
| 服务发现  | 手动记录 IP   | **自动注册 + DNS 解析** |
| 滚动发布  | 流程繁琐      | **自动滚动更新/回滚**     |

> Docker 管"单台机器上的容器"，
> K8s 管"一群机器上一大群容器"，是不同量级。

## 2. 核心架构：Master（主控）+ Worker（工作节点）

```
┌─ Control Plane（主控，不运行业务）────────────┐
│  kube-apiserver：所有操作的统一入口（API）              │
│  etcd：集群状态数据库，存储期望状态           │
│  kube-scheduler：把 Pod 分到哪台机器               │
│  kube-controller-manager：不断对账"实际→期望"      │
└──────────────────────────────────────────────┘
                   ↓ 控制
┌─ Worker Node（工作节点，运行业务）───────────────┐
│  kubelet：本机的 K8s 代理，执行命令                 │
│  kube-proxy：本机网络转发规则                      │
│  容器运行时（containerd / Docker）               │
│  [Pod] [Pod] [Pod]  ← 业务运行于此                 │
└──────────────────────────────────────────────┘
```

| 组件                     | 归属   | 职责               |
| ---------------------- | ---- | ---------------- |
| **kube-apiserver**     | 主控   | 所有请求入口，集群的中枢     |
| **etcd**               | 主控   | 存集群期望状态（键值数据库）|
| **kube-scheduler**     | 主控   | 决定 Pod 放哪个节点     |
| **controller-manager** | 主控   | 对账循环：让实际状态逼近期望状态 |
| **kubelet**            | 工作节点 | 本机代表，创建/管理 Pod   |
| **kube-proxy**         | 工作节点 | 本机网络转发           |

> 核心思想：**声明式对账**。
> 向 K8s 声明该有 3 个副本，controller 持续检查实际状态，副本不足时自动补充，始终维持实际状态等于期望状态。

## 3. 核心对象（Pod 是基本单位）

| 对象                     | 是什么                          | 类比          |
| ---------------------- | ---------------------------- | ----------- |
| **Pod**                | K8s 调度的最小单位，1 个容器 + 共享网络/存储  | 最小执行单元      |
| **Deployment**         | 管一组 Pod 的副本数 + 滚动更新          | 负责"保证有 N 个" |
| **Service**            | 稳定的访问入口，给一组 Pod 一个固定 IP/DNS  | 负载均衡器       |
| **Ingress**            | 外部 HTTP 入口，按域名/路径路由到 Service | 网关/路由       |
| **ConfigMap / Secret** | 配置和密钥，与镜像解耦                  | 外部配置        |
| **Volume**             | 存储，Pod 删了数据还在                | 持久盘         |
| **Namespace**          | 逻辑隔离的一组资源                    | 分区          |

### 数据流全景

```
用户/外部请求
    ↓
Ingress（域名 → Service）
    ↓
Service（固定 IP + 负载均衡）
    ↓ 分发给多个副本
[Pod 1] [Pod 2] [Pod 3]   ← Deployment 保证一直有 3 个
   ↕ 读写
Volume（持久数据）
```

## 4. 声明式 YAML 示例（Deployment + Service）

```yaml
apiVersion: apps/v1
kind: Deployment                      # 管副本
metadata:
  name: my-api
spec:
  replicas: 3                         # 期望 3 个副本
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: myregistry/api:v1
          ports: [{ containerPort: 8000 }]
---
apiVersion: v1
kind: Service                        # 稳定访问入口
metadata:
  name: api-svc
spec:
  selector: { app: api }             # 挑哪些 Pod
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP                    # 集群内固定 IP
```

```bash
kubectl apply -f deploy.yaml   # 声明式应用
kubectl get pods / deploy / svc    # 查看状态
kubectl scale deploy my-api --replicas=5   # 手动扩容
```

## 5. 常用操作速查

| 命令                               | 作用       |
| -------------------------------- | -------- |
| `kubectl get nodes / pods / svc` | 查看资源状态   |
| `kubectl apply -f x.yaml`        | 应用声明式配置  |
| `kubectl delete -f x.yaml`       | 删除资源     |
| `kubectl logs <pod>`             | 看 Pod 日志 |
| `kubectl exec -it <pod> -- bash` | 进容器      |
| `kubectl scale ... --replicas=N` | 手动扩缩容    |

## 6. 高频一句话

- **K8s = 容器编排**：自动调度、扩缩容、自愈一群容器
- **声明式对账**：声明期望状态，controller 持续将实际状态调整为期望状态
- **Pod = 最小调度单位，Deployment = 副本管理，Service = 稳定入口，Ingress = 外部路由**
- **Master 负责决策，Worker 运行业务**，etcd 存储期望状态
- 单容器用 Docker，**容器规模扩大后才需要 K8s**