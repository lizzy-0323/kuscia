# Kuscia 基于带宽调度实现

## 项目简介

- 项目名称：Kuscia 基于带宽调度实现
- 项目主导师：高永龙
- 申请人：李子毅
- 日期：2025 年 7 月 2 日
- 邮箱：<liziyi0323xxx@gmail.com>

## 项目背景

在多方安全计算（MPC）领域，带宽资源至关重要，对于很多任务来说带宽直接决定着任务执行效率，对于参与任务的多方来说带宽也是一种非常昂贵的资源，但当前MPC任务的调度主要是基于CPU、Memory，缺少基于带宽的调度策略。

Kuscia（Kubernetes-based Secure Collaborative InfrA）是一款基于 K3s 的轻量级隐私计算任务编排框架，旨在屏蔽异构基础设施和协议，并提供统一的隐私计算底座。

本课题需要在 Kuscia（基于k8s）中新增一种调度策略，当前 k8s已有基于 CPU 、Mem、Storage Request 的调度，现希望增加一种基于 Bandwidth的调度，比如希望每一方（节点）可设置一个 Bandwidth 的 Capacity，发起的 KusciaTask 可设置一个 Bandwidth 的 Request，Kuscia 可以根据节点 Bandwidth Capacity 与 Task Bandwidth Request 将 Task 调度到满足要求的Node上，如无满足要求的 Node 则 Task 进入 Pending 状态。

## 项目实施方案

从 issue 上 的描述，可以知道，项目的实施方案大概包含以下几个步骤，其中，每个步骤都编写了附带的实现思路，可作为初步参考。

### 在节点资源中新增带宽 Capacity 字段

- 说明: 扩展节点资源定义，新增带宽容量（Bandwidth Capacity）字段。每个节点可配置其支持的最大带宽容量（单位可选：Mbps或Gbps）。
- 参考实施细节：
  - 修改Kubernetes或Kuscia节点描述数据结构，新增bandwidthCapacity字段。
  - 通过配置文件或API，支持用户为节点设置带宽容量。
  - 确定单位以及取值范围，确保兼容性。

- **实现思路**：

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

2. 在CapacityManager中新增BandwidthAvailable 和 BandwidthTotal 字段，提供两种方式来获取配置，一种是通过配置文件来获取，另一种是当 localCapacity 为 true 时，从网卡中获取

### 在 KusciaTask 新增带宽 Request 字段

- **说明**: 扩展Kuscia任务定义，新增带宽请求（Bandwidth Request）字段。任务发起时可指定所需的带宽资源。
- **参考实施细节**：
  - 修改Kuscia任务描述数据结构，新增bandwidthRequest字段。
  - 通过API或任务配置文件，支持用户为任务指定带宽请求值。
  - 确保带宽请求值不能大于节点的最大容量。
- **实现思路**：

1. 扩展任务定义： 在kusciatask_types.go 中新增bandwidthRequest字段，如下所示：

```go
type KusciaTaskSpec struct {
    Initiator       string `json:"initiator"`
    TaskInputConfig string `json:"taskInputConfig"`
    // +optional
    ScheduleConfig ScheduleConfig `json:"scheduleConfig,omitempty"`
    Parties        []PartyInfo    `json:"parties"`
    // +optional
    BandwidthRequest []uint64 `json:"bandwidthRequest,omitempty"`
}
```

2. 然后运行make generate manifest，生成对应的crd yaml，deepcopy等

### 调度器扩展逻辑：基于带宽调度

- **说明**: 修改Kuscia的调度器逻辑，将带宽Capacity与带宽Request纳入调度决策，确保任务仅被调度到满足带宽要求的节点上。
- **参考实施细节**：
  - 修改调度器中的资源过滤逻辑，新增带宽过滤条件。
  - 当节点的剩余带宽容量低于任务带宽请求时，排除该节点。
  - 若所有节点均不能满足带宽要求，则将任务置为Pending状态，等待资源释放或重新调度。

- **实现思路**：

1. 在`TaskResourceSpec`中添加BandwidthRequest字段，对齐kusciatask_types.go中的BandwidthRequest字段
2. 在`kusciascheduling.go`中对PreFilter方法进行扩展，以考虑带宽要求，如下所示：

```go
func (cs *KusciaScheduling) PreFilter(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (*framework.PreFilterResult, *framework.Status) {
    // 现有逻辑...
    
    // Check bandwidth requirements
    if err := cs.trMgr.CheckBandwidthRequirements(ctx, pod); err != nil {
        return nil, framework.NewStatus(framework.UnschedulableAndUnresolvable, err.Error())
    }
    
    return nil, framework.NewStatus(framework.Success, "")
}
```

同时，在task_resource_manager.go中实现CheckBandwidthRequirements方法，kusciascheduling.go中修改Reserve方法，如下所示：

```go
func (f *KusciaScheduling) Reserve(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
    // 现有逻辑...
    
    // 预留带宽资源
    bandwidthRequest := extractBandwidthRequest(pod)
    if !bandwidthRequest.IsZero() {
        if err := f.capacityManager.ReserveBandwidth(bandwidthRequest); err != nil {
            return framework.NewStatus(framework.Error, fmt.Sprintf("Failed to reserve bandwidth: %v", err))
        }
    }
    
    return nil
}

func (f *KusciaScheduling) Unreserve(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) {
    // 释放预留的带宽资源
    bandwidthRequest := extractBandwidthRequest(pod)
    if !bandwidthRequest.IsZero() {
        f.capacityManager.ReleaseBandwidth(bandwidthRequest)
    }
}


func (trMgr *TaskResourceManager) CheckBandwidthRequirements(ctx context.Context, pod *v1.Pod) error {
    if err := trMgr.checkBandwidthRequirements(ctx, pod); err != nil {
        return err
    }
    
    return nil
}

// ReserveBandwidth reserves bandwidth for the pod on the node
func (m *taskResourceManager) ReserveBandwidth(ctx context.Context, pod *v1.Pod, nodeName string) error {
    // 实现带宽预留逻辑
    return nil
}

// ReleaseBandwidth releases bandwidth when pod completes or fails
func (m *taskResourceManager) ReleaseBandwidth(ctx context.Context, pod *v1.Pod) {
    // 实现带宽释放逻辑
}
```

### 资源动态更新：带宽使用管理

- **说明**: 在任务运行时动态更新节点的带宽使用情况，确保调度精确且数据实时。
- **参考实施细节**：
  - 增加对任务实时带宽使用情况的监控，定期更新节点的剩余带宽容量。
  - 在任务完成后释放带宽资源，确保资源可供后续任务使用。
- **实现思路**：

1. 修改capacity_manager.go，添加带宽使用情况的更新

```go
func (pa *CapacityManager) UpdateBandwidthAvailable(used resource.Quantity) {
 // 计算剩余带宽 = 总带宽 - 已使用带宽
 available := pa.bandwidthTotal.DeepCopy()
 available.Sub(used)

 // 确保可用带宽不为负数
 if available.Sign() < 0 {
  available = resource.NewQuantity(0, resource.DecimalSI)
 }

 pa.bandwidthAvailable = *available
}

// GetBandwidthTotal returns the total bandwidth capacity
func (pa *CapacityManager) GetBandwidthTotal() resource.Quantity {
 return pa.bandwidthTotal.DeepCopy()
}

// GetBandwidthAvailable returns the available bandwidth capacity
func (pa *CapacityManager) GetBandwidthAvailable() resource.Quantity {
 return pa.bandwidthAvailable.DeepCopy()
}

// SetBandwidthCapacity updates the bandwidth capacity configuration
func (pa *CapacityManager) SetBandwidthCapacity(capacity resource.Quantity) {
 pa.bandwidthTotal = capacity.DeepCopy()
 pa.bandwidthAvailable = capacity.DeepCopy()
}
```

### 错误处理与状态反馈

- **说明**: 在任务无法调度时提供明确的状态和反馈信息，便于用户理解任务Pending的原因。
- **参考实施细节**：
  - 定义新的Pending原因字段，指明因为带宽不足导致任务无法调度。
  - 在任务调度日志或事件系统中详细记录导致Pending的具体原因。
- **实现思路**：

1. 在kusciaTaskStatus中添加带宽不足的错误状态

```go
const (
    // ... 现有常量保持不变
    
    // TaskBandwidthInsufficientReason 表示因带宽不足导致任务无法调度
    TaskBandwidthInsufficientReason = "BandwidthInsufficient"
)
```

2. 在preFilter和Reserve失败时添加带宽不足的错误信息：

```go
func (trMgr *TaskResourceManager) patchTaskResource(phase kusciaapisv1alpha1.TaskResourcePhase, condType kusciaapisv1alpha1.TaskResourceConditionType, reason string, tr *kusciaapisv1alpha1.TaskResource) error {
    // ... 现有代码保持不变
    
    // 如果是带宽不足导致的失败，添加特定的错误信息
    if strings.Contains(reason, "bandwidth") {
        reason = fmt.Sprintf("%s: %s", kusciaapisv1alpha1.TaskBandwidthInsufficientReason, reason)
    }
    
    // ... 其余代码保持不变
}
```

### API扩展

- **说明**: 扩展现有Kuscia API，支持节点带宽Capacity的设置，任务带宽Request的定义，以及带宽相关的查询。
- **实施细节**：
  - 修改节点API，支持bandwidthCapacity字段的读写。
  - 修改任务API，支持bandwidthRequest字段的定义及查询。
  - 提供带宽调度结果查询的接口，使用户可以看到具体的调度策略及资源评估结果。
- **实现思路**：

1. 扩展`domain.proto`，如下所示：

```proto
// Node resource capacity information
message NodeResourceCapacity {
  string cpu_capacity = 1;
  string memory_capacity = 2;
  string storage_capacity = 3;
  string bandwidth_capacity = 4;
  string cpu_available = 5;
  string memory_available = 6;
  string storage_available = 7;
  string bandwidth_available = 8;
}
```

2. 扩展`config.proto`，添加带宽配置相关的api

### 监控与可视化支持

- **说明**: 在现有监控和可视化工具中集成带宽相关指标，便于用户跟踪资源利用情况。
- **参考实施细节**：
  - 在资源监控系统中增加对节点带宽Capacity和当前使用状态的监控。
  - 可视化带宽使用情况，提供剩余带宽资源等关键指标的查看功能。
- **实现细节**：
  1. 在 agent/metrics/ 下暴露 Gauge：node_bandwidth_available_bits、node_bandwidth_total_bits。
  2. Grafana Dashboard 样板见 docs/monitoring/bandwidth_dashboard.json。

### 文档更新

- **说明**: 更新相关文档，包含API定义、任务配置说明、节点资源设置指南，以及带宽调度策略的流程说明。
- **参考实施细节**：
  - 更新README文档，添加带宽调度功能说明及使用方法。
  - 编写新增字段和功能的详细开发设计文档。
  - 提供用户操作指南和FAQ。
- **实现细节**：
  1. 添加示例yaml，如：

```yaml
apiVersion: kuscia.secretflow/v1alpha1
kind: TaskResource
...
spec:
  pods:
  - name: worker
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
        bandwidth: 500Mi
```

  2. 添加节点侧的capacity配置，如：

```yaml
capacity:
  cpu: "4"
  memory: "16Gi"
  storage: "200Gi"
  bandwidth: "10Gi"     # 10 Gbps
```

### 总结：基于带宽的调度工作流

1. 资源注册阶段：
节点通过Domain CRD注册自己的带宽容量
通过API或配置更新节点带宽信息

2. 任务提交阶段：
用户在KusciaTask中指定带宽需求
控制器将KusciaTask转换为TaskResource时带上带宽请求

3. 调度阶段：
PreFilter：检查带宽需求是否可满足
Reserve：预留节点上的带宽资源

4. 任务运行阶段：
周期性监控和更新节点带宽使用情况，当任务完成时释放带宽资源主要是通过capacity_manager来更新，并通过agent来上报

其中，调度器相关代码，任务运行时，节点状态更新部分的代码为主要模块

## 完成项目后，对社区的贡献

1. 完善kuscia功能，增加带宽调度策略，提交PR
2. 完善相关文档，并在学生群体、开源社区进行进一步推广

## 技术方法及可行性

技术栈上，主要是Golang、kubernetes，同时还需要了解kuscia的架构

1. 语言基础相关

- 本人进行Go语言相关开发已经有约3年的时间，熟练掌握Go语言，了解map，struct，channel，GMP模型调度，垃圾回收，并发编程等技术，并熟悉docker，k8s等技术栈，了解k8s operator机制，对k8s的架构以及各组件职能都有所了解。

2. 开源经验相关

- 曾参与2024年的开源之夏，为ob-operator项目贡献代码九千余行，因完成出色，获得蚂蚁社区颁发的年度开源之星称号，并至今积极活跃在OceanBase社区中
- 曾参与阿里开源项目higress(Star数 2.2k), 完成基于wasm插件对百川大模型服务的对接，并编写了相关的k8s配置文件和dockerfile，熟悉了k8s基本操作
- 曾参与向量检索竞赛开源项目big-ann-benchmark(Star数 300+)，该项目最终需要提供容器的配置文件并进行向量检索模型的搭建，独立完成了代码编写和测试，熟悉CI/CD工作流，以及docker基本操作
- 曾参与开源项目shifu（Star数1.3k），完成了贡献，使得项目支持在 TelemetryService 对应 CRD 提出后完成 Service 和 Deployment 的自动部署功能

3. 隐语社区相关

- 曾经为隐语SPU项目贡献了代码，为隐语社区贡献者一枚，并有官方颁发的贡献者证书
- 曾经在隐语武汉站进行演讲，题目为《从开源探索到算法突破，我的隐语贡献三部曲》，并积极为社区宣传

## 项目实施规划

1. 项目研发第一阶段（07月11日-08月03日）
[ ] 学习相关知识，深刻理解k8s调度机制等
[ ] 明确功能模块，完成设计文档的编写，明确需要编写代码的模块，并明确各模块的排期
[ ] 完成大部分模块的代码编写

2. 项目研发第二阶段（08月04日-09月26日）
[ ] 解决第一阶段遇到的问题
[ ] 完善单元测试和端到端测试
[ ] 完成文档编写，并最好可以产出博客并发布
[ ] 合并代码到主分支，此过程中可以对之前代码做一定优化
