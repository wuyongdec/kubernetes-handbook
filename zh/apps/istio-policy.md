# Istio 策略管理

Mixer 为应用程序和基础架构后端之间提供了一个通用的策略控制层，负责先决条件检查（如认证授权）、配额管理并从 Envoy 代理中收集遥测数据等。

![](images/istio-mixer.png)

Mixer 支持灵活的插件模型（即 Adapters），支持 GCP、AWS、Prometheus、Heapster 等各种丰富功能的后端。

![](images/istio-adapters.png)


## 实现原理

本质上，Mixer 是一个 [属性](https://istio.io/docs/concepts/policy-and-control/attributes.html) 处理机，进入 Mixer 的请求带有一系列的属性，Mixer 按照不同的处理阶段处理：

- 通过全局 Adapters 为请求引入新的属性
- 通过解析（Resolution）识别要用于处理请求的配置资源
- 处理属性，生成 Adapter 参数
- 分发请求到各个 Adapters 后端处理

![](images/istio-phase.png)

Adapters 后端以 [Mixer 配置](https://istio.io/docs/tasks/policy-enforcement/rate-limiting/) 的方式注册到 Istio 中，参考 [这里](https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/kube/mixer-rule-ratings-ratelimit.yaml) 查看示例配置。

## 流量限制示例

```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: memquota
metadata:
  name: handler
  namespace: istio-system
spec:
  quotas:
  - name: requestcount.quota.istio-system
    maxAmount: 5000
    validDuration: 1s
    # The first matching override is applied.
    # A requestcount instance is checked against override dimensions.
    overrides:
    # The following override applies to 'ratings' when
    # the source is 'reviews'.
    - dimensions:
        destination: ratings
        source: reviews
      maxAmount: 1
      validDuration: 1s
    # The following override applies to 'ratings' regardless
    # of the source.
    - dimensions:
        destination: ratings
      maxAmount: 100
      validDuration: 1s

---
apiVersion: "config.istio.io/v1alpha2"
kind: quota
metadata:
  name: requestcount
  namespace: istio-system
spec:
  dimensions:
    source: source.labels["app"] | source.service | "unknown"
    sourceVersion: source.labels["version"] | "unknown"
    destination: destination.labels["app"] | destination.service | "unknown"
    destinationVersion: destination.labels["version"] | "unknown"

---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: quota
  namespace: istio-system
spec:
  actions:
  - handler: handler.memquota
    instances:
    - requestcount.quota
---
apiVersion: config.istio.io/v1alpha2
kind: QuotaSpec
metadata:
  creationTimestamp: null
  name: request-count
  namespace: istio-system
spec:
  rules:
  - quotas:
    - charge: 1
      quota: RequestCount
---
apiVersion: config.istio.io/v1alpha2
kind: QuotaSpecBinding
metadata:
  creationTimestamp: null
  name: request-count
  namespace: istio-system
spec:
  quotaSpecs:
  - name: request-count
    namespace: istio-system
  services:
  - name: ratings
  - name: reviews
  - name: details
  - name: productpage
```

## 参考文档

- [Istio Mixer](https://istio.io/docs/concepts/policy-and-control/mixer.html)
