## Istio in Action

> book address : https://livebook.manning.com/book/istio-in-action/welcome/v-16/ 

### Understanding Istio

#### Introducing Istio Service Mesh

- Our cloud infrastructure is not reliable

- Making service interaction resilient（使服务之间有弹性）

- The application-aware service proxy（应用感知服务代理）：将这些横向关注点(horizontal concerns)转移到基础架构中的一种方法是使用代理。代理是一个中间基础设施组件，可以处理连接并将它们重定向到适当的后端。

 Use a proxy to push these horizontal concerns such as resilience, traffic control, security, etc. out of the application  implementation（使用代理将这些横向问题（例如弹性、流量控制、安全性等）排除在应用程序实施之外）

![image-20220324144028482](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220324144028482.png)

我们想要的是一个“应用程序感知”的代理，并且能够代表我们的服务执行应用程序网络。要做到这一点，这个“服务代理”需要理解消息和请求等应用程序结构，而不是理解连接和数据包的更传统的基础设施代理。换句话说，我们需要一个第 7 层代理。

- ### Meet Envoy proxy（认识envoy代理）

Envoy 是 Lyft 开发的，作为其面向服务的架构基础设施的一部分，能够在应用程序之外实现应用程序弹性和其他网络问题。 Envoy 为我们提供了重试、超时、断路、客户端负载平衡、服务发现、安全性和指标收集等网络功能，而无需任何明确的语言或框架依赖。

![image-20220324144638199](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220324144638199.png)

Envoy 的强大功能不仅限于这些应用层弹性方面（application layer resilience aspects）。 Envoy 还捕获许多应用程序网络指标，例如每秒请求数、故障数、断路事件等。通过使用 Envoy，我们可以自动了解我们的服务之间发生的事情，这就是我们开始看到很多意想不到的复杂性的地方。 Envoy 代理为解决服务架构的横切、横向可靠性和可观察性问题奠定了基础，并允许我们将这些问题推到应用程序之外并进入基础设施。

服务代理还可以做一些事情，比如收集分布式跟踪跨度，这样我们就可以将特定请求所采取的所有步骤拼接在一起。我们可以看到每个步骤花费了多长时间，并在我们的系统中寻找潜在的瓶颈或错误。如果所有应用程序都通过自己的代理与外部世界通信，并且应用程序的所有传入流量都通过我们的代理，那么我们已经为我们的应用程序获得了一些重要的功能，而无需更改任何应用程序代码。这种代理+应用程序组合构成了称为服务网格的通信总线的基础。

我们可以将像 Envoy 这样的服务代理部署在我们应用程序的每个实例中作为单个原子单元。例如，在 Kubernetes 中，我们可以将服务代理与我们的应用程序共同部署在单个 Pod 中。这种部署模式称为边车部署（**sidecar** ）。

![image-20220324150023187](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220324150023187.png)

##### What’s a service mesh?

A *service mesh* is a distributed application infrastructure that is responsible for handling network traffic on behalf of the application in a transparent, out of process manner.（服务网格是一种分布式应用程序基础设施，负责以透明、进程外的方式代表应用程序处理网络流量。）

##### Introducing Istio service mesh

Istio 是由 Google、IBM 和 Lyft 创建的服务网格的开源实现。 Istio 可帮助您以透明的方式为您的服务架构添加弹性和可观察性。使用 Istio，应用程序不必知道它们是服务网格的一部分。每当他们与外界交互时，Istio 将代表应用程序处理网络。这意味着无论您是在做微服务、单体还是介于两者之间的任何东西，Istio 都可以带来很多好处。 Istio 的数据平面默认使用开箱即用的 Envoy 代理，并帮助您配置应用程序以在其旁边部署服务代理 (Envoy) 的实例。 Istio 的控制平面由几个组件组成，这些组件为最终用户/operators提供 API、代理配置 API、安全设置、策略声明等。

![image-20220324151647937](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220324151647937.png)

1. Traffic comes into the cluster from a client outside the mesh through the Istio ingress gateway
2. Traffic goes to the shopping-cart service. The traffic first passes through the shopping-cart service’s sidecar proxy. The service proxy can apply timeouts, metric collection, security enforcement etc for the service.
3. As the request makes its way through various services, Istio’s service proxy can intercept the request at various steps and make routing decisions (e.g., to route some requests intended to the Tax service to v1.1 of the Tax service which may have a fix for certain tax calculations)
4. Istio’s control plane (`istiod`) is used to configure the Istio proxies which handle routing, security, telemetry collection, and resilience
5. Request metrics are periodically sent back to various collection services; Distributed tracing spans (like Jaeger or Zipkin) are sent back to an tracing store which can be used to later track the path and latency of a request through the system

##### What are the drawbacks to using a service mesh?

首先，使用服务网格在请求路径中放置另一个中间件，特别是代理。这个代理能够提供很多价值，但对于那些不熟悉代理的人来说，这最终可能会成为一个黑匣子，并且使调试应用程序的行为变得更加困难。 Envoy 代理是专门为可调试而构建的，并且会公开很多关于网络上发生的事情的信息——比不存在时更重要——但对于不熟悉操作 Envoy 的人来说，这可能看起来非常复杂并抑制现有的调试实践。

使用服务网格的另一个缺点是在租赁方面。网格与网格中运行的服务一样有价值。也就是说，网格中的服务越多，网格对于运营这些服务就越有价值。但是，如果在物理网格部署的租赁和隔离模型中没有适当的策略、自动化和深思熟虑，您最终可能会遇到错误配置网格会影响许多服务的情况。

最后，服务网格成为您的服务和应用程序架构的根本重要部分，因为它位于请求路径上。服务网格可以提供很多机会来提高安全性、可观察性和路由控制状态。缺点是网格引入了另一个层和另一个复杂性的机会。了解如何在现有的组织流程和治理中以及在现有团队之间进行配置、操作和最重要的集成是很困难的。

#### First steps with Istio

##### Getting the Istio distribution

1.下载istio

```
https://github.com/istio/istio/releases/tag/1.13.0
```

2.查看版本

```shell
[root@biz-master-48 bin]# istioctl version
no running Istio pods in "istio-system"
1.13.0
```

3.precheck：安装istio之前的检验

```shell
[root@biz-master-48 bin]# istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
  To get started, check out https://istio.io/latest/docs/setup/getting-started/
```

4.demo版本istio安装

```shell
istioctl install --set profile=demo -y
```

```shell
[root@biz-master-48 istio-1.13.0]# istioctl install --set profile=demo -y
✔ Istio core installed                                                                                                   
✔ Istiod installed                                                                                                       
✔ Egress gateways installed                                                                                              
✔ Ingress gateways installed                                                                                             
✔ Installation complete                                                                                                  Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/pzWZpAvMVBecaQ9h9
```

service mesh由date-plane（that is service proxies）和control-plane组成。当我们安装完instio后，你可以看到一个control plane、ingress、egress。同样当我们安装application并注入service proxies后，我们就拥有了一个data plane。

```shell
[root@biz-master-48 istio-1.13.0]# kubectl get po -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-egressgateway-6cf5fb4756-9vrzq   1/1     Running   0          117s
istio-ingressgateway-dc9c8f588-bd6bj   1/1     Running   0          117s
istiod-7586c7dfd8-gffc6                1/1     Running   0          2m15s
[root@biz-master-48 istio-1.13.0]#
```

5.安装完后，需要校验安装是否成功

```shell
istioctl verify-install
```

```shell
[root@biz-master-48 istio-1.13.0]# istioctl verify-install
1 Istio control planes detected, checking --revision "default" only
✔ Deployment: istio-ingressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-ingressgateway.istio-system checked successfully
✔ Role: istio-ingressgateway-sds.istio-system checked successfully
✔ RoleBinding: istio-ingressgateway-sds.istio-system checked successfully
✔ Service: istio-ingressgateway.istio-system checked successfully
✔ ServiceAccount: istio-ingressgateway-service-account.istio-system checked successfully
✔ Deployment: istio-egressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-egressgateway.istio-system checked successfully
✔ Role: istio-egressgateway-sds.istio-system checked successfully
✔ RoleBinding: istio-egressgateway-sds.istio-system checked successfully
✔ Service: istio-egressgateway.istio-system checked successfully
✔ ServiceAccount: istio-egressgateway-service-account.istio-system checked successfully
✔ ClusterRole: istiod-istio-system.istio-system checked successfully
✔ ClusterRole: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istio-reader-service-account.istio-system checked successfully
✔ Role: istiod-istio-system.istio-system checked successfully
✔ RoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istiod-service-account.istio-system checked successfully
✔ CustomResourceDefinition: wasmplugins.extensions.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: destinationrules.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: envoyfilters.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: gateways.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: proxyconfigs.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: serviceentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: sidecars.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: virtualservices.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadgroups.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: authorizationpolicies.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: peerauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: requestauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: telemetries.telemetry.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: istiooperators.install.istio.io.istio-system checked successfully
✔ ClusterRole: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRole: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ConfigMap: istio.istio-system checked successfully
✔ Deployment: istiod.istio-system checked successfully
✔ ConfigMap: istio-sidecar-injector.istio-system checked successfully
✔ MutatingWebhookConfiguration: istio-sidecar-injector.istio-system checked successfully
✔ PodDisruptionBudget: istiod.istio-system checked successfully
✔ ClusterRole: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ Role: istiod.istio-system checked successfully
✔ RoleBinding: istiod.istio-system checked successfully
✔ Service: istiod.istio-system checked successfully
✔ ServiceAccount: istiod.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.11.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.11.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.12.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.12.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.13.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.13.istio-system checked successfully
✔ ValidatingWebhookConfiguration: istio-validator-istio-system.istio-system checked successfully
Checked 15 custom resource definitions
Checked 3 Istio Deployments
✔ Istio is installed and verified successfully
```

6.安装控制层支持的组件grafana和jaeger

grafana：可视化proxies暴露的metrics

jaeger：分布式链路追踪系统，可视化mesh的请求流

```shell
[root@biz-master-48 istio-1.13.0]# kubectl apply -f samples/addons/
serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

查看效果：

```shell
[root@biz-master-48 istio-1.13.0]# kubectl get po -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
grafana-6c5dc6df7c-rjrrb               1/1     Running   0          3m4s
istio-egressgateway-6cf5fb4756-9vrzq   1/1     Running   0          13m
istio-ingressgateway-dc9c8f588-bd6bj   1/1     Running   0          13m
istiod-7586c7dfd8-gffc6                1/1     Running   0          13m
jaeger-9dd685668-nckfw                 1/1     Running   0          3m4s
kiali-699f98c497-vclps                 1/1     Running   0          3m4s
prometheus-699b7cc575-9h9hk            2/2     Running   0          3m4s
```

##### Getting to know the Istio control plane

控制面组件如下：

![image-20220325104314136](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325104314136.png)

istiod：负责控制面的组件。

###### Istiod

`Istiod`是负责控制面的组件。`Istiod`有时候被称之为`Istio Pilot`（领航员），它负责将由`user/operator`指定的高级别Istio配置下发到数据面的service proxy中。

![image-20220325105222205](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325105222205.png)

`Istio`使用`Envoy`作为它的`service proxy`，所以`service-proxy-specific`配置将会被翻译为`Envoy`的配置。

> This data-plane API exposed by istiod implements Envoy's discovery APIS

This data-plane API exposed by istiod implements Envoy's discovery APIS（ Istiod暴露的这个数据平面API实现了Envoy的发现API），像service discovery（listener discovery service [LDS]），endpoints（endpoint discovery service [EDS]），和routing rules（路由规则）（route discovery service [RDS]）被称为`xDS APIs`。

###### Ingress and egress gateway

我们可能需要与集群外的服务打交道，那么就需要配置`istio`去指定什么流量可以允许进入集群，什么流量可以被允许从集群中出去。这里就涉及到`ingressgateway`和`istio-egressgateway`。

![image-20220325113351446](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325113351446.png)

###### Deploying your first application

下载示例：

```shell
git clone https://github.com/istioinaction/book-source-code
```

创建命名空间

```shell
kubectl create namespace istioinaction
```

设置上下文，设置后访问这个`namespace`不需要指定`-n` 

```shell
kubectl config set-context $(kubectl config current-context) --namespace=istioinaction
```

```shell
[root@biz-master-48 book-source-code]# kubectl get all
No resources found in istioinaction namespace.
[root@biz-master-48 book-source-code]#
```

现在不指定`-n`的时候，默认访问的命名空间由`default`变为`istioinaction`了。

我们创建以下资源,部署一个`catalog`服务：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalog
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalog
  name: catalog
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalog
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v1
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v1
  template:
    metadata:
      labels:
        app: catalog
        version: v1
    spec: 
      serviceAccountName: catalog
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false
```

执行以下命令,进行服务的注入：

```shell
[root@biz-master-48 ~]# kubectl label namespace istioinaction istio-injection=enabled
namespace/istioinaction labeled
[root@biz-master-48 ~]# kubectl apply -f book-source-code/services/catalog/kubernetes/catalog.yaml 
serviceaccount/catalog created
service/catalog created
deployment.apps/catalog created
[root@biz-master-48 ~]#
```

查看效果：

```shell
[root@biz-master-48 ~]# kubectl get po
NAME                       READY   STATUS    RESTARTS   AGE
catalog-68666d4988-7qwd9   2/2     Running   0          2m2s
```

查看容器,会发现有一个叫`istio-proxy`容器，它就是`sidecar`：

```yaml
Containers:
  catalog:
    Container ID:   docker://4df217235c7b812dba4cc669643cc02cf0eb72d5ae330dc68cef5ab5973b6bda
    Image:          istioinaction/catalog:latest
    Image ID:       docker-pullable://istioinaction/catalog@sha256:5ebcaad0491ea6e4ab303a48b51b90856ae2aaf40ea8fc5020f943a261a7b8df
    Port:           3000/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 25 Mar 2022 04:02:00 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      KUBERNETES_NAMESPACE:  istioinaction (v1:metadata.namespace)
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fpxs4 (ro)
  istio-proxy:
    Container ID:  docker://db6bb7a0e8d0ef9993be174222cc344f1f3494bbdeddc8186f494d8719a69963
    Image:         docker.io/istio/proxyv2:1.13.0
    Image ID:      docker-pullable://istio/proxyv2@sha256:2919336a667e83f3a1731a21252ff2f88c74218bffa9660638e0d190071a4510
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --log_output_level=default:info
      --concurrency
      2
    State:          Running
      Started:      Fri, 25 Mar 2022 04:02:00 +0000
    Ready:          True
```

访问这个服务：

```shell
[root@biz-master-48 ~]# kubectl get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
catalog   ClusterIP   10.21.137.250   <none>        80/TCP    108m
[root@biz-master-48 ~]# curl 10.21.137.250:/items/1
{
  "id": 1,
  "color": "amber",
  "department": "Eyewear",
  "name": "Elinor Glasses",
  "price": "282.00"
}[root@biz-master-48 ~]#
```

接下来，我们部署一个`webapp`服务，该服务可以聚合从其他从其他服务获取的数据，并在浏览器可视化展示。

```shell
[root@biz-master-48 ~]# kubectl apply -f book-source-code/services/webapp/kubernetes/webapp.yaml 
serviceaccount/webapp created
service/webapp created
deployment.apps/webapp created
```

查看部署

```shell
[root@biz-master-48 ~]# kubectl get po
NAME                       READY   STATUS    RESTARTS   AGE
catalog-68666d4988-7qwd9   2/2     Running   0          113m
webapp-7f47f74b9d-tbtbf    2/2     Running   0          29s
```

访问`webapp`

```shell
[root@biz-master-48 ~]# kubectl get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
catalog   ClusterIP   10.21.137.250   <none>        80/TCP         125m
webapp    NodePort    10.21.222.196   <none>        80:31561/TCP   12m
```

![image-20220325141133226](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325141133226.png)

###### Exploring the power of Istio with resilience,observability,and traffic control

现在，我们使用`Istio ingress gateway`去暴露`webapp`服务。使用`Istio ingress gateway`，我们可以从集群外访问我们的服务，作用类似于`ingress controller`

```shell
[root@biz-master-48 ~]# kubectl apply -f book-source-code/ch2/ingress-gateway.yaml 
gateway.networking.istio.io/outfitters-gateway created
virtualservice.networking.istio.io/webapp-virtualservice created
```

查看具体的内容，ingress-gateway：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: outfitters-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: webapp-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - outfitters-gateway
  http:
  - route:
    - destination:
        host: webapp
        port:
          number: 80
```

我们已经让istio意识到kubernetes边缘的webapp服务，我们尝试一下该服务是否可达。

首先，我们要获取`Istio gateway` 监听的端点：

```shell
kubectl port-forward deploy/istio-ingressgateway -n istio-system 8080:8080 --address 0.0.0.0
```

当在集群外无法访问8080端口的时候，使用`--address 0.0.0.0`

访问界面，输入http://10.10.13.48:8080：

![image-20220325151921072](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325151921072.png)

如果你遇到了一些错误，导致无法访问这个界面，此时需要做一些排查工作。

检查`gateway`有一个路由

```shell
[root@biz-master-48 ~]# istioctl proxy-config routes deploy/istio-ingressgateway.istio-system
NAME          DOMAINS     MATCH                  VIRTUAL SERVICE
http.8080     *           /*                     webapp-virtualservice.istioinaction
              *           /stats/prometheus*     
              *           /healthz/ready*  
```

出现`webapp-virtualservice.istioinaction`，说明我们的路由没有问题。如果没有，那么检查gateway、virtualservice是否安装。

```shell
[root@biz-master-48 ~]# kubectl get gateway
NAME                 AGE
outfitters-gateway   29m
[root@biz-master-48 ~]# kubectl get virtualservice
NAME                    GATEWAYS                 HOSTS   AGE
webapp-virtualservice   ["outfitters-gateway"]   ["*"]   29m
```

-  Istio observability

`Istio` 为两大类可观测性创建了遥测技术（telemetry）：top-line metrics、distributed tracing

**top-line metrics**

包含：request per second、number of failures、tail-latency percentiles

为了获取metrics，我们需要使用`prometheus`和`grafana`

访问我们之前部署的`grafana`,执行以下

```shell
istioctl dashboard grafana --address 0.0.0.0
```

```shell
[root@biz-master-48 ~]# istioctl dashboard grafana --address 0.0.0.0
http://0.0.0.0:3000
Failed to open browser; open http://0.0.0.0:3000 in your browser.
```

![image-20220325153818859](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325153818859.png)

执行以下脚本，然后查看服务对应的监控数据

```shell
while true; do curl http://10.21.137.250:80/api/catalog; sleep .5; done
```

![image-20220325154803709](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325154803709.png)

**distributed tracing**

包含：链路追踪，查找慢请求

我们使用`jaeger`处理分布式链路追踪，执行以下命令我们开启我们的`jaeger`

```shell
[root@biz-master-48 ~]# istioctl dashboard jaeger --address 0.0.0.0
http://0.0.0.0:16686
Failed to open browser; open http://0.0.0.0:16686 in your browser.
```

![image-20220325155912558](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325155912558.png)

我们多访问webapp地址，然后查看效果

```shell
http://10.10.13.48:8080/
```

![image-20220325160507953](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325160507953.png)

查看某个请求记录详情

![image-20220325160604730](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325160604730.png)

- Istio for resiliency（弹性）

一个弹性方面是在出现断断续续或瞬时的网络错误时进行请求的重试。

我们执行一个`chao`脚本，使catalog服务发生一些500的错误请求。

```shell
./book-source-code/bin/chaos.sh 500 50
```

500 代表 http 500，50代表 50%的请求时 500的响应。我们查看界面：

![image-20220325163937736](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220325163937736.png)

刷新的时候，基本上每两次的访问，会出现一次`Catalog`服务异常的现象。

让我们看看`Istio`如何使`webapp`和`catalog`之间的网络变得更有弹性：使用`VirtualService`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
    retries:
      attempts: 3
      retryOn: 5xx
      perTryTimeout: 2s
```

```shell
[root@biz-master-48 ~]# kubectl apply -f book-source-code/ch2/catalog-virtualservice.yaml 
virtualservice.networking.istio.io/catalog created
```

- Istio for traffic routing

流量路由。我们使用`Istio`中`DestinationRule`去通过version分离我们的服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
spec:
  host: catalog
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
      version: v2
```

我们安装一个v2版本的`catalog`，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v2
  name: catalog-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v2
  template:
    metadata:
      labels:
        app: catalog
        version: v2
    spec:
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: SHOW_IMAGE
          value: "true"
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false
```

执行安装：

```shell
[root@biz-master-48 ~]# kubectl apply -f book-source-code/services/catalog/kubernetes/catalog-deployment-v2.yaml 
deployment.apps/catalog-v2 created
[root@biz-master-48 ~]# kubectl get po
NAME                         READY   STATUS    RESTARTS   AGE
catalog-68666d4988-7qwd9     2/2     Running   0          5h25m
catalog-v2-86854b8c7-v5ghx   2/2     Running   0          12s
webapp-7f47f74b9d-tbtbf      2/2     Running   0          3h32m
```

安装 `catalog`对应的`DestinationRule`

```shell
[root@biz-master-48 ~]# kubectl apply -f book-source-code/ch2/catalog-destinationrule.yaml 
destinationrule.networking.istio.io/catalog created
[root@biz-master-48 ~]#
```

如果我们访问`catalog`服务很多次，我们会发现一些请求的响应包含了一个新的`imageUrl`属性。`imageUrl`是`catalog-v2` 版本中新增加的字，如：

```shell
[root@biz-master-48 ~]# kubectl get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
catalog   ClusterIP   10.21.137.250   <none>        80/TCP         2d22h
webapp    NodePort    10.21.222.196   <none>        80:31561/TCP   2d20h
[root@biz-master-48 ~]# curl 10.21.137.250/items/1
{
  "id": 1,
  "color": "amber",
  "department": "Eyewear",
  "name": "Elinor Glasses",
  "price": "282.00"
}[root@biz-master-48 ~]# curl 10.21.137.250/items/1
{
  "id": 1,
  "color": "amber",
  "department": "Eyewear",
  "name": "Elinor Glasses",
  "price": "282.00",
  "imageUrl": "http://lorempixel.com/640/480"
```

接下来，我们创建一个`catalog` `VirtualService` 将所有访问`catalog`的流量打到`catalog`的`v1`版本，`VirtualService`的定义如下。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
```

apply 这个yaml

我们再次访问`catalog`，通过 `webapp`的svc ip访问，`webapp`的`svc ip`为`10.21.222.196`

```shell
[root@biz-master-48 ~]# kubectl get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
catalog   ClusterIP   10.21.137.250   <none>        80/TCP         2d22h
webapp    NodePort    10.21.222.196   <none>        80:31561/TCP   2d20h
```

```shell
while true; do curl http://10.21.222.196/api/catalog | python -m json.tool ; sleep .5 ; done 
```

效果如下

```shell
[root@biz-master-48 ~]# while true; do curl http://10.21.222.196/api/catalog | python -m json.tool ; sleep .5 ; done  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   357  100   357    0     0  63568      0 --:--:-- --:--:-- --:--:-- 71400
[
    {
        "color": "amber",
        "department": "Eyewear",
        "id": 1,
        "name": "Elinor Glasses",
        "price": "282.00"
    },
    {
        "color": "cyan",
        "department": "Clothing",
        "id": 2,
        "name": "Atlas Shirt",
        "price": "127.00"
    },
    {
        "color": "teal",
        "department": "Clothing",
        "id": 3,
        "name": "Small Metal Shoes",
        "price": "232.00"
    },
    {
        "color": "red",
        "department": "Watches",
        "id": 4,
        "name": "Red Dragon Watch",
        "price": "232.00"
    }
]
```

从上面的结果可知，所有的流量都打到`catalog v1`版本上面去了。

我们再修改一下`VirtualService`，当访问`catalog`时指定了某个定义的header，那么就访问v2版本，否则访问v1版本。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - match:
    - headers:
        x-dark-launch:
          exact: "v2"
    route:
    - destination:
        host: catalog
        subset: version-v2
  - route:
    - destination:
        host: catalog
        subset: version-v1
```

apply 这个yaml

我们再次访问`catalog`

```shell
[root@biz-master-48 ~]# kubectl apply -f book-source-code/ch2/catalog-virtualservice-dark-v2.yaml 
virtualservice.networking.istio.io/catalog configured
[root@biz-master-48 ~]# while true; do curl http://10.21.222.196/api/catalog -H "x-dark-launch: v2" | python -m json.tool ; sleep .5 ; done 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   529  100   529    0     0  10809      0 --:--:-- --:--:-- --:--:-- 11020
[
    {
        "color": "amber",
        "department": "Eyewear",
        "id": 1,
        "imageUrl": "http://lorempixel.com/640/480",
        "name": "Elinor Glasses",
        "price": "282.00"
    },
    {
        "color": "cyan",
        "department": "Clothing",
        "id": 2,
        "imageUrl": "http://lorempixel.com/640/480",
        "name": "Atlas Shirt",
        "price": "127.00"
    },
    {
        "color": "teal",
        "department": "Clothing",
        "id": 3,
        "imageUrl": "http://lorempixel.com/640/480",
        "name": "Small Metal Shoes",
        "price": "232.00"
    },
    {
        "color": "red",
        "department": "Watches",
        "id": 4,
        "imageUrl": "http://lorempixel.com/640/480",
        "name": "Red Dragon Watch",
        "price": "232.00"
    }
]
```

ok，符合期望。

至此，简单的demo已完成，我们清理一些资源

```shell
[root@biz-master-48 ~]# kubectl delete deploy,svc,gateway,vs,dr --all -n istioinaction
deployment.apps "catalog" deleted
deployment.apps "catalog-v2" deleted
deployment.apps "nginx-deployment" deleted
deployment.apps "webapp" deleted
service "catalog" deleted
service "webapp" deleted
gateway.networking.istio.io "outfitters-gateway" deleted
virtualservice.networking.istio.io "catalog" deleted
virtualservice.networking.istio.io "webapp-virtualservice" deleted
destinationrule.networking.istio.io "catalog" deleted
```

#### Istio's data plane: The Envoy proxy

本章节内容

- 理解`Envoy` proxy
- 了解`Envoy`在istio中有什么核心能力
- 使用静态配置配置`Envoy`
- 使用`Envoy` Admin API 进行debug

##### What is  the Envoy proxy?

`Envoy`从2016年开始开源，由2017年加入CNCF。

**`Envoy`有两个重要的原则**：

**（1）网络对应用来说应该是透明的**

**（2）当发生网络和应用的问题时，可以很简单的找到原因**

`Envoy` 是一个代理，所以在探索它之前，我们先了解一下什么是代理。我们之前提到了代理（`proxy`）在网络架构中是一个处于client和server中间的一个媒介，如下图。由于它是中间件，所以它可以提供一些额外的特性，如security、privacy和policy。

![image-20220328141154042](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220328141154042.png)

`Envoy`proxy具体地说是一个应用级别的代理。`Envoy`可以理解应用程序在与其他服务通信时可能使用的第7层协议。

作为代理，envoy旨在通过运行应用程序之外的进程来保护开发人员免受网络问题的影响。

###### Envoy's core features

`Envoy`有很多的特性。当了解这些特性之前，我们先熟悉以下`Envoy`的概念：

`Listeners`: 就是端口。暴露一个应用可以连接的外部端口

`Routes`: 就是路由规则。路由规则用于如何处理传入`Listeners`的流量

`Clusters`: 就是具体的应用服务的service。Envoy可以路由流量到的特定上游服务

![image-20220328150141654](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220328150141654.png)

`Downstream`：如果A服务访问B服务，那么A服务就是downstream

`Upstream`：如果A服务访问B服务，那么B服务就是upstream

<u>out of the box（开箱即用）</u>

###### Comparing Envoy to other proxies

`Envoy`在以下方面比其他的代理表现的更加出色：

- 对WebAssembly更据扩展性
- 开源
- 支持HTTP 2.0
- 很深的协议指标收集
- C++编写
- 动态配置，不需要热加载

##### Configuring Envoy

`Envoy`的配置文件可以是`JSON`或者`YAML`类型格式。`Envoy`的配置文件指定`listeners`，`routes`，`clusters`以及服务端配置，比如：是否开启`Admin API`，访问日志应该放在哪里，链路追踪的配置等等。`Envoy`有好几个版本，本书中使用v3版本。

`Envoy v3`配置API构建在gRPC之上。

###### Static configuration

我们可以指定`listeners`，`routes`，`clusters`使用配置文件。

![image-20220328164139519](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220328164139519.png)

###### Dynamic configuration

`Envoy`可以使用一系列的APIs去做内置配置的更新。

`Envoy`使用以下APIs去做动态配置：

- Listener discovery service（LDS）：它是一个允许`Envoy`去查找什么`listeners`应该被暴露在这个代理的API
- Route discovery service（RDS）：它是`listeners`配置中配置哪个`route`被使用的API
- Cluster discovery service（CDS）：
- Endpoint  discovery service（EDS）：
- Secret  discovery service（EDS）：
- Aggregate discovery service（ADS）：

以上所有的APIs都是作为`xDS`服务被引用的。

##### Envoy in action

`Envoy`以C++编写且编译在指定环境中。所以入门`Envoy`最好的方式是启动一个docker 容器。

```shell
docker pull envoyproxy/envoy:v1.19.0
docker pull curlimages/curl
docker pull citizenstig/httpbin
```

`httpbin`：就是用于查看我发出去的请求到底是什么样子的。地址：http://httpbin.org/

一旦我们启动了httpbin服务，我们将启动envoy并配置它来代理所有的流量到httpbin服务。然后，我们将开启一个client 应用去调用这个代理。

![image-20220328173004611](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220328173004611.png)

启动`httpbin`

```shell
[root@biz-master-48 mark]# docker run -d --name httpbin citizenstig/httpbin
81dbbef2009f5173e2fccbe0a2c022656d55ba4f6fd2ae77acd4d5f66c40074f
```

访问`httpbin`服务

```shell
[root@biz-master-48 ~]# docker run -it --rm --link httpbin curlimages/curl \ curl  http://httpbin:8000/headers
curl: (3) URL using bad/illegal format or missing URL
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin:8000", 
    "User-Agent": "curl/7.82.0-DEV"
  }
}
```

`docker --link`可以通过容器名互相通信，容器间共享环境变量。
`docker --link`主要用来解决两个容器通过ip地址连接时`容器ip地址`会变的问题.

启动`Envoy`

```shell
# 查看envoy支持的命令参数
docker run -it --rm envoyproxy/envoy:v1.19.0 envoy --help
# 启动envoy
docker run -it --rm envoyproxy/envoy:v1.19.0 envoy
```

我们发现启动报错了

```shell
[2022-03-28 09:46:13.218][1][critical][main] [source/server/server.cc:112] error initializing configuration '': At least one of --config-path or --config-yaml or Options::configProto() should be non-empty
[2022-03-28 09:46:13.218][1][info][main] [source/server/server.cc:855] exiting
At least one of --config-path or --config-yaml or Options::configProto() should be non-empty
```

发生了啥？我们尝试启动`proxy`，但是我们没用传递一个有效的配置文件。为了修复这个报错，我们运行的时候指定以下这个配置。

```yaml
admin:
  address:
    socket_address: { address: 0.0.0.0, port_value: 15000 }

static_resources:
  listeners:
  - name: httpbin-demo
    address:
      socket_address: { address: 0.0.0.0, port_value: 15001 }
    filter_chains:
    - filters:
      - name:  envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          http_filters:
          - name: envoy.filters.http.router
          route_config:
            name: httpbin_local_route
            virtual_hosts:
            - name: httpbin_local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  auto_host_rewrite: true
                  cluster: httpbin_service
  clusters:
    - name: httpbin_service
      connect_timeout: 5s
      type: LOGICAL_DNS
      dns_lookup_family: V4_ONLY
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: httpbin
        endpoints:
        - lb_endpoints:
          - endpoint:
              address:
                socket_address:
                  address: httpbin
                  port_value: 8000
```

大体上，我们暴露了一个单独的`listener`在15001端口，然后路由所有的流量到httpbin实例上。我们再次启动`Envoy`

```shell
[root@biz-master-48 ~]# docker run -d --name envoy --link httpbin envoyproxy/envoy:v1.19.0  --config-yaml "$(cat book-source-code/ch3/simple.yaml)"
641837d5fa2ef391b9f590150d7fde05c6e94dfb2c431f8bf8fcc6000b21e1db
[root@biz-master-48 ~]# docker logs -f 641837d5fa2ef391b9f590150d7fde05c6e94dfb2c431f8bf8fcc6000b21e1db
[2022-03-28 10:24:08.919][1][info][main] [source/server/server.cc:785] all clusters initialized. initializing init manager
[2022-03-28 10:24:08.919][1][info][config] [source/server/listener_manager_impl.cc:834] all dependencies initialized. starting workers
[2022-03-28 10:24:08.920][1][info][main] [source/server/server.cc:804] starting main dispatch loop
```

这个`Envoy`proxy成功的监听在15001端口，让我们使用curl容器去调用这个服务：

```shell
[root@biz-master-48 ~]# docker run -it --rm --link envoy curlimages/curl \ curl http://envoy:15001/headers
curl: (3) URL using bad/illegal format or missing URL
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin", 
    "User-Agent": "curl/7.82.0-DEV", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000", 
    "X-Request-Id": "eaec3279-1fe9-4489-837b-d50676b694a0"
  }
}
```

即使我们调用了代理，流量也被正确地发送到httpbin服务。

- X-Envoy-Expected-Rq-Timeout-Ms
- X-Request-Id

它生成了一个新的X-Request-Id，该id可用于将跨集群的请求关联起来，并可能用于跨服务的多个跳转来满足请求。

###### Envoy‘s Admin API

为了探索`Envoy`的更多功能，让我们熟悉一下`Envoy`的管理API。 

如查看clusters：

```shell
docker run -it --rm --link envoy curlimages/curl \ curl http://envoy:15000/clusters
```

```shell
[root@biz-master-48 ~]# docker run -it --rm --link envoy curlimages/curl \ curl http://envoy:15000/clusters
curl: (3) URL using bad/illegal format or missing URL
httpbin_service::observability_name::httpbin_service
httpbin_service::default_priority::max_connections::1024
httpbin_service::default_priority::max_pending_requests::1024
httpbin_service::default_priority::max_requests::1024
httpbin_service::default_priority::max_retries::3
httpbin_service::high_priority::max_connections::1024
httpbin_service::high_priority::max_pending_requests::1024
httpbin_service::high_priority::max_requests::1024
httpbin_service::high_priority::max_retries::3
httpbin_service::added_via_api::false
httpbin_service::172.18.18.2:8000::cx_active::0
httpbin_service::172.18.18.2:8000::cx_connect_fail::0
httpbin_service::172.18.18.2:8000::cx_total::1
httpbin_service::172.18.18.2:8000::rq_active::0
httpbin_service::172.18.18.2:8000::rq_error::0
httpbin_service::172.18.18.2:8000::rq_success::1
httpbin_service::172.18.18.2:8000::rq_timeout::0
httpbin_service::172.18.18.2:8000::rq_total::1
httpbin_service::172.18.18.2:8000::hostname::httpbin
httpbin_service::172.18.18.2:8000::health_flags::healthy
httpbin_service::172.18.18.2:8000::weight::1
httpbin_service::172.18.18.2:8000::region::
httpbin_service::172.18.18.2:8000::zone::
httpbin_service::172.18.18.2:8000::sub_zone::
httpbin_service::172.18.18.2:8000::canary::false
httpbin_service::172.18.18.2:8000::priority::0
httpbin_service::172.18.18.2:8000::success_rate::-1.0
httpbin_service::172.18.18.2:8000::local_origin_success_rate::-1.0
```

###### Envoy request retries

##### How Envoy fits with Istio

### Securing，observing，and controlling your service’s network traffic

#### Istio gateways：Getting traffic into a cluster

本章内容

- 在集群中定义`entry points`
- 在你的集群中路由ingress流量到deployment
- 保护ingress流量
- 路由非HTTP流量

##### Traffic ingress concepts

流量进入网络的时候，最开始是路由到像一个看门人一样的`ingress`端点。ingress端点加强了哪些流量可以被允许进入本地网络的规则和配置。

###### Virtual IPs：Simplifying service access

其实就是访问一个指定服务的时候，先访问的是一个代理，然后通过代理分发到实际的服务。好处在于高可用，然后有代理后，又可以做一些附加的操作，如LB。

![image-20220329103256682](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220329103256682.png)

###### Virtual hosting：Multiple services from a single access point

在一个入口点托管多个不同的服务称为`virtual hosting`。一个入口托管多个不同服务，是说 多个域名使用同一个 IP 映射。然后多个client 就访问到同一个 reverse proxy了,那么 proxy怎么知道你访问的真实服务后端是哪个呢？答案是通过在请求头中加东西，让reverse proxy去识别。在不同的协议中，有不同的处理方式。

总而言之，最重要的是我们在Istio中看到的边缘入口功能使用虚拟IP路由和虚拟主机将服务流量路由到集群中。

![image-20220329104600457](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220329104600457.png)

##### Istio ingress gateways

istio入口网关(ingress gateway)充当网络的入口点，并使用`Envoy`代理(Envoy proxy)进行路由和负载均衡。

![image-20220329110607736](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220329110607736.png)

如果你想确认`Envoy proxy`确实是运行在`Istio ingress gateway`中，那么可以执行以下命令进行查看：

```shell
[root@biz-master-48 ~]# kubectl -n istio-system exec deploy/istio-ingressgateway -- ps
  PID TTY          TIME CMD
    1 ?        00:00:29 pilot-agent
   15 ?        00:02:51 envoy
   34 ?        00:00:00 ps
```

###### Specifying Gateway resources

为了在istio中配置ingress gateway，我们使用`Gateway`资源指定想要开放的端口以及这些端口允许的虚拟主机（virtual hosts）。如下示例：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: coolstore-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "webapp.istioinaction.io"
```

查看我们集群的`istio-ingressgateway `service

```bash
[root@biz-master-48 ~]# kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                 AGE
istio-ingressgateway   LoadBalancer   10.21.154.138   <pending>     15021:31095/TCP,80:32619/TCP,443:32552/TCP,312699/TCP   6d
```

`istio-ingressgateway`的Service的Type是LoadBalancer, 它的`EXTERNAL-IP`处于`pending`状态， 这是因为我们目前的环境并没有可用于Istio Ingress Gateway外部的负载均衡器，为了使得可以从外部访问， 通过修改`istio-ingressgateway`这个Service的externalIps，因为当前Kubernetes集群的kube-proxy启用了ipvs，所以这个指定一个VIP `10.10.13.48`作为externalIp。

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"istio-ingressgateway","install.operator.istio.io/owning-resource":"unknown","install.operator.istio.io/owning-resource-namespace":"istio-system","istio":"ingressgateway","istio.io/rev":"default","operator.istio.io/component":"IngressGateways","operator.istio.io/managed":"Reconcile","operator.istio.io/version":"1.13.0","release":"istio"},"name":"istio-ingressgateway","namespace":"istio-system"},"spec":{"ports":[{"name":"status-port","port":15021,"protocol":"TCP","targetPort":15021},{"name":"http2","port":80,"protocol":"TCP","targetPort":8080},{"name":"https","port":443,"protocol":"TCP","targetPort":8443},{"name":"tcp","port":31400,"protocol":"TCP","targetPort":31400},{"name":"tls","port":15443,"protocol":"TCP","targetPort":15443}],"selector":{"app":"istio-ingressgateway","istio":"ingressgateway"},"type":"LoadBalancer"}}
  creationTimestamp: "2022-03-25T02:24:23Z"
  labels:
    app: istio-ingressgateway
    install.operator.istio.io/owning-resource: unknown
    install.operator.istio.io/owning-resource-namespace: istio-system
    istio: ingressgateway
    istio.io/rev: default
    operator.istio.io/component: IngressGateways
    operator.istio.io/managed: Reconcile
    operator.istio.io/version: 1.13.0
    release: istio
  name: istio-ingressgateway
  namespace: istio-system
  resourceVersion: "1106775"
  uid: a4651af9-15d2-4840-9ce1-7fdc615ff9b4
spec:
  clusterIP: 10.21.154.138
  clusterIPs:
  - 10.21.154.138
  externalIPs:
  - 10.10.13.48
  externalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: status-port
    nodePort: 31095
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http2
    nodePort: 32619
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 32552
    port: 443
    protocol: TCP
    targetPort: 8443
  - name: tcp
    nodePort: 30481
    port: 31400
    protocol: TCP
    targetPort: 31400
  - name: tls
    nodePort: 32699
    port: 15443
    protocol: TCP
    targetPort: 15443
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  sessionAffinity: None
  type: LoadBalancer
```

如上：

```yaml
externalIPs:
  - 10.10.13.48
```

###### Gateway routing with virtual services

`virtual service`允许我们将流量从`ingress gateway`打到指定的`service`

###### Istio ingress gateway vs. Kubernetes Ingress

![image-20220331145315083](C:\Users\mark\AppData\Roaming\Typora\typora-user-images\image-20220331145315083.png)

k8s ingress仅支持支持http，而且只支持80和443端口作为ingress端点。

像kafka这种需要支持tcp直连的消息队列来说，k8s的ingress不能支持。

###### Istio ingress gateway vs. API gateways





























































































