## 部署Bookinfo示例程序

在下载的Istio安装包的samples目录中包含了示例应用程序。


### Bookinfo应用

部署一个样例应用，它由四个单独的微服务构成，用来演示多种 Istio 特性。这个应用模仿在线书店的一个分类，显示一本书的信息。页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。
Bookinfo 应用分为四个单独的微服务：

- productpage ：productpage 微服务会调用 details 和 reviews 两个微服务，用来生成页面。
- details ：这个微服务包含了书籍的信息。
- reviews ：这个微服务包含了书籍相关的评论。它还会调用 ratings 微服务。
- ratings ：ratings 微服务中包含了由书籍评价组成的评级信息。

reviews 微服务有 3 个版本：

- v1 版本不会调用 ratings 服务。
- v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

下图展示了这个应用的端到端架构。

Bookinfo 是一个异构应用，几个微服务是由不同的语言编写的。这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 reviews 服务具有多个版本。
 
要在 Istio 中运行这一应用，无需对应用自身做出任何改变。我们只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。这个过程所需的具体命令和配置方法由运行时环境决定，而部署结果较为一致.


所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。
 
接下来可以根据 Istio 的运行环境，按照下面的讲解完成应用的部署。
 
先验证kubernetes 集群是否启用了 自动Sidecar 注入。


### Sidecar 的自动注入

使用 Kubernetes 的 mutating webhook admission controller，可以进行 Sidecar 的自动注入。Kubernetes 1.9 以后的版本才具备这一能力。使用这一功能之前首先要检查 kube-apiserver 的进程，是否具备 admission-control 参数，并且这个参数的值中需要包含 MutatingAdmissionWebhook 以及 ValidatingAdmissionWebhook 两项，并且按照正确的顺序加载，这样才能启用 admissionregistration API：

    [root@centos-110 ~]# kubectl api-versions | grep admissionregistration
    admissionregistration.k8s.io/v1beta1


    #ps -ef | grep apiserver

确认kubernetes 集群已启用了自动Sidecar 注入。
 
部署bookinfo 服务
 
创建 book 命名空间（可选），也可以直接使用默认的 default 命名空间。
 
    >kubectl create ns book

给 book 命名空间设置标签：istio-injection=enabled：

    $ kubectl label namespace book istio-injection=enabled 
    $ kubectl get namespace -L istio-injection
     
    
    [root@centos-110 ~]# kubectl get ns -L istio-injection
    NAME   STATUSAGE   ISTIO-INJECTION
    book   Active1denabled
    defaultActive27d   
    istio-system   Active8ddisabled
    kube-publicActive27d   
    kube-systemActive27d   
    weave  Active21d  
 

如果集群使用的是自动 Sidecar 注入，只需简单的 kubectl 就能完成服务的部署。

    [root@centos-110 istio-1.0.0]# kubectl apply -n book -f samples/bookinfo/platform/kube/bookinfo.yaml
    service "details" created
    deployment.extensions "details-v1" created
    service "ratings" created
    deployment.extensions "ratings-v1" created
    service "reviews" created
    deployment.extensions "reviews-v1" created
    deployment.extensions "reviews-v2" created
    deployment.extensions "reviews-v3" created
    service "productpage" created
    deployment.extensions "productpage-v1" created
 
上面的命令会启动全部的四个服务，其中也包括了 reviews 服务的三个版本（v1、v2 以及 v3）。

如果服务部署有问题，可以使用如下命令删除已经部署的服务。

    #kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml -n book

确认所有的服务和Pods 都正常运行：

    [root@centos-110 istio-1.0.0]# kubectl get svc -n book
    NAME  TYPECLUSTER-IP   EXTERNAL-IP   PORT(S)AGE
    details   ClusterIP   10.103.243.183   <none>9080/TCP   9s
    productpage   ClusterIP   10.111.96.136<none>9080/TCP   7s
    ratings   ClusterIP   10.111.136.187   <none>9080/TCP   9s
    reviews   ClusterIP   10.97.99.117 <none>9080/TCP   8s


    [root@centos-110 istio-1.0.0]# kubectl get pods -n book -o wide
    NAME   READY STATUSRESTARTS   AGE   IP NODE
    details-v1-6865b9b99d-4rpfs2/2   Running   0  12m   10.244.2.140   centos-112
    productpage-v1-f8c8fb8-9zlmb   2/2   Running   0  12m   10.244.2.150   centos-112
    ratings-v1-77f657f55d-mvx9g2/2   Running   0  12m   10.244.2.135   centos-112
    reviews-v1-6b7f6db5c5-8jfq72/2   Running   0  12m   10.244.2.142   centos-112
    reviews-v2-7ff5966b99-pfhz82/2   Running   0  12m   10.244.2.146   centos-112
    reviews-v3-5df889bcff-vhk492/2   Running   0  12m   10.244.1.147   centos-111

验证Pod是否会自动注入Sidecar

    kubectl describe pod productpage-v1-f8c8fb8-8dnjm -n book
 
被注入Sidecar的Pod 会有2个容器，多出了一个 istio-proxy 容器及其对应的存储卷。

或者通过如下命令查看pod 中是否有istio-proxy 容器：

    [root@centos-110 ~]# kubectl get pod productpage-v1-f8c8fb8-8dnjm -o jsonpath='{.spec.containers[*].name}' -n book
    productpage istio-proxy
 
输出结果显示pod 中有2个容器，分别为 productpage 和 istio-proxy。
 
可以禁用 book 命名空间的自动注入功能，然后检查新建 Pod 是不是就不带有 Sidecar 容器了。

#### 确定入口IP和端口
 
执行以下命令以确定您的 Kubernetes 集群是否在支持外部负载均衡器的环境中运行。

    [root@centos-110 ~]# kubectl get svc istio-ingressgateway -n istio-system
    NAME   TYPE   CLUSTER-IPEXTERNAL-IP   PORT(S) AGE
    istio-ingressgateway   NodePort   10.106.84.2   <none>80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:32329/TCP,8060:32167/TCP,15030:31095/TCP,15031:30203/TCP   8d

如果 EXTERNAL-IP 设置了该值，则要求您的环境具有可用于 Ingress 网关的外部负载均衡器。如果 EXTERNAL-IP 值是 <none>（或一直是 <pending> ），则说明可能您的环境不支持为 ingress 网关提供外部负载均衡器的功能。在这种情况下，您可以使用 Service 的 node port 方式访问网关。
 
使用 kubectl patch 更新 istio-ingressgateway 服务网关类型
由于在我的kubernetes 集群环境中，不支持外部负载均衡器。因此，使用 node port 方式访问网关。
 
    [root@centos-110 istio-1.0.0]# kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"type":"NodePort"}}'
    service "istio-ingressgateway" patched
 
给 Bookinfo 应用定义 ingress gateway
现在 Bookinfo 服务都已正常运行中，我们需要让kubernetes集群外部可以访问应用，如通过浏览器访问。Istio gateway 可以实现这一目的。

    >kubectl apply -n book -f samples/bookinfo/networking/bookinfo-gateway.yaml

输出结果：
gateway.networking.istio.io "bookinfo-gateway" created
virtualservice.networking.istio.io "bookinfo" created


确认gateway创建成功：

    kubectl get gateway -n book
    NAME   AGE
    bookinfo-gateway   15h



也可以通过 istioctl 命令，验证 gateway和virtualservice 创建成功，具体命令如下所示：

    #istioctl get gateway -n book
    
    #istioctl get virtualservices -n book


前面，我们通过NodePort 方式来暴露istio-ingressgateway 服务，现在根据如下命令来获取 ingress ports：

    $ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
    $ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

 
获取 ingress IP 地址：

    $ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')


通过浏览器访问 ingress 服务
 
接下来就可以在浏览器的 URL 中使用 $INGRESS_HOST:$INGRESS_PORT（也就是 192.168.56.110:31380）进行访问，输入 http://192.168.56.110:31380/productpage 网址之后，显示信息如下：



如果刷新几次应用的页面，就会看到页面中会随机展示 reviews 服务的不同版本的效果（红色、黑色的星形或者没有显示）。reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。
 
理解原理
Gateway 配置资源允许外部流量进入 Istio 服务网格，并使 Istio 的流量管理和策略功能可用于边缘服务。
在前面的步骤中，我们在 Istio 服务网格中创建了一个服务，并展示了如何将服务的 HTTP 端点暴露给外部流量。



=================================================
清理 Bookinfo 示例应用
结束对 Bookinfo 示例应用的体验之后，就可以使用下面的命令来完成应用的删除和清理了。
 
在 Kubernetes 环境中完成删除
1. 删除路由规则，并终结应用的 Pod

    $ samples/bookinfo/platform/kube/cleanup.sh
 
* 确认应用已经关停 - 如果前面创建了namespace - book，则下面的命令也需要添加 -n book 参数。

    `# istioctl get gateway `
    `# istioctl get virtualservices  VirtualService` 
    `#kubectl get pods` 
 
卸载 istio
使用kubectl 卸载 istio


    kubectl delete -f install/kubernetes/istio-demo.yaml
 
手动清除额外的job 资源


    kubectl -n istio-system delete job --all
 
删除CRD


    kubectl delete -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system
 
参考链接：
注入 Istio sidecar
https://preliminary.istio.io/zh/docs/setup/kubernetes/sidecar-injection/#sidecar-%E7%9A%84%E8%87%AA%E5%8A%A8%E6%B3%A8%E5%85%A5
Install with helm
https://istio.io/docs/setup/kubernetes/helm-install/#option-2-install-with-helm-and-tiller-via-helm-install
控制 Ingress 流量
https://preliminary.istio.io/zh/docs/tasks/traffic-management/ingress/
Bookinfo 应用
https://preliminary.istio.io/zh/docs/examples/bookinfo/
Istio及Bookinfo示例程序安装试用笔记
https://zhaohuabing.com/2017/11/04/istio-install_and_example/

重要参考：
https://www.cnblogs.com/rickie/p/istio_kubernetes.html
