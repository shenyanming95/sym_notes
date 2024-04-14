# 1.Kubernetes介绍

Kubernetes 是一个生产级别的容器编排平台和集群管理系统，能够创建、调度容器，监控、管理服务器。它采用“控制面 / 数据面”（Control Plane / Data Plane）架构。

一个k8s集群的节点，可以是物理机，也可以是虚拟机。少量节点用作数据面，负责集群的管理维护，这种节点叫做：Master Node；大部分节点用作数据面，跑业务运用，这种节点叫做Worker Node。

## 1.1.节点

Master 里有 4 个组件，分别是 apiserver、etcd、scheduler、controller-manager：

- apiserver：k8s系统唯一入口，对外提供RESTful API，其它组件只能和它通信；
- etcd：持久化存储系统里的各种资源对象和状态；
- scheduler：负责容器的编排工作，检查节点的资源状态，把 Pod 调度到最适合的节点上运行；
- controller-manager：负责维护容器和节点等资源的状态，实现故障检测、服务迁移、应用伸缩等功能；