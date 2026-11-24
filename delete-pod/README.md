# 删除 Pod 时发生了什么

本文以 Linux 节点、普通优雅删除、带 selector 的 Service 和 kube-proxy 的 iptables 模式为例。具体行为还受 Kubernetes 版本、网络插件和 Ingress controller 的实现影响。

## 总览

删除 Pod 不是一个严格串行的「先摘流量，再关闭容器」过程。API server 标记删除后，控制面和节点上的工作会并行、异步地发生：

```text
用户 / 工作负载 controller
        |
        | DELETE Pod
        v
API server 给 Pod 设置 deletionTimestamp 和删除宽限期
        |
        +--> EndpointSlice controller 更新 endpoint 状态
        |       +--> 各节点 kube-proxy 更新 Service 转发规则
        |       +--> Ingress controller 可能更新自己的后端
        |
        +--> Pod 所在节点的 kubelet 开始关闭容器
        |       preStop -> 停止信号（通常是 SIGTERM）-> 超时后 SIGKILL
        |
        +--> 工作负载 controller 按期望副本数协调，可能创建替代 Pod
```

图中分支没有先后完成的保证：容器可能在所有流量入口完成同步之前就收到停止信号甚至退出。

## 1. 发起删除并标记 Pod

用户可以执行 `kubectl delete pod <pod-name>`；Deployment 滚动更新、缩容等操作也可能使工作负载 controller 发起 Pod 删除请求。发起请求的并不一定是 controller。API server 接受普通优雅删除后，更新 Pod 的 `metadata.deletionTimestamp` 和删除宽限期，通知 watch 该对象的组件。

`kubectl get pod` 中常见的 `Terminating` 是展示状态，不是 Pod 的 `status.phase`。默认宽限期由 `spec.terminationGracePeriodSeconds` 决定，默认值为 30 秒，删除请求也可以指定宽限期。

如果这个 Pod 由 Deployment/ReplicaSet 管理且期望副本数未降低，ReplicaSet 通常会创建一个新的替代 Pod；新 Pod 有不同的 UID，也不一定调度到同一个节点。创建和就绪都需要时间，不保证旧 Pod 退出前替代 Pod 已就绪。缩容时则不一定需要替代 Pod。

## 2. EndpointSlice、Service 和 Ingress 的变化

对于带 selector 的 Service，EndpointSlice controller 根据 Service 和匹配的 Pod 维护 EndpointSlice。当它观察到 Pod 正在删除，会更新对应 endpoint 的 conditions；endpoint 不一定立刻从 EndpointSlice 中消失：

| condition | 含义 |
| --- | --- |
| `terminating` | 对应 Pod 正在删除，通常在容器退出前就会变为 `true`。 |
| `serving` | endpoint 是否仍在提供服务；对于 Pod endpoint，对应 Pod 的 Ready 状态。 |
| `ready` | 通常相当于 `serving && !terminating`，因此删除中的 endpoint 通常为 `false`。 |

一般情况下，消费者不会把 `ready: false` 的 endpoint 用于普通新流量。例外包括：Service 设置 `publishNotReadyAddresses: true` 时，`ready` 的语义不同；当可用 endpoint 都在终止时，Service proxy 在某些情况下可能仍向 `serving: true` 且 `terminating: true` 的 endpoint 转发流量。最终 endpoint 会在不再需要时从 EndpointSlice 中移除。无 selector 的 Service 可能由其他组件或用户自行管理 EndpointSlice，不能假设内置 controller 会按上述方式维护。

**Service 和 Ingress 对象本身通常不因为某个后端 Pod 被删除而修改。** Service 保留其 ClusterIP、端口和 selector；变化的是 EndpointSlice 中的后端状态。Ingress 定义的是路由规则，需要 Ingress controller 实现：

- 有些 Ingress controller 观察 Service 和 EndpointSlice（或其他后端数据），更新自身的 upstream，直接向 Pod IP 转发；此时后端选择不一定经过 Service ClusterIP 或 kube-proxy。
- 有些实现或配置通过 Service ClusterIP 转发，则 Service 的数据平面负责选择后端。

具体监听哪些资源、何时更新 upstream、如何处理已有连接，取决于所用的 Ingress controller 及其配置。Ingress controller、EndpointSlice controller 和 kube-proxy 之间没有全局同步屏障。

## 3. kube-proxy 更新节点转发规则

kube-proxy 在各节点监听 Service 和 EndpointSlice 的变化，根据本地观察到的状态协调数据平面。在 Linux iptables 模式下，简化的路径是：

```text
Service ClusterIP:port
    -> Service 对应的 iptables 规则
    -> 选出可用 endpoint
    -> DNAT 到 Pod IP:targetPort
```

观察到 EndpointSlice 变化后，各节点会各自更新相关规则，通常使被删除 Pod 不再被选为普通新连接的后端；规则不会在所有节点同时生效。传播、队列和规则同步都可能产生延迟。`internalTrafficPolicy`、`externalTrafficPolicy`、会话亲和性以及上述 terminating endpoint 的例外等也会影响实际选择。

iptables 更新主要影响**新连接**的后端选择；已有连接可能继续使用 conntrack 中的 NAT 映射，因此不会因为删掉一条新连接选择规则就自动平滑结束。连接最终是否成功还取决于应用、代理和连接超时等。

这只是 iptables 模式的例子：kube-proxy 也可使用 nftables 或 IPVS 等模式；部分集群使用其他 Service 数据平面替代 kube-proxy。应以实际集群配置为准。

## 4. kubelet 关闭容器

Pod 所在节点的 kubelet 观察到删除标记后开始本地终止，和上述网络同步并行：

1. 宽限期开始计时。如果容器配置了 `preStop` 且宽限期非零，kubelet 先执行该 hook。hook 的耗时**包含在**整个宽限期内，不是在宽限期之外额外等待；如果 hook 到期时仍在运行，kubelet 可请求一次短暂的两秒延长。
2. hook 完成后，kubelet 请求容器运行时停止容器。运行时通常向容器内的主进程（PID 1）发送 `SIGTERM`；镜像的 `STOPSIGNAL` 或受版本和功能开关控制的容器停止信号配置可能改变实际信号。应用应处理停止信号，停止接收新工作并完成或安全中止进行中的工作。
3. 宽限期到期时，如果仍有进程运行，kubelet 触发容器运行时发送 `SIGKILL`，强制结束剩余进程。`SIGKILL` 不能被应用捕获或处理。
4. 容器退出后，kubelet 清理 Pod 资源并向 API server 报告状态，随后 Pod API 对象被删除。finalizer、节点失联等情况可能使删除不能按正常路径及时完成。

普通多容器 Pod 的容器停止顺序没有保证；Kubernetes 原生 sidecar 容器有特殊顺序：主容器退出后再按定义的逆序终止 sidecar。若宽限期耗尽，剩余容器仍可能被强制终止。

## 5. 如何降低请求中断

- 应用正确处理停止信号，支持停止接收新工作、等待正在处理的请求，并为长连接制定排空策略。确保 PID 1 能接收和转发信号。
- 为传播延迟和应用清理留出足够的 `terminationGracePeriodSeconds`。必要时使用 `preStop` 等待一段时间，让网络组件有机会摘流量，但固定 sleep 不是同步完成的保证，也会占用宽限期。
- 保持足够的健康副本，并合理设置就绪探针、滚动更新策略；客户端和代理也应适当处理超时与重试。避免将重试用于不安全的非幂等操作，除非应用有去重机制。
- 直接访问 Pod IP、已有长连接、外部负载均衡器以及不同 Ingress 实现都可能绕过或延迟上述摘流量过程，需要分别验证。

## 6. 观察过程

下面以 namespace `default`、Service `my-service` 和 Pod `my-pod` 为例；请替换为实际名称。先在不同终端启动观察，再从另一个终端删除 Pod。EndpointSlice 名称不固定，因此按 Service 标签查询：

```bash
kubectl -n default get pod my-pod -w
kubectl -n default get endpointslice -l kubernetes.io/service-name=my-service -o yaml -w
kubectl -n default get events --sort-by=.metadata.creationTimestamp
```

```bash
kubectl -n default delete pod my-pod --wait=false
```

在 EndpointSlice 输出中查找该 Pod 对应的 `targetRef`、地址和 `conditions`；变化可能很快，且 watch 输出不会保证捕获每个中间状态。使用 `kubectl describe pod my-pod` 可在 Pod 对象仍存在时查看事件。若需要检查实际 iptables 规则，应在运行 kube-proxy 的相应 Linux 节点上使用有权限的账户，例如 `sudo iptables-save -t nat`；链名和规则内容取决于版本及集群配置。

不要把 `kubectl delete pod --force --grace-period=0` 当作普通优雅删除：API server 不等待 kubelet 确认容器已停止，旧进程可能仍在节点上运行。

## 参考资料

- [Termination of Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-termination/)
- [EndpointSlices：Conditions](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/#conditions)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
