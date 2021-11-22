# k8s-prometheus-grafana

[Kubernetes+Prometheus+Grafana部署笔记](https://www.cnblogs.com/kevingrace/p/11151649.html)

## 一、Prometheus介绍
之前已经详细介绍了Kubernetes集群部署篇，今天这里重点说下Kubernetes监控方案-Prometheus+Grafana。Prometheus（普罗米修斯）是一个开源系统监控和警报工具，最初是在SoundCloud建立的。自2012年成立以来，许多公司和组织都采用了普罗米修斯，该项目拥有一个非常活跃的开发者和用户社区。它现在是一个独立的开放源码项目，并且独立于任何公司，为了强调该点并澄清项目的治理结构，Prometheus在2016年加入了云计算基金会，成为继Kubernetes之后的第二个托管项目。 Prometheus是用来收集数据的,同时本身也提供强大的查询能力,结合Grafana即可以监控并展示出想要的数据。

### Prometheus的主要特征
- 多维度数据模型
- 灵活的查询语言 (PromQL)
- 不依赖分布式存储，单个服务器节点是自主的
- 以HTTP方式，通过pull模型拉去时间序列数据
- 也通过中间网关支持push模型
- 通过服务发现或者静态配置，来发现目标服务对象
- 支持多种多样的图表和界面展示，grafana也支持它

### Prometheus组件
Prometheus生态包括了很多组件，它们中的一些是可选的：
- 主服务Prometheus Server负责抓取和存储时间序列数据
- 客户库负责检测应用程序代码
- 支持短生命周期的PUSH网关
- 基于Rails/SQL仪表盘构建器的GUI
- 多种导出工具，可以支持Prometheus存储数据转化为HAProxy、StatsD、Graphite等工具所需要的数据存储格式
- 警告管理器 (AlertManaager)
- 命令行查询工具
- 其他各种支撑工具

### Prometheus监控Kubernetes集群过程中，通常情况为:
- 使用metric-server收集数据给k8s集群内使用，如kubectl,hpa,scheduler等
- 使用prometheus-operator部署prometheus，存储监控数据
- 使用kube-state-metrics收集k8s集群内资源对象数据
- 使用node_exporter收集集群中各节点的数据
- 使用prometheus收集apiserver，scheduler，controller-manager，kubelet组件数据
- 使用alertmanager实现监控报警
- 使用grafana实现数据可视化

### Prometheus架构
下面这张图说明了Prometheus的整体架构，以及生态中的一些组件作用
![Prometheus架构](Prometheus架构.png)

Prometheus整体流程比较简单，Prometheus 直接接收或者通过中间的 Pushgateway 网关被动获取指标数据，在本地存储所有的获取的指标数据，并对这些数据进行一些规则整理，用来生成一些聚合数据或者报警信息，Grafana 或者其他工具用来可视化这些数据。

Prometheus服务可以直接通过目标拉取数据，或者间接地通过中间网关拉取数据。它在本地存储抓取的所有数据，并通过一定规则进行清理和整理数据，并把得到的结果存储到新的时间序列中，PromQL和其他API可视化展示收集的数据在K8s中，关于集群的资源有metrics度量值的概念，有各种不同的exporter可以通过api接口对外提供各种度量值的及时数据，prometheus在与k8s融合工作的过程中就是通过与这些提供metric值的exporter进行交互，获取数据，整合数据，展示数据，触发告警的过程。

1. Prometheus获取metrics
- 对短暂生命周期的任务，采取拉的形式获取metrics (不常见)
- 对于exporter提供的metrics，采取拉的方式获取metrics(通常方式),对接的exporter常见的有：kube-apiserver 、cadvisor、node-exporter，也可根据应用类型部署相应的exporter，获取该应用的状态信息，目前支持的应用有：nginx/haproxy/mysql/redis/memcache等。

2. Prometheus数据汇总及按需获取
可以按照官方定义的expr表达式格式，以及PromQL语法对相应的指标进程过滤，数据展示及图形展示。不过自带的webui较为简陋，但prometheus同时提供获取数据的api，grafana可通过api获取prometheus数据源，来绘制更精细的图形效果用以展示。
expr书写格式及语法参考官方文档：https://prometheus.io/docs/prometheus/latest/querying/basics/

3. Prometheus告警推送
prometheus支持多种告警媒介，对满足条件的告警自动触发告警，并可对告警的发送规则进行定制，例如重复间隔、路由等，可以实现非常灵活的告警触发。

### Prometheus适用场景
Prometheus在记录纯数字时间序列方面表现非常好。它既适用于面向服务器等硬件指标的监控，也适用于高动态的面向服务架构的监控。对于现在流行的微服务，Prometheus的多维度数据收集和数据筛选查询语言也是非常的强大。Prometheus是为服务的可靠性而设计的，当服务出现故障时，它可以使你快速定位和诊断问题。它的搭建过程对硬件和服务没有很强的依赖关系。

### Prometheus不适用场景
Prometheus，它的价值在于可靠性，甚至在很恶劣的环境下，你都可以随时访问它和查看系统服务各种指标的统计信息。 如果你对统计数据需要100%的精确，它并不适用，例如：它不适用于实时计费系统

## 二、Prometheus+Grafana部署
依据之前部署好的Kubernetes容器集群管理环境为基础，继续部署Prometheus+Grafana。如果不部署metrics-service的话，则要部署kube-state-metrics，它专门监控的是k8s资源对象，如pod，deployment，daemonset等，并且它会被Prometheus以endpoint方式自动识别出来。记录如下: (k8s-prometheus-grafana.git打包后下载地址：https://pan.baidu.com/s/1nb-QCOc7lgmyJaWwPRBjPg  提取密码: bh2e）

由于Zabbix不适用于容器环境下的指标采集和监控告警，为此使用了与K8s原生的监控工具Prometheus；Prometheus可方便地识别K8s中相关指标，并以极高的效率和便捷的配置实现了指标采集和监控告警。具体工作:
1. 在K8s集群中部署Prometheus，以K8s本身的特性实现了Prometheus的高可用；
2. 优化Prometheus配置，实现了配置信息的热加载，在更新配置时无需重启进程；
3. 配置了Prometheus抓取规则，以实现对apiserver/etcd/controller-manager/scheduler/kubelet/kube-proxy以及K8s集群内运行中容器的信息采集；
4. 配置了Prometheus及告警规则，以实现对采集到的信息进行计算和告警；

### 1.  Prometheus和Grafana部署
> 1）在k8s-master01节点上进行安装部署。安装git，并下载相关yaml文件
> [root@k8s-master01 ~]# cd /opt/k8s/work/
> [root@k8s-master01 work]# git clone https://github.com/redhatxl/k8s-prometheus-grafana.git
>     
> 2）在所有的node节点下载监控所需镜像
> [root@k8s-master01 work]# source /opt/k8s/bin/environment.sh
> [root@k8s-master01 work]# for node_node_ip in ${NODE_NODE_IPS[@]}
>   do
>     echo ">>> ${node_node_ip}"
>     ssh root@${node_node_ip} "docker pull prom/node-exporter"
>   done
>     
> [root@k8s-master01 work]# for node_node_ip in ${NODE_NODE_IPS[@]}
>   do
>     echo ">>> ${node_node_ip}"
>     ssh root@${node_node_ip} "docker pull prom/prometheus:v2.0.0"
>   done
>     
> [root@k8s-master01 work]# for node_node_ip in ${NODE_NODE_IPS[@]}
>   do
>     echo ">>> ${node_node_ip}"
>     ssh root@${node_node_ip} "docker pull grafana/grafana:4.2.0"
>   done
>     
> 3）采用daemonset方式部署node-exporter组件
> [root@k8s-master01 work]# cd k8s-prometheus-grafana/
> [root@k8s-master01 k8s-prometheus-grafana]# ls
> grafana  node-exporter.yaml  prometheus  README.md
>   
> 这里需要修改下默认的node-exporter.yaml文件内存，修改后的node-exporter.yaml内容如下：
> [root@k8s-master01 k8s-prometheus-grafana]# cat node-exporter.yaml
> ---
> apiVersion: extensions/v1beta1
> kind: DaemonSet
> metadata:
>   name: node-exporter
>   namespace: kube-system
>   labels:
>     k8s-app: node-exporter
> spec:
>   template:
>     metadata:
>       labels:
>         k8s-app: node-exporter
>     spec:
>       containers:
>       - image: prom/node-exporter
>         name: node-exporter
>         ports:
>         - containerPort: 9100
>           protocol: TCP
>           name: http
>       tolerations:
>       hostNetwork: true
>       hostPID: true
>       hostIPC: true
>       restartPolicy: Always
>   
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   annotations:
>     prometheus.io/scrape: 'true'
>     prometheus.io/app-metrics: 'true'
>     prometheus.io/app-metrics-path: '/metrics'
>   labels:
>     k8s-app: node-exporter
>   name: node-exporter
>   namespace: kube-system
> spec:
>   ports:
>   - name: http
>     port: 9100
>     nodePort: 31672
>     protocol: TCP
>   type: NodePort
>   selector:
>     k8s-app: node-exporter
>   
> [root@k8s-master01 k8s-prometheus-grafana]# kubectl create -f  node-exporter.yaml
>     
> 稍等一会儿，查看node-exporter部署是否成功了
> [root@k8s-master01 k8s-prometheus-grafana]# kubectl get pods -n kube-system|grep "node-exporter*"
> node-exporter-9h2z6                     1/1     Running   0          117s
> node-exporter-sk4g4                     1/1     Running   0          117s
> node-exporter-stlwb                     1/1     Running   0          117s
>     
> 4）部署prometheus组件
> [root@k8s-master01 k8s-prometheus-grafana]# cd prometheus/
>     
> 4.1）部署rbac文件
> [root@k8s-master01 prometheus]# kubectl create -f rbac-setup.yaml
>     
> 4.2）以configmap的形式管理prometheus组件的配置文件
> [root@k8s-master01 prometheus]# kubectl create -f configmap.yaml
>     
> 4.3）Prometheus deployment 文件
> [root@k8s-master01 prometheus]# kubectl create -f prometheus.deploy.yaml
>     
> 4.4）Prometheus service文件
> [root@k8s-master01 prometheus]# kubectl create -f prometheus.svc.yaml
>     
> 5）部署grafana组件
> [root@k8s-master01 prometheus]# cd ../grafana/
> [root@k8s-master01 grafana]# ll
> total 12
> -rw-r--r-- 1 root root 1449 Jul  8 17:19 grafana-deploy.yaml
> -rw-r--r-- 1 root root  256 Jul  8 17:19 grafana-ing.yaml
> -rw-r--r-- 1 root root  225 Jul  8 17:19 grafana-svc.yaml
>     
> 5.1）grafana deployment配置文件
> [root@k8s-master01 grafana]# kubectl create -f grafana-deploy.yaml
>     
> 5.2）grafana service配置文件
> [root@k8s-master01 grafana]# kubectl create -f grafana-svc.yaml
>     
> 5.3）grafana ingress配置文件
> [root@k8s-master01 grafana]# kubectl create -f grafana-ing.yaml
>     
> 6）web访问界面配置
> [root@k8s-master01 k8s-prometheus-grafana]# kubectl get pods -n kube-system
> NAME                                    READY   STATUS    RESTARTS   AGE
> coredns-5b969f4c88-pd5js                1/1     Running   0          30d
> grafana-core-5f7c6c786b-x8prc           1/1     Running   0          17d
> kube-state-metrics-5dd55c764d-nnsdv     2/2     Running   0          23d
> kubernetes-dashboard-7976c5cb9c-4jpzb   1/1     Running   0          16d
> metrics-server-54997795d9-rczmc         1/1     Running   0          24d
> node-exporter-9h2z6                     1/1     Running   0          74s
> node-exporter-sk4g4                     1/1     Running   0          74s
> node-exporter-stlwb                     1/1     Running   0          74s
> prometheus-8697656888-2vwbw             1/1     Running   0          10d
>     
> [root@k8s-master01 k8s-prometheus-grafana]# kubectl get svc -n kube-system
> NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
> grafana                         NodePort    10.254.95.120    <none>        3000:31821/TCP                  17d
> kube-dns                        ClusterIP   10.254.0.2       <none>        53/UDP,53/TCP,9153/TCP          30d
> kube-state-metrics              NodePort    10.254.228.212   <none>        8080:30978/TCP,8081:30872/TCP   23d
> kubernetes-dashboard-external   NodePort    10.254.223.104   <none>        9090:30090/TCP                  16d
> metrics-server                  ClusterIP   10.254.135.197   <none>        443/TCP                         24d
> node-exporter                   NodePort    10.254.72.22     <none>        9100:31672/TCP                  2m11s
> prometheus                      NodePort    10.254.241.170   <none>        9090:30003/TCP                  10d
>     
> 7）查看node-exporter （http://node-ip:31672/）
> http://172.16.60.244:31672/
> http://172.16.60.245:31672/
> http://172.16.60.246:31672/
>    
> 8）prometheus对应的nodeport端口为30003，通过访问http://node-ip:30003/targets 可以看到prometheus已经成功连接上了k8s的apiserver
> http://172.16.60.244:30003/targets
> http://172.16.60.245:30003/targets
> http://172.16.60.246:30003/targets
> 

[Kubernetes+Prometheus+Grafana部署笔记](http://blog.51cto.com/blogger/publish/2160569)
