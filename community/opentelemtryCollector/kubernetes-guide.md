![](https://github.com/open-telemetry/opentelemetry-operator/workflows/Continuous%20Integration/badge.svg#crop=0&crop=0&crop=1&crop=1&id=wagEX&originHeight=20&originWidth=205&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=) ![](https://goreportcard.com/badge/github.com/open-telemetry/opentelemetry-operator#crop=0&crop=0&crop=1&crop=1&id=PXOzn&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=) ![](https://godoc.org/github.com/open-telemetry/opentelemetry-operator?status.svg#crop=0&crop=0&crop=1&crop=1&id=JDNkS&originHeight=20&originWidth=90&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />

<a name="15b05141"></a>
# OpenTelemetry for Kubernetes 指南
首先，你需要在Kubernetes 安装 OpenTelemetry Operator<br />OpenTelemetry Operator：OpenTelemetry 为了在Kubernetes更好管理Collector，Agent，一种 Kubernetes Operator的具体实现
<a name="tUfL3"></a>
### Kubernetes Operator 
Operator 是 Kubernetes 的扩展软件，它利用 [定制资源](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 管理应用及其组件。 <br />
<br />OpenTelemetry  Operator 管理的组件:

- [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector)
- 使用 OpenTelemetry 自动构件库完成自动构建 



<a name="Documentation"></a>
## 文档
- [API 文档](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api.md)
## 开始
安装Operator前, 先确保安装好Kubernetes`[cert-manager](https://cert-manager.io/docs/installation/)`，安装运行教程参考[ ](https://cert-manager.io/docs/installation/)[install cert-manager ](https://cert-manager.io/docs/installation/) :
```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```
`opentelemetry-operator` deployment 完成后, 对于的Kubernetes 自定义资源会看到：

![image.png](/assets/kubernetes-crd.png)<br />官方文档给了一个创建 OpenTelemetry Collector (otelcol) 实例, 像这样:<br />

```yaml
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: simplest
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:

    exporters:
      logging:

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [logging]
EOF
```

<br />更多细节参考[官方文档](https://github.com/open-telemetry/opentelemetry-operator#getting-started) ，我们这里重点讲部署实践<br />

<a name="05cb21b1"></a>
## 部署

<br />Kubernete 自定义资源为 `OpenTelemetryCollector` 暴露了一个参数 `.Spec.Mode`, 可以通过它配置我们以何种方式启动Collector，目前支持 `DaemonSet`, `Sidecar`, or `Deployment` (缺省). <br />

<a name="3efc1608"></a>
### Sidecar 注入
一个OpenTelemetry Collector的sidecar 可以通过设置pod的声明`sidecar.opentelemetry.io/inject` 是为 `"true"`来注入到pod中。或者如果在同一个命名空间，还可以用`OpenTelemetryCollector`具体名字来注入，像下面的例子<br />

```yaml
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: sidecar-for-my-app
spec:
  mode: sidecar
  config: |
    receivers:
      jaeger:
        protocols:
          grpc:
      otlp:
        protocols:
          grpc:
          http:
    processors:

    exporters:
      logging:

    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger]
          processors: []
          exporters: [logging]
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    sidecar.opentelemetry.io/inject: "true"
spec:
  containers:
  - name: myapp
    image: jaegertracing/vertx-create-span:operator-e2e-tests
    ports:
      - containerPort: 8080
        protocol: TCP
EOF
```
在Kubernete部署后，从Kubernete看到Opentelemetry sidecar 注入到应用程序中，采集到数据

![image.png](/assets/kubernetes-collector-2.jpeg) <br />

当多个 `OpenTelemetryCollector`资源在同一命名空间设置`Sidecar` 模式时，最好使用具体命名的方式注入。在同一个命名空间通过声明方式注入，只会有一个`Sidecar` 实例运行。<br />声明值可以在pod，namespace配置，越细粒度声明，它的优先级越高：<br />

- pod中声明先生效：pod指定了具体命名Collector，或者设置`sidecar.opentelemetry.io/inject` 为`"false"`
- namespace 中声明生效： namespace 设置了一个具体的Collector，而且pod 没有声明或者`sidecar.opentelemetry.io/inject` 为`"true"`


<br />当我们管理Pod工作资源，采用`Deployment` or `Statefulset`模式时，一定要在`PodTemplate` <br />部分添加声明，下面有正确、错误两种示例：
```yaml
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    sidecar.opentelemetry.io/inject: "true" # WRONG
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        sidecar.opentelemetry.io/inject: "true" # CORRECT
    spec:
      containers:
      - name: myapp
        image: jaegertracing/vertx-create-span:operator-e2e-tests
        ports:
          - containerPort: 8080
            protocol: TCP
EOF
```
<a name="7d5fca9b"></a>
### 
<a name="c40a9ed4"></a>
## 兼容矩阵
目前官方给出Opentelemetry 在Kubernetes兼容版本对照图
<a name="41c85136"></a>
### OpenTelemetry Operator vs. Kubernetes vs. Cert Manager
| OpenTelemetry Operator | Kubernetes | Cert-Manager |
| --- | --- | --- |
| v0.42.0 | v1.21 to v1.23 | 1.6.1 |
| v0.41.1 | v1.21 to v1.23 | 1.6.1 |
| v0.41.0 | v1.20 to v1.22 | 1.6.1 |
| v0.40.0 | v1.20 to v1.22 | 1.6.1 |
| v0.39.0 | v1.20 to v1.22 | 1.6.1 |
| v0.38.0 | v1.20 to v1.22 | 1.6.1 |
| v0.37.1 | v1.20 to v1.22 | v1.4.0 to v1.6.1 |
| v0.37.0 | v1.20 to v1.22 | v1.4.0 to v1.5.4 |
| v0.36.0 | v1.20 to v1.22 | v1.4.0 to v1.5.4 |
| v0.35.0 | v1.20 to v1.22 | v1.4.0 to v1.5.4 |
| v0.34.0 | v1.20 to v1.22 | v1.4.0 to v1.5.4 |
| v0.33.0 | v1.20 to v1.22 | v1.4.0 to v1.5.4 |
| v0.32.0 (skipped) | n/a | n/a |
| v0.31.0 | v1.19 to v1.21 | v1.4.0 to v1.5.4 |
| v0.30.0 | v1.19 to v1.21 | v1.4.0 to v1.5.4 |
| v0.29.0 | v1.19 to v1.21 | v1.4.0 to v1.5.4 |
| v0.28.0 | v1.19 to v1.21 | v1.4.0 to v1.5.4 |
| v0.27.0 | v1.19 to v1.21 | v1.4.0 to v1.5.4 |


