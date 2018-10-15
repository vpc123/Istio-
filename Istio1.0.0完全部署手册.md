## Istio1.0.0 完全部署手册


### 创建 istio 目录

    [root@centos-110 ~]# mkdir istio
    [root@centos-110 ~]# cd istio


#### 方案一：

    # 去下面的地址下载压缩包 
    # https://github.com/istio/istio/releases
    $ wget https://github.com/istio/istio/releases/download/1.0.0/istio-1.0.0-linux.tar.gz
    $ tar -zvxf istio-1.0.0-linux.tar.gz


    方案二：
    # 使用官方的安装脚本安装
    运行如下命令，自动下载并解压最新的发布包
    $ curl -L https://git.io/getLatestIstio | sh -

##### 这里我们建议使用方案一进行安装部署，因为我们可以指定版本信息还有可定制化的程度比较高。

##### 安装目录包含如下内容：

- 在 install/ 目录中包含了 Kubernetes 安装所需的 .yaml 文件
- samples/ 目录中是示例应用
- istioctl 客户端文件保存在 bin/ 目录之中。istioctl 的功能是手工进行 Envoy Sidecar 的注入，以及对路由规则、策略的管理
- istio.VERSION 配置文件
 

####  安装配置环境变量

    编辑/etc/profile，添加istioctl 到PATH
    [root@centos-110 istio-1.0.0]# vim /etc/profile
    在profile文件底部，增加如下一行：
    export PATH=$PATH:/root/istio/istio-1.0.0/bin


##### 执行source命令，使修改马上生效

    [root@centos-110 istio-1.0.0]# source /etc/profile
    [root@centos-110 istio-1.0.0]# echo $PATH
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/root/istio/istio-1.0.0/bin


#### 验证istioctl 安装成功

    [root@centos-110 bin]# istioctl version
    Version: 1.0.0
    GitRevision: 3a136c90ec5e308f236e0d7ebb5c4c5e405217f4
    User: root@71a9470ea93c
    Hub: gcr.io/istio-release
    GolangVersion: go1.10.1
    BuildStatus: Clean

#### 安装Istio的CRD（Custom Resource Definitions）

    #kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
 
#### 安装Istio - Sidecars之间不启用TLS认证

    $ kubectl apply -f install/kubernetes/istio-demo.yaml


需要拉取镜像：

参考上面的docker 命令，获取如下所有镜像：

gcr.io/istio-release/citadel:1.0.0
gcr.io/istio-release/galley:1.0.0
gcr.io/istio-release/proxyv2:1.0.0
gcr.io/istio-release/grafana:1.0.0
gcr.io/istio-release/mixer:1.0.0
gcr.io/istio-release/servicegraph:1.0.0
gcr.io/istio-release/pilot:1.0.0
gcr.io/istio-release/sidecar_injector:1.0.0


##### 下面这个镜像是sidecar 自动注入是需要使用的镜像：
gcr.io/istio-release/proxy_init:1.0.0

docker pull registry.cn-hangzhou.aliyuncs.com/istio-release/proxy_init:1.0.0
docker tag registry.cn-hangzhou.aliyuncs.com/istio-release/proxy_init:1.0.0 gcr.io/istio-release/proxy_init:1.0.0
docker rmi registry.cn-hangzhou.aliyuncs.com/istio-release/proxy_init:1.0.0


##### 验证安装结果
确认下列 Kubernetes 服务已经部署：istio-pilot、 istio-ingressgateway、istio-policy、istio-telemetry、prometheus 、istio-sidecar-injector（可选）。


    #kubectl get svc -n istio-system

确保所有相应的Kubernetes pod都已被部署且所有的容器都已启动并正在运行：istio-pilot-*、istio-ingressgateway-*、istio-egressgateway-*、istio-policy-*、istio-telemetry-*、istio-citadel-*、prometheus-*、istio-sidecar-injector-*（可选）。

    #kubectl get pods -o wide -n istio-system

从上面的输出可以看到，这里部署的主要是Istio控制面的服务，而数据面的网络代理要如何部署呢？
根据服务网格（Service Mesh）的架构可以得知，网络代理是随着应用程序以sidecar的方式部署的，在下面部署Bookinfo示例程序时会演示如何部署网络代理。
 
至此，Istio 已经安装完成了。:)



