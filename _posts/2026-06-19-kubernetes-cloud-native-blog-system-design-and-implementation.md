---
title: 基于 Kubernetes 的云原生博客系统设计与实现：从博客功能实现到云原生部署实践
date: 2026-06-19 21:30:00 +0800
categories: [Kubernetes, 云原生]
tags: [Kubernetes, K8s, 云原生, 博客系统, Nginx, Node.js, MySQL, Ingress, PV, PVC, Prometheus]
media_subpath: /assets/img/posts/k8s-cloud-native-blog/
---

## 前言

最近在已有 Kubernetes 集群的基础上，我动手部署并扩展了一个云原生博客系统。这个项目一开始并不是完全从零设计的，我参考了 [0xReLogic/kubernetes-for-students](https://github.com/0xReLogic/kubernetes-for-students) 中 Chapter 10 的 Personal Blog Platform 示例，但最终落地时已经不再是一个简单的教学演示，而是一个更贴近真实集群环境的多组件博客系统。

我将最终整理好的 YAML 文件公开在这个仓库中：

- [Sunshine-skyy/k8s-cloud-native-blog](https://github.com/Sunshine-skyy/k8s-cloud-native-blog)

如果你想直接获取本文涉及的 YAML 配置内容，可以直接到上面的仓库中查看。

这篇文章主要想做三件事：

1. 介绍这个博客系统的整体架构和实现思路。
2. 梳理我在 Kubernetes 集群中实际部署它的完整过程。
3. 总结我在部署过程中遇到的问题、原因和解决方式，方便后续复现。

## 一、项目背景

这个项目的目标并不只是“把一个网页跑起来”，而是尽量把 Kubernetes 课程中学到的几个关键对象串起来，形成一个完整的小型云原生应用。

我希望这个博客系统至少具备下面几类能力：

- 前后端分离
- 数据持久化
- 集群内部服务发现
- 对外访问入口
- 基础的健康检查
- 基础的可观测性能力

在功能层面，我也不满足于最简单的文章新增和查询，而是继续扩展了：

- 分页
- 搜索
- 分类
- 删除
- 健康检查
- 就绪检查
- Prometheus 指标暴露

因此，这个项目最终更适合被理解为一个“基于 Kubernetes 的博客系统部署实践”，而不是一个仅用于演示的最小示例。

## 二、项目来源与改造方向

这个项目参考了 [0xReLogic/kubernetes-for-students](https://github.com/0xReLogic/kubernetes-for-students) 的博客 capstone 示例。

原项目的特点主要是：

- 面向学生学习 Kubernetes
- 使用 `minikube`
- 使用博客项目来串联 Deployment、Service、ConfigMap、Secret、PVC、Ingress 等对象
- 更偏向教学示例，而不是完整功能系统

我在它的基础上主要做了这些改造：

- 将部署环境从 `minikube` 调整到真实 Kubernetes 集群
- 适配我当前使用的 `kubectl v1.18.20`
- 手动创建 MySQL 的 PV/PVC
- 补充删除、搜索、分类、分页等功能
- 增加 `/health`、`/ready`、`/metrics`
- 重做前端页面，使其更适合展示“云原生博客系统”的完整性
- 增加状态检查逻辑和基础监控展示

所以，这个项目和原始示例之间的关系更准确地说是“借鉴与扩展”，而不是简单照搬。

## 三、系统架构设计

整个系统采用三层结构：

- 前端：Nginx
- 后端：Node.js + Express
- 数据库：MySQL 8.0.41

在 Kubernetes 中，对应关系如下：

- `blog-frontend`：前端 Deployment，负责页面展示
- `blog-api`：后端 Deployment，提供 REST API
- `blog-mysql`：数据库 Deployment，持久化保存文章数据
- `Service`：负责前后端、后端和数据库之间的服务发现
- `ConfigMap`：负责注入后端配置
- `Secret`：负责注入数据库用户名密码
- `PersistentVolume / PersistentVolumeClaim`：负责 MySQL 数据持久化
- `Ingress`：作为可选七层入口
- `NodePort`：作为实际可访问入口

可以用一句话概括访问链路：

`浏览器 -> Nginx 前端 -> Blog API -> MySQL`

除此之外，我还为后端增加了：

- `/health`：健康检查接口
- `/ready`：就绪检查接口
- `/metrics`：指标暴露接口

这使得它不只是一个“能访问”的博客系统，也开始具备基础的云原生运行特征。

## 四、最终实现的核心功能

结合最终的 YAML 和内嵌代码实现，目前系统已经具备以下功能：

- 发布文章
- 查询文章列表
- 分页查看文章
- 按关键词搜索文章
- 按分类筛选文章
- 删除指定文章
- 切换文章摘要和全文
- 查看 API 健康状态
- 查看就绪状态
- 查看 Metrics 暴露状态

这也是我认为这个项目比原始示例更适合公开出来的原因之一：它已经不再只是“把 Kubernetes 对象连起来”，而是形成了一个相对完整的小型应用。

## 五、部署前的准备工作

在正式部署之前，需要先确认自己的环境满足这些条件：

### 1. 已经有可用的 Kubernetes 集群

本文默认你已经搭建好一个可以正常使用的 Kubernetes 集群，而不是从零开始安装集群。

如果你的集群还没有搭好，我后续也准备单独整理一篇关于 Kubernetes 集群搭建的文章。

### 2. 确认 Kubernetes 版本

我的环境使用的是：

- `kubectl v1.18.20`

这个版本比较旧，因此有些 YAML 写法会和新版本 Kubernetes 不一样，最明显的地方就是 Ingress。

可以先检查 API 兼容性：

```bash
kubectl api-versions | grep networking
```

如果能看到和 `networking.k8s.io` 相关的输出，说明集群中存在对应网络 API；如果能看到 `networking.k8s.io/v1beta1`，通常说明当前环境可以直接使用本文里的 Ingress 写法。

### 3. 检查存储能力

因为 MySQL 需要持久化数据，所以最好先查看集群里是否有可用的 `StorageClass`：

```bash
kubectl get storageclass
```

如果能看到已经存在的存储类，说明集群具备一定的存储能力；如果没有合适的动态存储类，也可以像我这样手动创建 PV/PVC。

### 4. 检查 Ingress Controller

如果你希望后续通过域名访问，可以先检查是否安装过 Ingress Controller：

```bash
kubectl get pods -n ingress-nginx
```

如果能看到 Ingress Controller 相关 Pod，并且状态为 `Running`，说明控制器已经可以使用；如果没有安装，也可以先完成系统其他部分的部署，后续再补充 Ingress。

## 六、MySQL 持久化存储设计

数据库是整个博客系统里最需要保证数据不丢失的部分，因此我首先解决的是存储问题。

### 1. 在目标节点上准备数据目录

因为我采用的是 `hostPath + PV/PVC` 的方式，所以需要先选定一个节点来保存 MySQL 数据。

例如：

```bash
kubectl get nodes
```

确认节点后，在目标节点上创建目录：

```bash
ssh k8s-worker1
sudo mkdir -p /data/blog-mysql
sudo chmod 777 /data/blog-mysql
exit
```

这里的 `777` 主要是为了简化实验环境下的权限问题。正式生产环境不建议这么做，应该使用更细粒度的权限控制。

### 2. 创建 PV 和 PVC

我的 `mysql-pv.yaml` 中不仅创建了 PVC，还额外创建了一个手动 PV，并通过 `nodeAffinity` 指定它绑定到固定节点。

这样做的好处是：

- MySQL 数据不会随着 Pod 重建而丢失
- 数据存储位置明确
- 更接近真实集群中的持久化思路

### 3. 创建命名空间并应用配置

```bash
kubectl create namespace blog-platform
kubectl apply -f mysql-pv.yaml
```

后续可以通过下面命令检查 PVC 是否正常绑定：

```bash
kubectl get pvc -n blog-platform
kubectl get pv
```

如果 PVC 和 PV 状态都是 `Bound`，说明持久化存储已经准备好了；如果状态仍然是 `Pending`，一般需要继续检查节点目录、标签匹配关系和 namespace 是否正确。

## 七、数据库层部署

数据库层涉及两个文件：

- `mysql-secret.yaml`
- `mysql-deployment.yaml`

### 1. Secret

我使用 `Secret` 注入了以下信息：

- root-password
- database
- user
- password

这样数据库敏感配置就不会直接硬编码在后端应用中。

### 2. MySQL Deployment

MySQL 使用 `mysql:8.0.41` 镜像，挂载到 `/var/lib/mysql`，并配置了：

- `livenessProbe`
- `readinessProbe`

其中就绪探针会在数据库可以接受查询之后才将 Pod 标记为可用，这对于后续后端连接数据库很重要。

部署命令如下：

```bash
kubectl apply -f mysql-secret.yaml -n blog-platform
kubectl apply -f mysql-deployment.yaml -n blog-platform
```

可以通过以下命令检查运行情况：

```bash
kubectl get pods -n blog-platform
kubectl describe svc blog-mysql -n blog-platform
```

正确结果应该是：

- MySQL Pod 状态为 `Running`
- `blog-mysql` Service 已正常创建
- `kubectl describe svc blog-mysql -n blog-platform` 输出中的 `Selector` 能匹配到 MySQL Pod 的标签

这里还需要特别注意 `Service` 的 selector 必须和 MySQL Pod 的 label 一致，否则后端即使能解析服务名，也无法正确访问数据库。

## 八、后端 API 设计与部署

后端是整个系统最核心的业务层，对应文件包括：

- `api-configmap.yaml`
- `api-deployment.yaml`

### 1. ConfigMap

后端通过 `ConfigMap` 注入这些配置：

- `DB_HOST`
- `DB_PORT`
- `API_PORT`

### 2. API 实现方式

后端使用 `node:16-alpine` 镜像，在容器启动时动态创建并运行一个 `Express` 应用。

后端逻辑主要包括：

- 初始化 MySQL 连接池
- 自动创建 `posts` 表
- 自动兼容 `category` 字段
- 提供 `/posts`、`/posts/search`、`/posts/:id` 等接口
- 提供 `/health`、`/ready`、`/metrics`

### 3. 为什么我没有直接把后端代码单独做成镜像

这是因为当前项目更偏向“使用 YAML 快速复现和教学展示”，所以我把后端脚本直接写进了 Deployment 中。

这种做法的优点是：

- 复现简单
- 不需要额外构建自定义镜像
- 便于课程和实验环境快速落地

缺点也很明显：

- 工程化程度有限
- 不适合生产环境
- 后端代码和部署配置耦合较深

如果后续继续完善，这部分最值得进一步重构成独立应用源码仓库。

### 4. 后端部署

```bash
kubectl apply -f api-configmap.yaml -n blog-platform
kubectl apply -f api-deployment.yaml -n blog-platform
```

部署完成后，可以检查：

```bash
kubectl get pods -n blog-platform
kubectl logs -n blog-platform -l tier=backend
```

正确结果应该是：

- `blog-api` 的两个副本都能正常启动
- Pod 状态为 `Running`
- 日志中可以看到数据库初始化成功、服务监听成功等信息

我这里将后端副本数设置为 `2`，这样能够体现 Deployment 多副本管理的能力。

## 九、前端页面设计与部署

前端对应文件是：

- `frontend-deployment.yaml`

它的特点是：前端页面和 Nginx 配置同样是动态写入容器的，而不是从外部镜像仓库中拉一个完整前端项目。

### 1. 前端做了哪些事情

前端不仅仅是一个文章展示页，而是一个具有“系统展示”性质的单页应用，包含：

- 首页概览
- 系统状态区
- 文章管理区
- 搜索与分类筛选
- 删除弹窗
- Metrics 展示说明

页面里还通过 `/api/health`、`/api/ready`、`/api/metrics` 实时检查后端状态，用于体现这个项目的云原生特征。

### 2. 前端和后端如何通信

在 Nginx 配置中，我将：

```nginx
location /api/ {
  proxy_pass http://blog-api:3000/;
}
```

这样浏览器请求 `/api/...` 时，就会转发给后端服务 `blog-api`。

这也是为什么前端页面中可以直接用：

```javascript
const API_URL = '/api';
```

### 3. 前端部署

```bash
kubectl apply -f frontend-deployment.yaml -n blog-platform
```

前端副本数同样设置为 `2`，对应的 `Service` 类型为：

```yaml
type: NodePort
nodePort: 30080
```

因此部署完成后，可以通过：

```text
http://<NodeIP>:30080
```

访问博客系统。

正确结果应该是：

- 前端 Pod 状态为 `Running`
- `blog-frontend` Service 已创建成功
- 浏览器访问 `http://<NodeIP>:30080` 时可以正常打开博客首页

## 十、Ingress 配置

如果你的集群里已经安装了 Ingress Controller，还可以继续部署：

```bash
kubectl apply -f blog-ingress.yaml -n blog-platform
```

我的 Ingress 配置使用的是：

- `networking.k8s.io/v1beta1`
- host：`blog.local`

如果 Ingress Controller 已经正常运行，并且本地已经做好 hosts 映射或 DNS 解析，就可以通过下面的地址访问：

```text
http://blog.local
```

这里必须特别说明一下：这不是新版本 Kubernetes 推荐的写法，而是因为我的环境是 `kubectl v1.18.20`，所以需要使用旧版本 API。

这也是这个项目在公开时必须写清楚的兼容性信息之一。

## 十一、推荐的部署顺序

如果从头开始复现，我建议按下面顺序执行：

```bash
kubectl create namespace blog-platform
kubectl apply -f mysql-pv.yaml
kubectl apply -f mysql-secret.yaml -n blog-platform
kubectl apply -f mysql-deployment.yaml -n blog-platform
kubectl apply -f api-configmap.yaml -n blog-platform
kubectl apply -f api-deployment.yaml -n blog-platform
kubectl apply -f frontend-deployment.yaml -n blog-platform
kubectl apply -f blog-ingress.yaml -n blog-platform
```

部署完成后，可以用下面命令继续检查：

```bash
kubectl get ingress -n blog-platform
```

正确结果应该是可以看到 `blog-ingress` 被成功创建，并且 host 为 `blog.local`。

## 十二、分类功能对应的数据库修改

由于添加分类功能会涉及数据库表结构修改，这部分更适合放在系统正式运行和功能验证之前说明。如果数据库中的 `posts` 表还没有 `category` 列，那么分类功能将无法正常工作。

### 1. 找到 MySQL Pod 名称

先执行：

```bash
kubectl get pods -n blog-platform -l tier=database
```

正确结果应该是可以看到一个 MySQL Pod，状态通常为 `Running`，例如：

```text
blog-mysql-xxxxxx-xxxxx   1/1   Running
```

### 2. 进入 Pod 并打开 MySQL 客户端

拿到 Pod 名称后执行：

```bash
kubectl exec -it <mysql-pod-name> -n blog-platform -- mysql -uroot -p
```

这里输入的密码就是 `mysql-secret.yaml` 中定义的 `root-password`，默认是：

```text
RootPass123
```

### 3. 检查当前表结构

进入 MySQL 后执行：

```sql
USE blog_db;
SHOW TABLES;
DESCRIBE posts;
```

如果当前表结构中只有：

- `id`
- `title`
- `content`
- `author`
- `created_at`

说明还没有添加分类字段。

### 4. 添加 category 列

推荐使用 `ALTER TABLE`，这样可以保留已有数据：

```sql
ALTER TABLE posts ADD COLUMN category VARCHAR(100) NOT NULL DEFAULT 'general';
```

执行完成后再次检查：

```sql
DESCRIBE posts;
```

正确结果应该是可以看到新增的 `category` 列。

### 5. 退出 MySQL

```sql
EXIT;
```

### 6. 数据备份（可选但推荐）

如果想更稳妥一些，可以在修改前先导出当前数据：

```bash
kubectl exec -it <mysql-pod-name> -n blog-platform -- mysqldump -uroot -pRootPass123 blog_db > backup.sql
```

执行成功后，会在当前本地目录中生成一个 `backup.sql` 文件。

### 7. 补充说明

当前后端代码里其实已经加入了分类字段兼容逻辑，启动时会尝试执行：

```sql
ALTER TABLE posts ADD COLUMN category VARCHAR(100) NOT NULL DEFAULT 'general';
```

如果字段已经存在，就忽略重复添加错误。

不过从部署和排错角度来看，手动掌握这一步仍然非常重要，因为它直接关系到分类功能能否正常运行，也更有助于理解后端功能和数据库结构之间的关系。

## 十三、页面截图展示


### 1. 系统主页截图


![系统主页截图](/assets/img/posts/k8s-cloud-native-blog/home-page.png)


### 2. 文章管理页面截图



![文章管理页面截图](/assets/img/posts/k8s-cloud-native-blog/manage-page.png)


### 3. 系统状态页面截图



![系统状态页面截图](/assets/img/posts/k8s-cloud-native-blog/status-page.png)


## 十四、部署完成后的验证方式

系统部署完成后，我通常会从下面几个方面验证：

### 1. 基础资源状态

```bash
kubectl get pods -n blog-platform
kubectl get svc -n blog-platform
kubectl get pvc -n blog-platform
kubectl get pv
```

### 2. 前端访问

在浏览器中访问：

```text
http://<NodeIP>:30080
```

如果已经配置了 Ingress，也可以访问：

```text
http://blog.local
```

### 3. 功能验证

重点测试：

- 能否发布文章
- 能否查看文章
- 分页是否正常
- 搜索是否正常
- 分类是否正常
- 删除是否正常

### 4. 状态接口验证

还可以手动验证：

```text
/api/health
/api/ready
/api/metrics
```

如果这些接口可用，说明后端运行状态和可观测性入口都已经基本正常。

## 十五、遇到的问题与解决方式

这一部分是整个项目里我认为最有价值的地方，因为真实部署和只看教程最大的差别，就在这里。

### 1. PVC 无法立即正常绑定

#### 现象

在还没有整理好 namespace 或 PV/PVC 依赖关系时，存储对象无法按预期工作。

#### 原因

- PVC 所在 namespace 还没有创建
- 手动 PV 和 PVC 的创建顺序没有理清
- label 与 selector 对不上

#### 解决方式

- 先创建 `blog-platform` namespace
- 再创建 PV/PVC
- 检查 `matchLabels` 是否一致

实际操作中，先把 namespace 建好再统一 apply，问题会少很多。

### 2. Ingress Controller 不可用

#### 现象

集群中没有安装 Ingress Controller，或者安装后 Pod 没有正常 Running。

#### 原因

- 集群初始状态没有 Ingress Controller
- 控制器镜像拉取失败

#### 解决方式

先检查：

```bash
kubectl get pods -n ingress-nginx
```

如果镜像拉不下来，可以先手动拉取对应镜像，再让 Kubernetes 复用本地镜像。

另外，在实验环境里，NodePort 和 Ingress 都可以保留：前者适合直接访问验证，后者适合基于域名进行访问。

### 3. Service selector 与 Pod label 不匹配

#### 现象

Service 已经创建成功，但前端访问后端或后端访问数据库依旧异常。

#### 原因

Kubernetes Service 能否转发流量，核心依赖 selector 是否选中了正确的 Pod。

例如 MySQL 的 Service：

- selector：`app: blog`、`tier: database`

如果 Pod 没有这组标签，Service 就相当于“没有后端”。

#### 解决方式

检查：

```bash
kubectl get pods -n blog-platform --show-labels
kubectl describe svc blog-mysql -n blog-platform
```

逐项对比 selector 和 labels。

### 4. 旧版本 Kubernetes API 兼容性问题

#### 现象

某些网上的 YAML 示例在本地集群中无法直接使用。

#### 原因

我的集群使用的是 `kubectl v1.18.20`，而很多新教程默认使用的是更新版本的 API。

#### 解决方式

在公开项目和博客中明确标注版本，并使用适配旧版本的写法，例如：

- `Ingress` 使用 `networking.k8s.io/v1beta1`

这类兼容性说明必须提前告诉读者，否则别人很容易觉得“照着做却跑不起来”。

### 5. 后端镜像版本和脚本实现需要适配

#### 现象

原始教学示例使用的是 `node:18-alpine`，但在我实际整理和适配过程中，最终改成了 `node:16-alpine`。

#### 原因

实验环境、依赖安装方式和整体稳定性需要综合考虑。

#### 解决方式

根据自己实际集群与运行情况选择最终可稳定工作的镜像版本，而不是机械照抄示例。

### 6. 前端能打开，但功能不完整

#### 现象

系统页面打开之后，并不意味着整个系统已经完全可用。

#### 原因

前端只是入口页面，真正的功能依赖后端 API 和数据库都要正常。

#### 解决方式

除了看页面是否能打开，还要继续验证：

- `/api/health`
- `/api/ready`
- `/api/metrics`
- 文章 CRUD 是否正常

这也是我为什么在前端页面里加入状态看板的原因之一。

## 十六、总结

这个博客系统项目让我对 Kubernetes 的理解不再停留在“知道有哪些对象”，而是开始真正理解：

- Deployment 如何管理副本
- Service 如何完成服务发现
- ConfigMap 和 Secret 如何解耦配置
- PV/PVC 如何为数据库提供持久化能力
- NodePort 和 Ingress 如何承担访问入口
- 健康检查、就绪检查和 metrics 如何体现云原生应用的运行特征

如果只是看教程，很多内容看上去都很顺；但当你真正开始在集群里部署时，PV、selector、Ingress、版本兼容、探针和访问路径这些问题会一个个冒出来。也正因为这样，这个项目才真正让我把 Kubernetes 的基础对象串成了一套完整实践。

后续如果继续完善，我还想继续做两件事：

- 单独整理一篇 Kubernetes 集群搭建文章
- 将当前这种“YAML 内嵌代码”的方式继续重构成更工程化的前后端项目结构

如果你也在做类似的 Kubernetes 部署练习，希望这篇文章和上面的仓库能帮你少踩一些坑。
