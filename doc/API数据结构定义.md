# API 数据结构定义

本文档列出了 MetaService Serverless Platform API 中使用的所有数据结构定义。

## 数据结构列表

| Schema名称 | 类型 | 必需字段 | 描述 |
|-----------|------|---------|------|
| Tenant | object | tenant\_id, quotas, usages, created\_at, last\_updated | 租户对象,包含租户ID、状态、配额、使用量和元数据 |
| QuotasMap | object | - | 配额键值对,如cpu\_cores、memory\_gb、functions |
| UsagesMap | object | - | 与配额键对应的实际使用量 |
| Quota | object | limit, unit, is\_hard | 配额详情,包含限制值、单位和是否硬限制 |
| TenantCreateRequest | object | tenant\_id, quotas | 创建租户请求体 |
| TenantUpdateRequest | object | - | 更新租户请求体,可更新status和quotas |
| Role | object | id, tenant, rules, created\_at, last\_updated | 角色对象,包含角色ID、租户、规则和元数据 |
| RoleRule | object | resources, verbs | 角色规则,定义资源和允许的操作 |
| RoleResource | object | type, selector | 角色资源,指定资源类型和选择器 |
| RoleCreateRequest | object | tenant, rules | 创建角色请求体 |
| RoleUpdateRequest | object | - | 更新角色请求体,可更新rules |
| RoleBinding | object | id, role\_id, subject, tenant, created\_at, last\_updated | 角色绑定对象,将角色绑定到主体 |
| RoleBindingCreateRequest | object | role\_id, subject, tenant | 创建角色绑定请求体 |
| AuthzRequest | object | subject, context\_tenant, resource, verb | 鉴权检查请求,包含主体、租户、资源和操作 |
| ResourceIdentifier | object | type, id | 资源标识符,指定资源类型和唯一ID |
| AuthzResponse | object | allowed | 鉴权检查响应,返回是否允许及拒绝原因 |
| Metadata | object | - | 资源元数据,包含labels和annotations |
| ErrorResponse | object | code, message | 标准错误响应,包含错误码、消息和详细信息 |

## 数据结构详细说明

### 核心资源对象

- **Tenant**: 租户管理的核心对象，包含配额限制和使用情况
- **Role**: RBAC 角色定义，包含访问规则集合
- **RoleBinding**: 将角色绑定到主体（用户/服务账号）

### 请求对象

- **TenantCreateRequest**: 创建租户时的请求体
- **TenantUpdateRequest**: 更新租户时的请求体（支持部分更新）
- **RoleCreateRequest**: 创建角色时的请求体
- **RoleUpdateRequest**: 更新角色时的请求体
- **RoleBindingCreateRequest**: 创建角色绑定时的请求体
- **AuthzRequest**: 权限检查请求

### 响应对象

- **AuthzResponse**: 权限检查响应
- **ErrorResponse**: 标准错误响应格式

### 辅助对象

- **QuotasMap**: 配额映射表
- **UsagesMap**: 使用量映射表
- **Quota**: 单个配额的详细定义
- **RoleRule**: 角色规则定义
- **RoleResource**: 角色资源选择器
- **ResourceIdentifier**: 资源标识符
- **Metadata**: 资源元数据（标签和注解）

## 枚举值

### Tenant Status
- `active`: 租户活跃
- `suspended`: 租户暂停

### Resource Type
- `function`: 函数
- `instance`: 实例
- `tenant`: 租户
- `role`: 角色
- `role_binding`: 角色绑定

### Verbs (操作)
- `create`: 创建
- `invoke`: 调用
- `exit`: 退出
- `save_state`: 保存状态
- `load_state`: 加载状态
- `kill`: 终止
- `update`: 更新
- `delete`: 删除
- `get`: 获取
- `list`: 列表

### Quota Unit (配额单位)
- `count`: 数量
- `cores`: 核心数
- `MB`: 兆字节
- `GB`: 吉字节
- `TB`: 太字节
- `requests_per_second`: 每秒请求数
- `concurrent_instances`: 并发实例数

### Error Code (错误码)
- `INVALID_REQUEST`: 无效请求
- `UNAUTHORIZED`: 未授权
- `FORBIDDEN`: 禁止访问
- `NOT_FOUND`: 未找到
- `CONFLICT`: 冲突
- `INTERNAL_ERROR`: 内部错误
- `ETCD_ERROR`: etcd 错误
- `QUOTA_EXCEEDED`: 配额超限

