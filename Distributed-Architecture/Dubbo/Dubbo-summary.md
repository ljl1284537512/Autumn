## 目录

[为什么需要Dubbo ? ](#为什么需要Dubbo ?)

[Dubbo的架构](#Dubbo的架构)

[Dubbo的使用](#Dubbo的使用)

[Dubbo注册中心原理](#Dubbo注册中心原理)

[如何快速启动Dubbo服务](#如何快速启动Dubbo服务)

[多协议支持](#多协议支持)

[多注册中心支持](#多注册中心支持)



### 为什么需要Dubbo ?

服务拆分之后，在单个服务的集群模式下，首先需要考虑如下问题：

1. 地址维护；
2. 负载均衡；
3. 限流/ 容错/ 降级；
4. 监控；

传统的HTTP无法完成服务的治理（负载，容错，降级）- 服务多了，就需要去治理；



### Dubbo 的架构？

- Dubbo如何解决地址问题？

  Dubbo通过zk来注册地址信息（ip/port）；

- Dubbo 如何完成监控？

  Dubbo里面的Monitor可以监控调用信息；



Provider : 服务端；

Consumer：调放方；

Container : 服务端需要将服务部署到Container中；

Registroy : 注册中心;

Monitor : 监控中心;





