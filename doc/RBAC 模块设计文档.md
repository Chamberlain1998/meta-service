- # 多租户细粒度 RBAC 模块设计文档

  **版本 1.0 — 无状态架构 **

  ------

  ## 文档信息

  | 项目         | 内容                                                         |
  | :----------- | :----------------------------------------------------------- |
  | **文档名称** | 多租户细粒度 RBAC 模块设计文档                               |
  | **版本号**   | 1.0                                                          |
  | **作者**     | 纪成林                                                       |
  | **审核人**   |                                                              |
  | **生效日期** | 2026年2月xx日                                                |
  | **适用系统** | 支持多租户、函数即服务（FaaS）或动态实例化工作负载的云原生平台 |

  ------

  ![image-20260117163846925](/Users/chamberlain/Library/Application Support/typora-user-images/image-20260117163846925.png)

  ## 1. 设计目标

  本模块旨在为多租户平台提供一套**安全、可扩展、无状态**的基于角色的访问控制（RBAC）机制，满足以下核心需求：
  
  1. **基于一维字符串标签的权限分组**：通过扁平化字符串标签集合（如 `["env-prod", "team-finance"]`）实现语义化资源分组，避免对具体资源 ID 的依赖。
  2. **完全无状态鉴权服务**：权限策略与资源元数据集中存储于 etcd，鉴权组件无本地持久化状态，支持水平弹性伸缩。
  3. **最小权限原则**：权限粒度精确到 Instance 级操作，且仅可通过显式标签选择器授权。
  4. **强一致性与高可用**：依托 etcd 的线性一致性和集群容错能力，确保权限变更实时、可靠生效。

  ------

  ## 2. 核心概念定义

  ### 2.1 租户（Tenant）
  
  - 平台中的逻辑隔离单元，所有资源与策略均归属于单一租户。
  - 权限策略在租户内定义，并可在该上下文中授予其他租户。

  ### 2.2 资源类型（Resource Types）
  
  | 类型            | 说明                                                         |
  | :-------------- | :----------------------------------------------------------- |
  | **Function**    | 服务模板，由用户命名，代表一类可实例化的功能单元。           |
  | **Instance**    | 运行时实体，ID 由系统自动生成（如 UUID），不可预测，不可重用。 |
  | **Tenant**      | 租户                                                         |
  | **Role**        | 权限规则的声明式集合，由资源所属租户定义                     |
  | **RoleBinding** | 建立 Role 与 Subject 的授权关系，由role所属租户定义          |
  
  ### 2.3 操作（Verbs）
  
  Posix操作
  
  | Verb        | 语义                                |
  | :---------- | :---------------------------------- |
  | `Create`    | 在指定 Function 下创建新 Instance   |
  | `Invoke`    | 触发 Instance 执行任务              |
  | `Exit`      | 请求 Instance 强制退出 （暂未使用） |
  | `SaveState` | 保存 Instance 当前运行状态          |
  | `LoadState` | 从快照恢复 Instance 状态            |
  | `Kill`      | 向 Instance 发送信号量              |
  
  Function 支持Create权限
  
  Instance 支持Invoke，Exit，SaveState，LoadState，Kill操作：
  
  Tenant，Role，RoleBinding 作为Restful原生的资源，支持Create，Delelet，Update，Get
  
  ### 2.4 标签模型（Label Model）

  - **结构**：一维字符串集合（`[]string`），无键值对结构。
  - **示例**：`["env-prod", "app-payment", "team-billing", "critical"]`
  - **约束**：
    - 非空（至少一个标签）；
    - 创建后不可变；
    - 不支持通配符、正则或层级语义。
  
  ### 2.5 主体（Subject）
  
  当前支持租户级主体，表示整个目标租户获得授权，适用于跨租户服务调用场景。
  
  ### 2.6 角色（Role）
  
  - 权限规则的声明式集合，由资源所属租户定义。
  - **不直接绑定主体**，需通过 RoleBinding 生效。
  - **支持标签选择器**，`instance`和`function`在创建时会将自身关键信息存储到label中。
    - `function`创建时，会自动添加一个`name-{functionName}`标签。
    - `instance`创建时，会自动添加一个`name-{instanceName}`和`function-{functionName}`标签

  ### 2.7 角色绑定（RoleBinding）

  - 建立 Role 与 Subject 的授权关系，由role所属租户定义。
  - 明确指定权限作用的租户。
  
  ------
  
  ## 3. 数据模型与存储设计
  
  ### 3.1 存储后端：etcd
  
  - **唯一可信数据源**：所有策略与元数据持久化至 etcd v3 API。
  - **强一致性保障**：利用 etcd 的 linearizable read/write 保证权限变更立即全局可见。
  - **无本地状态**：RBAC 鉴权服务启动时不加载任何数据，每次请求按需读取（后续考虑改为**读写穿透**）。
  
  ### 3.2 键空间布局
  
  #### 3.2.1 RBAC 策略数据
  
  ```
  /sn/rbac/bussiness/yrk/rbac/
  ├── roles/
  │   └── {tenant-id}/{role-id}.json
  └── rolebindings/
      └── {tenant-id}/
          ├── {binding-id}.json
          └── by-subject/{subject}/{binding-id}   ← 反向索引，用于高效查询
  ```
  
  #### 3.2.2 Instance 元数据
  
  参照《MetaService 组件整体设计文档》
  
  > 🔒 RBAC 服务仅具备 `/sn/instance/bussiness/yrk/tenant/{tenant-id}` 的只读权限。
  
  ------
  
  #### 3.2.3 function 元数据
  
  参照《MetaService 组件整体设计文档》
  
  > 🔒 RBAC 服务仅具备 `/sn/function/bussiness/yrk/tenant/{tenant-id}/` 的只读权限。
  
  #### 3.2.4 tenant 元数据
  
  ```
  /sn/tenant/bussiness/yrk/{tenant-id}
  ```
  
  **内容结构**：
  
  ```json
  {
    "version": 123,
    "tenant_id": "t-abc123",
    "status": "active",
    "quotas": {
      "instanceCount": {
        "limit": 1000,
        "unit": "count",
        "is_hard": true,
      }
    },
    "usages": {
      "instanceCount": 890
    },
    "last_updated": "2026-01-16T10:15:00Z"
  }
  ```
  
  ## 4. 策略配置格式
  
  ### 4.1 Role 定义
  
  ```json
  {
    "id": "role-analytics-access",
    "tenant": "tenant-A",
    "rules": [
      {
        "resources": [
          {
            "type": "function",
            "selector": [
              "name-function-0"
              "env-prod",
              "team-public"
            ]
          }
        ],
        "verbs": [
          "Create",
        ]
      },
      {
        "resources": [
          {
            "type": "instance",
            "selector": [
              "name-instance-001"
              "function-function-0"
              "env-prod",
              "team-public"
            ]
          }
        ],
        "verbs": [
          "Invoke",
          "SaveState"
        ]
      }
    ]
  }
  ```
  
  #### 语义约束：
  
  - `resources` 中types只支持在《2.2 资源类型（Resource Types）》中声明字段；
  - `selector` 为字符串数组，匹配采用 **集合包含（subset）** 语义
  - `verbs`只支持在《2.3 操作（Verbs）》中定义的字段
  
  ### 4.2 RoleBinding 定义
  
  ```json
  {
    "id": "role-binding-analytics-access-binding",
    "role_id": "role-analytics-access",         // 引用 Role ID              
    "subject": "tenant:team-analytics"          // 被授权方      
  }
  ```

  

  ## 5. 对外接口

  ```yaml
  
  ```
  
  

  ## 6. 权限评估逻辑
  
  ### 6.1 输入
  
  - `subject`: 请求主体（如 `tenant-analytics`）
  - `context_tenant`: 目标资源所属租户（如 `tenant-platform`）
  - `resource`: 目标资源（如 `instance:inst-abc123`）
  - `verb`: 请求操作（如 `Invoke`）
  
  ### 6.2 评估流程
  
  1. **合法性校验**：验证 `verb` 是否属于Instance允许的操作。
  2. **元数据获取**：从 `/sn/instance/bussiness/yrk/tenant/{tenant-id}/function/{function-name}/version/{version}/defaultaz/{request-id}/{instance-id}` 读取 Instance 标签。
  3. **绑定查询**：通过反向索引 /serverless/v1/rbac/rolebindings/{context_tenant}/by-subject/{subject}/` 获取所有 RoleBinding。
  4. **策略加载**：并行获取关联的 Role。
  5. **规则匹配**：
     - 若存在 `selector`，且 `set(selector) ⊆ set(instance.labels)` → **ALLOW**。
  6. **默认拒绝**：无匹配规则 → **DENY**（HTTP 403）。
  
  > ✅ 时间复杂度：O(k·m)，其中 k 为绑定数，m 为 selector 长度，性能可接受。
  
  ------
  
  ## 7. 用户体验与最佳实践
  
  ### 7.1 标签命名规范建议
  
  为提升可读性与可维护性，推荐采用前缀约定模拟维度：
  
  | 维度   | 命名模式                         | 示例                               |
  | :----- | :------------------------------- | :--------------------------------- |
  | 环境   | `env-{name}`                     | `env-prod`, `env-staging`          |
  | 团队   | `team-{name}`                    | `team-finance`, `team-security`    |
  | 应用   | `app-{name}`                     | `app-payment`, `app-notifications` |
  | 重要性 | `critical`, `debug`, `ephemeral` | —                                  |
  
  ### 7.2 典型授权场景
  
  | 场景           | Selector                        | Verbs                                  |
  | :------------- | :------------------------------ | :------------------------------------- |
  | 生产报表调用   | `["env-prod", "app-reporting"]` | `["Invoke"]`                           |
  | 调试团队访问   | `["env-staging"]`               | `["Invoke", "SaveState", "LoadState"]` |
  | 运维紧急终止   | `["critical"]`                  | `["Kill"]`                             |
  | 跨团队数据处理 | `["team-ml", "project-fraud"]`  | `["Invoke", "Create"]`                 |
  
  ------
  
  ## 8. 架构与运维特性
  
  ### 8.1 无状态鉴权服务
  
  - **零本地存储**：每次鉴权请求独立完成，无会话或缓存依赖（缓存为可选优化）。
  - **弹性伸缩**：可任意扩缩容，无状态同步开销。
  - **故障容忍**：单实例宕机不影响整体可用性。