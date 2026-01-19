# MetaService 组件整体设计文档

## 文档信息

| 项目     | 内容                                                         |
| -------- | ------------------------------------------------------------ |
| 文档名称 | MetaService 组件整体设计文档                                 |
| 版本号   | 1.0                                                          |
| 作者     | 纪成林                                                       |
| 审核人   | [待定]                                                       |
| 生效日期 | 2026年2月xx日                                                |
| 适用系统 | 支持多租户、函数即服务（FaaS）或动态实例化工作负载的云原生平台 |

## 1. 概述

MetaService 是云原生平台的核心元数据管理服务，负责维护平台的关键元数据，包括租户、函数、角色和角色绑定等信息。该服务采用模块化设计，通过严格的准入检查机制确保系统安全与数据一致性。所有请求在到达具体功能模块前，必须经过AdmitHandler链的鉴权和准入检查。

![MetaService组件图](https://github.com/Chamberlain1998/meta-service/blob/main/doc/images/MetaService组件图.drawio.png?raw=true)

## 2. 设计原则

- **单一可信数据源**：所有元数据存储于etcd，确保数据一致性
- **租户隔离**：严格隔离不同租户的数据，确保数据安全
- **细粒度权限控制**：基于RBAC模型实现细粒度权限控制
- **无状态架构**：所有组件均为无状态设计，支持水平弹性伸缩
- **最小权限原则**：每个组件仅具备必要的etcd访问权限
- **高可用性**：通过etcd的高可用性和组件的无状态特性实现

## 3. 组件架构

![MetaService组件图.drawio](/Users/chamberlain/Downloads/MetaService组件图.drawio.png)
*图：MetaService组件架构图，展示各组件间交互关系*

## 4. 主要模块设计

### 4.1 Server 组件

**角色**：MetaService的统一入口，提供HTTP/HTTPS路由服务

**核心功能**：
- 提供统一的API路由接口
- 作为AdmitHandler链的入口点
- 处理请求的序列化和反序列化
- 为请求添加必要的上下文信息
- 负责API版本管理

**路由设计**：
```http
/serverless/v1/tenants          # 租户管理API
/serverless/v1/functions        # 函数管理API
/serverless/v1/roles            # 角色管理API
/serverless/v1/rolebindings     # 角色绑定管理API
/serverless/v1/authz            # 鉴权检查API
```

**工作流程**：

1. 接收HTTP/HTTPS请求
2. 将请求传递给AdmitHandler链
3. 处理AdmitHandler返回的Error信息，如果有错误则直接返回异常信息
4. 根据请求路径分发到对应模块
5. 收集处理结果
6. 返回HTTP/HTTPS响应

**关键设计**：
- 采用Go的`gorilla`包实现轻量级HTTP服务
- 使用路由中间件处理请求生命周期
- 提供OpenAPI 3.0规范的API文档

### 4.2 IAM 组件

**角色**：继承自AdmitHandler，负责请求的认证和租户信息提取

![admitHandler类图](https://github.com/Chamberlain1998/meta-service/blob/main/doc/images/admitHandler类图.drawio.png?raw=true)

**核心功能**：
- 验证JWT令牌
- 验证AKSK凭证

**认证流程**：
```
1. 检查请求头中的Authorization
2. 如果包含Bearer token，验证JWT
3. 如果包含AKSK，验证AKSK凭证
```

**JWT验证细节**：
- Xxx

### 4.3 tenantMgr 组件

**角色**：继承自AdmitHandler，负责租户信息维护和配额检查

**核心功能**：

- 维护etcd中租户元数据
- 检查请求涉及的租户有效性
- 对涉及Function的Create请求进行配额检查
- 根据Function的Create和instance的Kill请求更新租户配额使用情况

**租户元数据存储**：
路径：`/sn/tenant/bussiness/yrk/{tenant-id}`
内容结构：

```json
{
  "version": 123,
  "tenant_id": "t-abc123",
  "status": "active",
  "quotas": {
    "instanceCount": {
      "limit": 1000,
      "unit": "count",
      "is_hard": true
    }
  },
  "usages": {
    "cpu_cores": 28.5,
    "instanceCount": 890
  },
  "last_updated": "2026-01-16T10:15:00Z"
}
```

**配额检查流程**：
```
1. 检查请求中的租户ID
2. 从etcd获取租户元数据
3. 验证租户状态
4. 如果是Instance操作（Create/Kill）：
   a. 检查配额是否足够
   b. 如果是Create，计算实例数量的配额
   c. 如果是Kill，更新实例数量配额
5. 如果配额不足，拒绝请求
6. 如果配额足够，更新租户使用情况
```

**关键设计**：

- 支持硬配额（is_hard=true）和软配额（is_hard=false）
- 提供配额使用情况的实时更新

**约束：**

- 当前版本quota管理只支持实例数量控制

### 4.4 RBAC 组件

**角色**：继承自AdmitHandler，负责权限检查

**核心功能**：
- 维护etcd中Role和RoleBinding信息
- 检查请求主体是否有权限执行指定操作
- 基于标签模型进行细粒度权限检查

**RBAC数据存储**：
```
/sn/rbac/bussiness/yrk/
├── roles/
│   └── {tenant-id}/{role-id}.json
└── rolebindings/
    ├── {tenant-id}/{binding-id}.json
    └── by-subject/{subject}/{binding-id}
```

**Role定义**：
```json
{
  "id": "role-analytics-access",
  "tenant": "tenant-A",
  "rules": [
    {
      "resources": [
        {
          "type": "function",
          "selector": ["name-function-0", "env-prod", "team-public"]
        }
      ],
      "verbs": ["Create"]
    },
    {
      "resources": [
        {
          "type": "instance",
          "selector": ["name-instance-001", "function-function-0", "env-prod", "team-public"]
        }
      ],
      "verbs": ["Invoke", "SaveState"]
    }
  ]
}
```

**RoleBinding定义**：
```json
{
  "id": "role-binding-analytics-access-binding",
  "tenant": "tenant-A",
  "role_id": "role-analytics-access",
  "subject": "tenant:team-analytics"
}
```

**权限检查流程**：

```
1. 从请求上下文中获取subject（如tenant:team-analytics）
2. 获取context_tenant（目标资源所属租户，如tenant-platform）
3. 获取resource（如instance:inst-abc123）和verb（如Invoke）
4. 从etcd获取相关RoleBinding（通过by-subject索引）
5. 并行加载关联的Role
6. 检查是否存在匹配规则：
   a. 如果资源类型为function且verb == "Create"，检查标签匹配：set(selector) ⊆ set(function.labels) → ALLOW
   b. 如果资源类型为instance，检查标签匹配：set(selector) ⊆ set(instance.labels)
   c. 如果资源类型为Tenant，检查发起方是否有权限进行
7. 如果匹配，ALLOW；否则，DENY（返回403）
```

**关键设计**：

- 当前共支持三种资源[function, instance, tenant]
- Function支持Create权限
- Function创建时，会自动添加一个`name-{functionName}`标签。
- Instance 支持六种posix操作（Create, Invoke, Exit, SaveState, LoadState, Kill）
- Instance 创建时，会自动添加一个`name-{instanceName}`和`function-{functionName}`标签
- Tenant 支持Create，Delelet，Update，Get权限
- Tenant 当前只能有**root（预制）**用户有权限进行增删改查
- 每个租户默认对自己的function, instance拥有全部权限

- 采用集合包含（subset）语义进行标签匹配

### 4.5 functionMgr 组件

**角色**：负责维护etcd中函数元信息

**核心功能**：
- 维护etcd中函数元数据
- 提供函数的创建、更新、删除、查询接口
- 确保函数元数据的完整性

**函数元数据存储**：
路径：`/sn/function/bussiness/yrk/tenant/{tenant-id}/function/{function-id}/version/{version}`
内容结构：

```json
{
  "id": "",
  "createTime": "",
  "updateTime": "",
  "functionUrn": "",
  "name": "",
  "tenantId": "",
  "businessId": "",
  "productId": "",
  "reversedConcurrency": 0,
  "description": "",
  "lastModified": "",
  "Published": "",
  "minInstance": 0,
  "maxInstance": 0,
  "concurrentNum": 0,
  "status": "",
  "instanceNum": 0,
  "tag": {},
  "functionVersionUrn": "",
  "revisionId": "",
  "codeSize": 0,
  "codeSha256": "",
  "bucketId": "",
  "objectId": "",
  "handler": "",
  "layers": [],
  "cpu": 0,
  "memory": 0,
  "runtime": "",
  "timeout": 0,
  "versionNumber": "",
  "versionDesc": "",
  "environment": {},
  "customResources": {},
  "statefulFlag": 0
}
```

## 5. 请求处理流程

### 5.1 完整请求处理流程

```
1. 客户端发送HTTP请求到Server
2. Server将请求传递给IAM组件
   a. 验证JWT/AKSK
3. Server将请求传递给tenantMgr组件
   a. 验证租户有效性
   b. 检查配额（如果涉及）
   c. 更新配额使用情况（如果涉及）
4. Server将请求传递给RBAC组件
   a. 检查subject是否有权限执行操作
   b. 返回权限检查结果
5. Server根据请求路径分发到对应模块
   a. 如果是租户请求 → tenantMgr
   b. 如果是函数请求 → functionMgr
   c. 如果是角色请求 → RBAC
6. 功能模块处理请求，更新etcd
7. Server返回HTTP响应
```

### 5.2 典型请求处理示例：创建Instance

```
1. 客户端发送POST /api/v1/functions/{function-id}/instances
2. IAM验证JWT，提取tenant_id
3. tenantMgr检查租户配额（instanceCount）
4. RBAC检查tenant_id是否有权限执行Create操作
5. functionMgr获取函数元数据，验证函数是否存在
6. functionMgr创建Instance，添加标签
7. 更新租户配额
8. 返回201 Created响应
```

## 6. 数据存储设计

### 6.1 etcd键空间布局

```
/sn/
├── tenant/
│   └── bussiness/
│       └── yrk/
│           └── {tenant-id}  # 租户元数据
├── function/
│   └── bussiness/
│       └── yrk/
│           └── tenant/
│               └── {tenant-id}/
│                   └── function/
│                       └── {function-id}/
│                           └── version/
│                               └── {version}  # 函数元数据
├── instance/
│   └── bussiness/
│       └── yrk/
│           └── tenant/
│               └── {tenant-id}/
│                   └── function/
│                       └── {function-name}/
│                           └── version/
│                               └── {version}/
│                                   └── defaultaz/
│                                       └── {request-id}/
│                                           └── {instance-id}  # Instance元数据
└── rbac/
    └── bussiness/
        └── yrk/
            ├── roles/
            │   └── {tenant-id}/
            │       └── {role-id}.json
            └── rolebindings/
                ├── {tenant-id}/
                │   ├── {binding-id}.json
                │   └── by-subject/
                │       └── {subject}/
                │           └── {binding-id}
                └── {tenant-id}/
                    └── by-subject/{subject}/{binding-id}
```

### 6.2 组件etcd权限

| 组件        | 读权限                                                       | 写权限                                                       | 说明                 |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------- |
| IAM         | `/sn/tenant/bussiness/yrk/{tenant-id}`                       | 无                                                           | 仅需读取租户信息     |
| tenantMgr   | `/sn/tenant/bussiness/yrk/{tenant-id}`                       | `/sn/tenant/bussiness/yrk/{tenant-id}`                       | 读取和更新租户信息   |
| RBAC        | `/sn/rbac/bussiness/yrk/roles/{tenant-id}``/sn/rbac/bussiness/yrk/rolebindings/{tenant-id}` | `/sn/rbac/bussiness/yrk/roles/{tenant-id}``/sn/rbac/bussiness/yrk/rolebindings/{tenant-id}` | 读取和更新RBAC策略   |
| functionMgr | `/sn/function/bussiness/yrk/tenant/{tenant-id}/function/{function-id}/version/{version}` | `/sn/function/bussiness/yrk/tenant/{tenant-id}/function/{function-id}/version/{version}` | 读取和更新函数元数据 |
| Server      | 无                                                           | 无                                                           | 仅作为路由层         |

## 7. 安全与合规

- **数据加密**：etcd数据存储使用TLS加密
- **访问控制**：所有组件仅具备必要的etcd路径访问权限
- **审计日志**：所有etcd写入操作记录审计日志
- **配额限制**：租户配额限制防止资源滥用
- **最小权限原则**：每个组件仅具备必要的etcd权限
- **权限验证**：所有请求必须通过IAM、tenantMgr和RBAC检查
- **敏感信息处理**：AKSK等敏感信息不存储在日志中

## 8. 高可用与可扩展

- **无状态设计**：所有组件无状态，支持水平扩展
- **etcd高可用**：etcd集群提供高可用性
- **故障隔离**：单个组件故障不影响整体服务
- **重试机制**：etcd操作失败时实现指数退避重试

## 9. 最佳实践

### 9.1 标签命名规范

| 维度   | 命名模式                   | 示例                           |
| ------ | -------------------------- | ------------------------------ |
| 环境   | env-{name}                 | env-prod, env-staging          |
| 团队   | team-{name}                | team-finance, team-security    |
| 应用   | app-{name}                 | app-payment, app-notifications |
| 重要性 | critical, debug, ephemeral | —                              |

### 9.2 典型授权场景

| 场景           | Selector                      | Verbs                                |
| -------------- | ----------------------------- | ------------------------------------ |
| 生产报表调用   | ["env-prod", "app-reporting"] | ["Invoke"]                           |
| 调试团队访问   | ["env-staging"]               | ["Invoke", "SaveState", "LoadState"] |
| 运维紧急终止   | ["critical"]                  | ["Kill"]                             |
| 跨团队数据处理 | ["team-ml", "project-fraud"]  | ["Invoke", "Create"]                 |

### 9.3 配额管理最佳实践

- **硬配额**：设置硬配额（is_hard=true）防止资源滥用
- **软配额**：设置软配额（is_hard=false）用于预警
- **配额预警**：当使用率超过85%时发送预警
- **配额升级**：提供自助式配额升级接口
- **配额审计**：定期审计配额使用情况

## 10. 附录 A：接口列表

[接口列表]: https://github.com/Chamberlain1998/meta-service/blob/main/doc/API接口列表.md

## 11. 附录B：参数结构列表

[参数列表]: https://github.com/Chamberlain1998/meta-service/blob/main/doc/API数据结构定义.md

## 12. 附录B：OpenAPI 3.0 规范接口文件

[openapi.yaml]: https://github.com/Chamberlain1998/meta-service/blob/main/api/openapi.yaml

