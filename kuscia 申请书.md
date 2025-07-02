# Kuscia 基于带宽调度实现

## 项目背景
在多方安全计算（MPC）领域，带宽资源至关重要，对于很多任务来说带宽直接决定着任务执行效率，对于参与任务的多方来说带宽也是一种非常昂贵的资源，但当前MPC任务的调度主要是基于CPU、Memory，缺少基于带宽的调度策略。

Kuscia（Kubernetes-based Secure Collaborative InfrA）是一款基于 K3s 的轻量级隐私计算任务编排框架，旨在屏蔽异构基础设施和协议，并提供统一的隐私计算底座。

本课题需要在 Kuscia （基于k8s） 中新增一种调度策略， 当前 k8s 已有基于 CPU 、Mem、Storage Request 的调度， 现希望增加一种基于 Bandwidth 的调度， 比如希望每一方（节点）可设置一个 Bandwidth 的 Capacity， 发起的 KusciaTask 可设置一个 Bandwidth 的 Request ， Kuscia 可以根据节点 Bandwidth Capacity 与 Task Bandwidth Request 将 Task 调度到满足要求的 Node上，如无满足要求的Node 则 Task 进入 Pending 状态。

## 技术点

## 项目实施方案

从 issue 上 的描述，可以知道，项目的实施方案大概包含以下几个步骤，其中每个步骤我都编写了附带的实现思路。

### 在节点资源中新增带宽 Capacity 字段
- 说明: 扩展节点资源定义，新增带宽容量（Bandwidth Capacity）字段。每个节点可配置其支持的最大带宽容量（单位可选：Mbps或Gbps）。
- 参考实施细节：
    - 修改Kubernetes或Kuscia节点描述数据结构，新增bandwidthCapacity字段。
    - 通过配置文件或API，支持用户为节点设置带宽容量。
    - 确定单位以及取值范围，确保兼容性。

- 实现细节：

1. 在 CapacityCfg 中新增 BandwidthCapacity 字段

```go
type CapacityCfg struct {
	CPU               string `yaml:"cpu"`
	Memory            string `yaml:"memory"`
	Pods              string `yaml:"pods"`
	Storage           string `yaml:"storage"`
	EphemeralStorage  string `yaml:"ephemeralStorage"`
	BandwidthCapacity string `yaml:"bandwidthCapacity"`
}
```

2. 在CapacityManager中新增BandwidthAvailable 和 BandwidthTotal 字段，提供两种方式来获取配置，一种是通过配置文件来获取，另一种是当 localCapacity 为 true 时，从 os 中获取

### 在 KusciaTask 新增带宽 Request 字段
- 说明: 扩展Kuscia任务定义，新增带宽请求（Bandwidth Request）字段。任务发起时可指定所需的带宽资源。
- 参考实施细节：
    - 修改Kuscia任务描述数据结构，新增bandwidthRequest字段。
    - 通过API或任务配置文件，支持用户为任务指定带宽请求值。
    - 确保带宽请求值不能大于节点的最大容量。
- 实现细节：

1. 扩展任务定义：这里经过我的观察，实际上不需要对 crd 进行修改，这是因为 crd 中有一个字段resources，其中添加bandwidth字段即可，如下所示：
```yaml
resources:
  requests:
    bandwidth: 500Mi  # Request 500 Mbps bandwidth
  limits:
    bandwidth: 1Gi    # Limit bandwidth to 1 Gbps
```

### 调度器扩展逻辑：基于带宽调度
- 说明: 修改Kuscia的调度器逻辑，将带宽Capacity与带宽Request纳入调度决策，确保任务仅被调度到满足带宽要求的节点上。
- 参考实施细节：
    - 修改调度器中的资源过滤逻辑，新增带宽过滤条件。
    - 当节点的剩余带宽容量低于任务带宽请求时，排除该节点。
    - 若所有节点均不能满足带宽要求，则将任务置为Pending状态，等待资源释放或重新调度。

### 资源动态更新：带宽使用管理
- 说明: 在任务运行时动态更新节点的带宽使用情况，确保调度精确且数据实时。
- 参考实施细节：
    - 增加对任务实时带宽使用情况的监控，定期更新节点的剩余带宽容量。
    - 在任务完成后释放带宽资源，确保资源可供后续任务使用。

### 错误处理与状态反馈
- 说明: 在任务无法调度时提供明确的状态和反馈信息，便于用户理解任务Pending的原因。
- 参考实施细节：
    - 定义新的Pending原因字段，指明因为带宽不足导致任务无法调度。
    - 在任务调度日志或事件系统中详细记录导致Pending的具体原因。

### API扩展
- 说明: 扩展现有Kuscia API，支持节点带宽Capacity的设置，任务带宽Request的定义，以及带宽相关的查询。
- 实施细节：
    - 修改节点API，支持bandwidthCapacity字段的读写。
    - 修改任务API，支持bandwidthRequest字段的定义及查询。
    - 提供带宽调度结果查询的接口，使用户可以看到具体的调度策略及资源评估结果。

### 监控与可视化支持
- 说明: 在现有监控和可视化工具中集成带宽相关指标，便于用户跟踪资源利用情况。
- 参考实施细节：
    - 在资源监控系统中增加对节点带宽Capacity和当前使用状态的监控。
    - 可视化带宽使用情况，提供剩余带宽资源等关键指标的查看功能。

### 文档更新
- 说明: 更新相关文档，包含API定义、任务配置说明、节点资源设置指南，以及带宽调度策略的流程说明。
- 参考实施细节：
    - 更新README文档，添加带宽调度功能说明及使用方法。
    - 编写新增字段和功能的详细开发设计文档。
    - 提供用户操作指南和FAQ。

## 对社区的贡献

## 个人简介

## 项目实施规划