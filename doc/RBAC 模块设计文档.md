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

  ![rbac设计图](https://github.com/Chamberlain1998/meta-service/blob/main/doc/images/rbac.png?raw=true)

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
  
  | 类型         | 格式              | 说明                                                         |
  | :----------- | :---------------- | :----------------------------------------------------------- |
  | **Function** | `function:{id}`   | 服务模板，由用户命名，代表一类可实例化的功能单元。           |
  | **Instance** | `instance:{uuid}` | 运行时实体，ID 由系统自动生成（如 UUID），不可预测，不可重用。 |

  > ⚠️ Instance 生命周期短暂且动态，其身份管理完全依赖元数据而非名称。

  ### 2.3 操作（Verbs）

  仅支持以下六种原子操作，覆盖实例全生命周期管理：
  
  | Verb        | 语义                              |
  | :---------- | :-------------------------------- |
  | `Create`    | 在指定 Function 下创建新 Instance |
  | `Invoke`    | 触发 Instance 执行任务            |
  | `Exit`      | 请求 Instance 正常退出            |
  | `SaveState` | 保存 Instance 当前运行状态        |
  | `LoadState` | 从快照恢复 Instance 状态          |
  | `Kill`      | 向 Instance 发送信号量            |

  ### 2.4 标签模型（Label Model）
  
  - **结构**：一维字符串集合（`[]string`），无键值对结构。
  - **示例**：`["env-prod", "app-payment", "team-billing", "critical"]`
  - **约束**：
    - 非空（至少一个标签）；
    - 创建后不可变；
    - 不支持通配符、正则或层级语义。

  ### 2.5 主体（Subject）
  
  当前支持租户级主体，格式为：
  `tenant:{tenant-id}`
  表示整个目标租户获得授权，适用于跨租户服务调用场景。

  ### 2.6 角色（Role）
  
  - 权限规则的声明式集合，由资源所属租户定义。
  - **不直接绑定主体**，需通过 RoleBinding 生效。
  - **支持标签选择器**，`instance`和`function`在创建时会将自身关键信息存储到label中。
    - `function`创建时，会自动添加一个`name-{functionName}`标签。
  - `instance`创建时，会自动添加一个`name-{instanceName}`和`function-{functionName}`标签

  ### 2.7 角色绑定（RoleBinding）
  
  - 建立 Role 与 Subject 的授权关系。
  - 明确指定权限作用的租户上下文（`tenant` 字段）。

  ------

  ## 3. 数据模型与存储设计

  ### 3.1 存储后端：etcd
  
  - **唯一可信数据源**：所有策略与元数据持久化至 etcd v3 API。
  - **强一致性保障**：利用 etcd 的 linearizable read/write 保证权限变更立即全局可见。
  - **无本地状态**：RBAC 鉴权服务启动时不加载任何数据，每次请求按需读取。

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

  由控制面在 Instance 创建/销毁时维护：
  
  ```
  /sn/instance/bussiness/yrk/tenant/{tenant-id}/function/{function-name}/version/{version}/defaultaz/{request-id}/{instance-id}
  ```

  **内容结构**：
  
  ```protobuf
  message InstanceInfo {
    // podname in K8S BCM, InstanceID in YuanRong system.
    string instanceID = 1;
  
    // which request to create this instance
    string requestID = 2;
  
    // hostname while be set to /etc/hostname when K8S BCM, runtime in YuanRong system
    string runtimeID = 3;
  
    // runtime ip:port in YuanRong system
    string runtimeAddress = 4;
  
    // functionAgentID in YuanRong system
    string functionAgentID = 5;
  
    // K8S BCM is nodeName;
    string functionProxyID = 6;
  
    // container image in K8S BCM, function name in YuanRong system
    string function = 7;
  
    // the restart policy when instance running failed
    string restartPolicy = 8;
    Resources resources = 9;
  
    Resources actualUse = 10;
  
    // special option for scheduler
    ScheduleOption scheduleOption = 11;
  
    // create options (eg.concurrency)
    map<string, string> createOptions = 12;
  
    // instance labels
    repeated string labels = 13;
  
    // Instance start time
    string startTime = 14;
  
    InstanceStatus instanceStatus = 15;
  
    string jobID = 16;
  
    // the topology is local->domain1->domain2
    repeated string schedulerChain = 17;
  
    // parentID is the instanceID of creator
    string parentID = 18;
  
    // parentFunctionProxyAID is functionProxyAID of creator
    string parentFunctionProxyAID = 19;
  
    // the storage type of the function corresponding to this instance.
    string storageType = 20;
  
    // schedule retry times
    int32 scheduleTimes = 21;
  
    // local redeploy times (in original local scheduler), default is 1
    int32 deployTimes = 22;
  
    // args in creating request
    repeated common.Arg args = 23;
  
    bool isCheckpointed = 24;
  
    // version indicates the number of times that instance information is modified in etcd.
    int64 version = 25;
  
    string dataSystemHost = 26;
  
    bool detached = 27;
  
    int64 gracefulShutdownTime = 28;
  
    string tenantID = 29;
  
    bool isSystemFunc = 30;
  
    string groupID = 31;
  
    // indicate an instance whether is a low reliability instance
    bool lowReliability = 32;
    // extension field
    map<string, string> extensions = 33;
    // the instance was scheduled on this resource unit
    string unitID = 34;
  
    // kv labels 跟黄区proto与runtime的proto不一致，存疑？
    map<string, string> kvLabels = 35;
  }
  ```

  > 🔒 RBAC 服务仅具备 `/sn/instance/bussiness/yrk/tenant/{tenant-id}` 的只读权限。

  ------

  3.2.3 function 元数据

  由MetaService functionManager在 function 创建/销毁时维护：
  
  ```
  /sn/function/bussiness/yrk/tenant/{tenant-id}/function/{function-id}/version/{version}
  ```

  **内容结构**：
  
  ```go
  type FunctionInfo struct {
  	ID                  string             `json:"id"`
  	CreateTime          string             `json:"createTime"`
  	UpdateTime          string             `json:"updateTime"`
  	FunctionURN         string             `json:"functionUrn"`
  	FunctionName        string             `json:"name"`
  	TenantID            string             `json:"tenantId"`
  	BusinessID          string             `json:"businessId"`
  	ProductID           string             `json:"productId"`
  	ReversedConcurrency int                `json:"reversedConcurrency"`
  	Description         string             `json:"description"`
  	LastModified        string             `json:"lastModified"`
  	Published           string             `json:"Published"`
  	MinInstance         int                `json:"minInstance"`
  	MaxInstance         int                `json:"maxInstance"`
  	ConcurrentNum       int                `json:"concurrentNum"`
  	Status              string             `json:"status"`
  	InstanceNum         int                `json:"instanceNum"`
  	Tag                 map[string]string  `json:"tag"`
  	FunctionVersionURN  string             `json:"functionVersionUrn"`
  	RevisionID          string             `json:"revisionId"`
  	CodeSize            int64              `json:"codeSize"`
  	CodeSha256          string             `json:"codeSha256"`
  	BucketID            string             `json:"bucketId"`
  	ObjectID            string             `json:"objectId"`
  	Handler             string             `json:"handler"`
  	Layers              []string           `json:"layers"`
  	CPU                 int                `json:"cpu"`
  	Memory              int                `json:"memory"`
  	Runtime             string             `json:"runtime"`
  	Timeout             int                `json:"timeout"`
  	VersionNumber       string             `json:"versionNumber"`
  	VersionDesc         string             `json:"versionDesc"`
  	Environment         map[string]string  `json:"environment"`
  	CustomResources     map[string]float64 `json:"customResources"`
  	StatefulFlag        int                `json:"statefulFlag"`
    FuncLayer           []Layer            `json:"funcLayer"`
  	Device              types.Device       `json:"device,omitempty"`
  }
  ```

  > 🔒 RBAC 服务仅具备 `/sn/function/bussiness/yrk/tenant/{tenant-id}/` 的只读权限。

  3.2.3 tenant 元数据
  
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
      "cpu_cores": {
        "limit": 32,
        "unit": "cores",
        "is_hard": true,
        "warning_threshold": 0.85
      },
      "instanceCount": {
        "limit": 1000,
        "unit": "count",
        "is_hard": true,
        "warning_threshold": 0.9
      }
    },
    "usages": {
      "cpu_cores": 28.5,
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
  
  - `resources` 中types只支持function和instance两种字段；
  - `selector` 为字符串数组，匹配采用 **集合包含（subset）** 语义。
  - `function`只支持`Create`权限，其他任何权限都将被视作非法。。

  ### 4.2 RoleBinding 定义
  
  ```json
  {
    "id": "role-binding-analytics-access-binding",
    "tenant": "tenant-A",                       // 权限作用上下文（必须 = Role.tenant）
    "role_id": "role-analytics-access",         // 引用 Role ID              
    "subject": "tenant:team-analytics"          // 被授权方      
  }
  ```

  

  ## 5. 对外接口
  
  ```yaml
  paths:
    # =============== 鉴权接口 ===============
    /serverless/v1/authz/check/:
      post:
        summary: Check if the current user is authorized to perform an action
        operationId: checkAuthorization
        requestBody:
          required: true
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CheckAuthzRequest'
              examples:
                example-1:
                  summary: Check permission to create a task
                  value:
                    action: "task:create"
                    resource: "project-alpha"
        responses:
          '200':
            description: Authorization check passed
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/CheckAuthzResponse'
                examples:
                  success:
                    summary: Authorized
                    value:
                      authorized: true
                      tenant_id: "t-abc123"
                      user_id: "u-789"
          '403':
            description: Not authorized
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/Error'
                examples:
                  denied:
                    summary: Access denied
                    value:
                      error: "rbac_denied"
                      message: "User lacks required role for action 'task:create'"
          '401':
            $ref: '#/components/responses/Unauthorized'
    schemas:
      # --- AuthZ ---
      CheckAuthzRequest:
        type: object
        required: [action]
        properties:
          verb:
            type: string
            description: The permission string to check 
            example: "Create"
          resource:
            type: string
            description: Optional resource identifier (e.g., project ID, dataset name)
            example: "proj-xyz"
          tenant:
          	type: string
          	description: The target tenant 
          	example: "tenant-0"
      CheckAuthzResponse:
        type: object
        properties:
          authorized:
            type: boolean
            example: true
          error:
          	$ref: '#/components/schemas/Error'
      Error:
        type: object
        properties:
          error:
            type: string
            example: "quota_exceeded"
          message:
            type: string
            example: "GPU hours quota exceeded"
          resource:
            type: string
            example: "gpu_hours"
          limit:
            type: integer
          usage:
            type: integer
          requested:
            type: integer
        required: [error]
  
    responses:
      Unauthorized:
        description: Missing or invalid JWT token
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Error'
            example:
              error: "invalid_token"
              message: "JWT signature verification failed"
  ```

  

  ## 6. 权限评估逻辑

  ### 6.1 输入
  
  - `subject`: 请求主体（如 `tenant-analytics`）
  - `context_tenant`: 目标资源所属租户（如 `tenant-platform`）
  - `resource`: 目标资源（如 `instance:inst-abc123`）
  - `verb`: 请求操作（如 `Invoke`）

  ### 6.2 评估流程
  
  1. **合法性校验**：验证 `verb` 是否属于六种标准操作。
  2. **元数据获取**：从 `/sn/instance/bussiness/yrk/tenant/{tenant-id}/function/{function-name}/version/{version}/defaultaz/{request-id}/{instance-id}` 读取 Instance 标签。
  3. **绑定查询**：通过反向索引 /serverless/v1/rbac/rolebindings/{context_tenant}/by-subject/{subject}/` 获取所有 RoleBinding。
  4. **策略加载**：并行获取关联的 Role。
  5. **规则匹配**：
     - 若 `verb == "Create"` 且 `function:{fn}` 在资源列表中 → **ALLOW**；
     - 若存在 `selector`，且 `set(selector) ⊆ set(instance.labels)` → **ALLOW**。
  6. **默认拒绝**：无匹配规则 → **DENY**（HTTP 403）。

  > ✅ 时间复杂度：O(k·m)，其中 k 为绑定数，m 为 selector 长度，性能可接受。

  ------

  ## 7. 安全与合规约束
  
  | 约束项                | 说明                                                         |
  | :-------------------- | :----------------------------------------------------------- |
  | **禁止精确授权**      | Role 中不得出现 `instance:xxx` 形式的资源引用                |
  | **标签不可变性**      | Instance 标签在创建后锁定，防止运行时权限逃逸                |
  | **租户隔离**          | 所有 etcd 读写操作严格限定于 `context_tenant` 命名空间       |
  | **Function 语义限制** | 任何对 function 配置的非`Create`权限都将被丢弃，不会被继承到由该 function 创建的 instance 中 |

  ------

  ## 8. 用户体验与最佳实践

  ### 8.1 标签命名规范建议

  为提升可读性与可维护性，推荐采用前缀约定模拟维度：
  
  | 维度   | 命名模式                         | 示例                               |
  | :----- | :------------------------------- | :--------------------------------- |
  | 环境   | `env-{name}`                     | `env-prod`, `env-staging`          |
  | 团队   | `team-{name}`                    | `team-finance`, `team-security`    |
  | 应用   | `app-{name}`                     | `app-payment`, `app-notifications` |
  | 重要性 | `critical`, `debug`, `ephemeral` | —                                  |

  ### 8.2 典型授权场景
  
  | 场景           | Selector                        | Verbs                                  |
  | :------------- | :------------------------------ | :------------------------------------- |
  | 生产报表调用   | `["env-prod", "app-reporting"]` | `["Invoke"]`                           |
  | 调试团队访问   | `["env-staging"]`               | `["Invoke", "SaveState", "LoadState"]` |
  | 运维紧急终止   | `["critical"]`                  | `["Kill"]`                             |
  | 跨团队数据处理 | `["team-ml", "project-fraud"]`  | `["Invoke", "Create"]`                 |

  ------

  ## 9. 架构与运维特性

  ### 9.1 无状态鉴权服务
  
  - **零本地存储**：每次鉴权请求独立完成，无会话或缓存依赖（缓存为可选优化）。
  - **弹性伸缩**：可任意扩缩容，无状态同步开销。
  - **故障容忍**：单实例宕机不影响整体可用性。

  ### 9.2 可选缓存机制
  
  - **Watch 驱动**：监听 `/rbac/v1/...` 和 `/platform/v1/instances/...` 变更事件。
  - **TTL 保底**：缓存条目设置生存时间（如 60s），防止 stale data。
  - **兜底直连**：缓存未命中时，直接查询 etcd。

  ### 9.3 管理接口
  
  - **策略写入**：由专用配置服务或 GitOps 工具写入 etcd，RBAC 模块仅读。
  - **审计就绪**：所有 etcd 写入操作可集成审计日志（如 etcd audit log + SIEM）。