---
slug: custom-pod-autoscaler-on-k8s
title: Kubernetes Stack - 自定义容器自动缩放
authors: higan
tags: [ Kubernetes, Grafana, Prometheus, Micrometer, HAP ]
---

Kubernetes 本身拥有基于 CPU 与 Memory 的 HPA(Horizontal Pod Autoscaler) 功能，但是很多情况下 CPU 与 Memory
并不能完全表示一个业务服务的繁忙状态，所以很多时候使用 HTTP 请求数当作繁忙标准是一个更好的选择。

在本篇文章中，我们将会详细的了解如何将 [Kubernetes](https://kubernetes.io/)
与 [Prometheus](https://prometheus.io/)，[Grafana](https://grafana.com/) 整合，然后使用 Spring Boot
与 [Micrometer](https://micrometer.io) 创建自定义指标，并使用该指标定义容器自动缩放行为。

<!--truncate-->

# 在 Kubernetes 集群上部署监控系统

要使 Kubernetes 的 HPA 支持自定义指标，首先我们需要收集一个自定义指标。而 Prometheus 是一个用于收集应用指标的监控系统，通过
Prometheus 我们可以收集我们想要的指标。

Grafana 则可用于以可视化的方式呈现这些指标，以便我们更好的分析数据，并为制订自动缩放策略提供依据。

但是整合这一套监控系统是十分繁琐的，并且也容易出现意料之外的错误。非常幸运的是已经有人为我们提供了解决方案。那就是由 Red
Hat 下的 CoreOS Team 开源的 [prometheus-operator](https://github.com/coreos/prometheus-operator)。

我们需要先将 prometheus-operator 克隆到本地，在其 contrib/kube-prometheus 目录下就是将 Prometheus，Grafana 还有
prometheus-operator 整合部署到 Kubernetes 集群的基础示例工程，一般来说我们可以直接采用这个工程部署。

> 由于我们采用了 Aliyun 的 k8s 集群服务，而 Aliyun 并未提供 kubelet 的 https 接口，所以在 Aliyun 上部署这套系统需要修改
> Prometheus 用于监控 kubelet

指标所用的端口，详细修改请参考该提交 [2b3f62c](https://github.com/AliyunContainerService/prometheus-operator/commit/2b3f62c12fa34db63a9bceb19564288203a20eac)。

- 部署 prometheus-operator

下面的命令会在集群中创建一个 monitoring 命名空间，并部署 Prometheus，Grafana，prometheus-operator，prometheus-adapter 等服务。

由于配置间的依赖问题，这个命令可能会执行失败，可以多运行几次重试。另外推荐采用集群管理员权限部署，有些服务的部署需要获取 k8s
核心服务信息。

```shell
kubectl apply -f manifests/
```

部署完成之后可以观察 monitoring 命名空间下的各个服务是否正常启动，这通常需要一些时间。并且由于配置间的依赖问题，一些服务在初期启动时可能会报错，例如某
ConfigMap 无法找到之类的问题，可以查看出现的次数与时间。一般服务会自动重新部署，在 ConfigMap 创建之后这个问题会自动解决。

- 拆除 prometheus-operator

下面的命令会将 prometheus-operator 完全删除，只需要执行一次，一般不会报错。

```shell
kubectl delete -f manifests/
```

## 添加监听的命名空间

默认情况下，prometheus-operator 会监听所有的 ServiceMonitor 资源，但是这也取决于 prometheus-k8s ServiceAccount
的权限，在上面的部署中，prometheus-operator 会自动的添加 default，kube-system，monitoring
三个命名空间的访问权限，如果你在使用自己的命名空间，则需要修改一下配置文件。

在 prometheus-roleBindingSpecificNamespaces.yaml 中添加对目标命名空间的访问权限。

```yaml
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: prometheus-k8s
    namespace: <your-namespace>
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: prometheus-k8s
  subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring
```

在 prometheus-roleSpecificNamespaces.yaml 中添加对目标命名空间的访问权限。

```yaml
- apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: prometheus-k8s
    namespace: <your-namespace>
  rules:
  - apiGroups:
    - ""
    resources:
    - services
    - endpoints
    - pods
    verbs:
    - get
    - list
    - watch
```

修改了上面的配置之后，重新应用一下所有的配置。

```shell
kubectl apply -f manifests/
```

## ServiceMonitor 资源

部署上述服务后，会在集群中创建新的 ApiGroup，并创建几种自定义资源类型，ServiceMonitor 资源就是其中一种。ServiceMonitor 资源定义了
Prometheus
如何获取服务的状态，其结构定义可以参考文档 [ServiceMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#servicemonitor)。

prometheus-operator 会监控所有的 ServiceMonitor 资源，并更新到 Prometheus 中，实现了配置分离。当为服务提供了 ServiceMonitor
资源后，Prometheus 中就会自动的开始轮询服务的状态。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: prometheus
  name: prometheus
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: web
  selector:
    matchLabels:
      prometheus: k8s
```

上面的配置文件描述了一个对 Prometheus 服务的 ServiceMonitor，配置 endpoint 用于 Prometheus 的轮询，selector
用于指定条件。更多的配置可以参考上面的对于 [ServiceMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#servicemonitor)
的配置定义。

## 配置集群的 kube-controller-manager 的参数

为了让 Kubernetes 应用指标能够正常被 Prometheus 获取，我们需要额外修改一下 kube-controller-manager 的启动参数，而
kube-controller-manager 是一个 Static Pod，你需要手动登录到集群的每一个 Master 节点上，修改
/etc/kubernetes/manifests/kube-controller-manager.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --horizontal-pod-autoscaler-use-rest-clients=true
```

上面配置的最后一行就是我们需要修改的内容，我们需要将 horizontal-pod-autoscaler-use-rest-clients 设置为
true。保存之后，Kubernetes 会检查到文件的修改，自动的更新 kube-controller-manager，每一个 Master 节点都需要修改。

# 使用 Spring Boot 与 Micrometer 为服务提供监控

我们已经在集群中部署了 Prometheus 服务和其他的附带服务，接下来就是让我们的应用服务能被 Prometheus
发现并分析了。我们需要为我们的应用服务提供一个可以用于为 Prometheus 提供监控信息的 API。

[Micrometer](https://micrometer.io/) 曾经是 Spring Framework 的一部分，如今独立出来，并且可以通过 Spring Boot Actuator
整合在一起。Micrometer 抽象了各个不同的监控系统之间的差异，并提供了一些对于 Spring MVC 的支持，我们无需添加任何代码既可获取应用运行的指标信息。

在 Spring Boot (1.5.x)应用中添加以下依赖：

```kotlin
compile('org.springframework.boot:spring-boot-starter-actuator')
compile('io.micrometer:micrometer-spring-legacy:1.1.3')
compile('io.micrometer:micrometer-registry-prometheus:1.1.3')
```

在 Spring Boot (2.0)应用中，Micrometer 已经被当作 Spring 默认仪表盘库了，所以只需要添加以下依赖：

```kotlin
compile('org.springframework.boot:spring-boot-starter-actuator')
compile('io.micrometer:micrometer-registry-prometheus:1.1.3')
```

然后为了将 Spring Boot Actuator 提供的 API 与业务 API 区分，所以我们需要在应用的 application.yml 中设置一下端口，并且关闭验证以便
Prometheus 访问。

```yaml
management:
  security:
    enabled: false
  port: 8081
```

如果你采用 application.properties 配置应用，则采用下面的配置：

```properties
management.security.enabled=false
management.port=8081
```

重新启动应用，就能在 http://localhost:8081/prometheus 看到应用提供给 Prometheus 的统计信息了。

## 为服务提供 ServiceMonitor 资源

我们服务已经准备好提供数据给 Prometheus 了，接下来就是打通这一个部分。当我们创建好了我们的服务镜像 my-service:1.0 之后。创建相应的
Deployment 与 Service。

- my-service-deployment.yaml

```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
  namespace: playground
  name: my-service
  labels:
  app: my-service
  spec:
  replicas: 1
  selector:
  matchLabels:
  app: my-service
  template:
  metadata:
  labels:
  app: my-service
  spec:
  containers:
    - image: my-service:1.0
      name: my-service
      ports:
    - name: http
      containerPort: 8080
    - name: monitor
      containerPort: 8081
```

上面的配置在 playground 命名空间中添加了一个 Deployment 为 my-service，并开放了两个端口，一个 8080 命名为 http，用于提供业务
API，另一个 8081 命名为 monitor，用于 Prometheus 获取统计信息。

- my-service.yaml

```yaml
  apiVersion: v1
  kind: Service
  metadata:
  namespace: playground
  name: my-service
  labels:
  app: my-service
  spec:
  selector:
  app: my-service
  ports:
    - protocol: TCP
      name: http
      port: 8080
    - protocol: TCP
      name: monitor
      port: 8081
```

上面的配置在 playground 命名空间中添加了一个 Service 为 my-service，也开放了两个端口。

- my-service-monitor.yaml
```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
  name: my-service
  namespace: playground
  spec:
  endpoints:
    - interval: 15s
      port: monitor
      path: /prometheus
      selector:
      matchLabels:
      app: my-service
```

上面的配置在 playground 命名空间添加了一个 Prometheus 的 ServiceMonitor，并将其监听的对象设置为 app 标签为 my-service 的
Service。由于 Micrometer 提供的 Prometheus 的接口默认是在 /prometheus 下，所以我们也需要设置一下 ServiceMonitor 的
endpoint 的 path，如果不设置的话，会采用默认值 /metrics。

部署所有内容，Prometheus 应该就可以会监听到 my-service 了。可以通过 kubectl 的隧道功能，在本地访问 Prometheus 与 Grafana。

```shell
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
```

```shell
kubectl --namespace monitoring port-forward svc/grafana 3000
```

这两个命令分别在本地打开了端口 9090 用于访问集群上的 Prometheus，3000 端口用于访问 Grafana。

在 [http://localhost:9090/targets](http://localhost:9090/targets) 中我们应该就能看到服务的状态了，应该就能从列表中找到
my-service。

# 添加自动扩缩容策略

要通过 HPA 实现自动扩缩容，我们还需要一些其他的工作，例如提供 Custom Metrics API 用于基于自定义指标的自动扩缩容，还有为服务添加
HPA 策略之类的。

## 为我们的集群添加 Custom Metrics API 实现

在 Kubernetes 中有一些保留的 API 定义，但是并未实现，例如 Custom Metrics API，我们需要将 Prometheus 提供的指标转化为
Kubernetes 能够理解，HPA 能够使用的 Kubernetes Metrics。

而在上面的过程中，prometheus-operator 会自动部署 prometheus-adapter，而 prometheus-adapter 就是一种 Custom Metrics API
实现，所以我们只需要将 prometheus-adapter 注册为 Custom Metrics API 实现就可以了。

将目光转移到 prometheus-operator\contrib\kube-prometheus\experimental\custom-metrics-api 中，这个目录里面就有 Custom
Metrics API 相关配置。

```shell
kubectl apply -f custom-metrics-apiserver-resource-reader-cluster-role-binding.yaml
kubectl apply -f custom-metrics-apiservice.yaml
kubectl apply -f custom-metrics-cluster-role.yaml
kubectl apply -f custom-metrics-configmap.yaml
kubectl apply -f hpa-custom-metrics-cluster-role-binding.yaml
```

运行上述代码将 prometheus-adapter 注册为 Custom Metrics API 实现。值得注意的是，custom-metrics-apiservice.yaml 定义了
Custom Metrics API 实现为 monitoring 命名空间的 prometheus-adapter。

而 custom-metrics-configmap.yaml 则更新了adapter-config，由于只更新 ConfigMap 并不会更新 prometheus-adapter
服务，所以我们需要手动重新部署一下 prometheus-adapter。

完成这些内容之后，运行下面的指令：

```shell
kubectl api-versions
```

可以获得当前集群所支持的所有 API，其中需要有 custom.metrics.k8s.io/v1beta1。

```
admissionregistration.k8s.io/v1alpha1
admissionregistration.k8s.io/v1beta1 
alicloud.com/v1beta1                 
apiextensions.k8s.io/v1beta1         
apiregistration.k8s.io/v1            
apiregistration.k8s.io/v1beta1       
apps/v1                              
apps/v1beta1                         
apps/v1beta2                         
authentication.k8s.io/v1             
authentication.k8s.io/v1beta1        
authorization.k8s.io/v1              
authorization.k8s.io/v1beta1         
autoscaling/v1                       
autoscaling/v2beta1                  
batch/v1                             
batch/v1beta1                        
certificates.k8s.io/v1beta1          
crd.projectcalico.org/v1             
custom.metrics.k8s.io/v1beta1        
events.k8s.io/v1beta1                
extensions/v1beta1                   
log.alibabacloud.com/v1alpha1        
metrics.k8s.io/v1beta1               
monitoring.coreos.com/v1             
networking.k8s.io/v1                 
policy/v1beta1                       
rbac.authorization.k8s.io/v1         
rbac.authorization.k8s.io/v1beta1    
scheduling.k8s.io/v1beta1            
storage.k8s.io/v1                    
storage.k8s.io/v1beta1               
v1    
```

通过下面的指令能够获取到当前集群的所有 Custom Metrics：

    kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"

如果返回了 No Resource，请检查 prometheus-adapter 是否正确的重启了，并且处于正常状态，kube-controller-manager
参数是否设置正确了，Prometheus 相关服务是否正常工作。

## 为 Deployment 提供 HPA

完成上面的所有工作之后，终于可以配置服务的 HPA 了，并且可以在 HPA 中使用自定义 Metrics。

```yaml
apiVersion: autoscaling/v2alpha1
kind: HorizontalPodAutoscaler
metadata:
name: gateway
spec:
  scaleTargetRef:
    kind: Deployment
    name: my-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Object
  object:
    target:
    kind: Service
    name: my-service
    metricName: <custom-metric>
    targetValue: <target-value>
```

上面的配置就是一个 HPA 的描述，将 Deployment 的 my-service 的缩放绑定到 Service 的 my-service 的自定义 Metric 上。

targetValue 可以填写数字和单位，m 代表毫，500m 为 0.5 的意思，k 代表千，1k 就是 1000，而 M 代表兆，为 1000000。

## 为 Custom Metrics 提供约定

我们回过头看 custom-metrics-configmap.yaml 在这个文件中提供了如何将 Kubernetes Metrics 转换为 PromQL(Prometheus
的查询语句)，理论上讲 HPA 中使用的 Kubernetes Metrics 和 Prometheus 中统计的 Metrics 可以完全不相干，而 prometheus-adapter
提供了这种绑定关系，他们通过 PromQL 绑定在一起。

```yaml
- seriesQuery: '{namespace!="",__name__!~"^container_.*"}'
  seriesFilters:
    - isNot: .*_seconds_total
  resources:
    template: <<.Resource>>
  name:
    matches: ^(.*)_total$
    as: ""
  metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
```

在这一段说明了，如果 Prometheus 提供了 http_requests_total 这个 Metric，将会如何转换为 http_requests 的 Kubernetes
Metrics。

- seriesQuery 指定了在查询 Prometheus 时的 series 条件，seriesFilters 用来详细判断名字是否满足某个正则表达式。
- name 则表示如何将 Prometheus 的 Metrics 名字转化为 Kubernetes Metrics 名字。as 为空的情况下，会自动的采用 $1 捕获组当作
  Kubernetes Metrics 名字。
- metricsQuery 则表示了最后生成的 PromQL 的模板。

具体更多自定的配置内容可以参考 [Config of prometheus-adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config.md)。

---

到这里，本篇文章就结束了，文章挺长的。我也花了接近 2 天多的时间实现了上面的内容，中途也遇到了各种无奈的坑，在这一块网上的文档基本都是千篇一律的，东补一点西补一点，挺不容易的。特别是上面对于
kube-controller-manager 参数的配置，和注册了 Custom Metrics API 时需要重启
prometheus-adapter，都是遇到的坑，希望这篇文章能够为后人指路，不用继续踩坑了。
