在 Kubernetes 中部署和使用 Jenkins

# 实验介绍

本次实验带你在 Kubernetes 中部署 Jenkins，并完成 Jenkins 的初始化操作，然后创建一个简单的流水线测试 Jenkins 是否能正常工作，最后配置如何基于 Kubernetes 实现动态 Slave。

#### 知识点

- Jenkins 安装
- RBAC 认证
- 流水线创建
- 动态 Slave

# Jenkins 简介

Jenkins 是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。 Jenkins 前身是 Hudson，使用 java 语言开发的自动化发布工具。在中大型金融等企业中普遍使用 Jenkins 来作为项目发布工具。 Jenkins 官方提供的插件使 Jenkins 更为强大。

Jenkins 支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序。

Jenkins 的特点如下：

- 开源免费
- 多平台支持（windows/linux/macos）
- 主从分布式架构
- 提供 web 可视化配置管理页面
- 安装配置简单
- 插件资源丰富

# Jenkins 部署

Jenkins 的部署方式很多，主要如下：

- WAR 包独立部署
- RPM 部署
- Docker 部署

为了方便，这里采用 Docker 部署，并且使用 Kubernetes 编排。

在 `/home/shiyanlou/Code/devops/` 目录下创建一个 `sy-02-1` 目录，命令如下：

```bash
mkdir -p /home/shiyanlou/Code/devops/sy-02-1
cd /home/shiyanlou/Code/devops/sy-02-1
```

# 创建 PVC，为 Jenkins 提供数据持久化

在 `/home/shiyanlou/Code/devops/sy-02-1` 目录下创建一个 `jenkins-pvc.yaml` 文件，写入如下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: devops
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-hostpath
  resources:
    requests:
      storage: 5Gi
```

然后使用 `kubectl apply -f jenkins-pvc.yaml` 创建 PVC，并使用 `kubectl get pvc -n devops jenkins-pvc` 查看 PVC 状态，如下：