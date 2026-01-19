# API 接口列表

本文档列出了 MetaService Serverless Platform 支持的所有 API 接口。

## 接口列表

| 标签 | HTTP方法 | 路径 | 操作ID | 描述 |
|------|---------|------|--------|------|
| Tenant | POST | `/tenants` | createTenant | 创建新租户 |
| Tenant | GET | `/tenants` | listTenants | 列出租户 |
| Tenant | GET | `/tenants/{tenant_id}` | getTenant | 获取租户详情 |
| Tenant | PATCH | `/tenants/{tenant_id}` | updateTenant | 部分更新租户(支持更新配额/状态等,不支持改ID) |
| Tenant | DELETE | `/tenants/{tenant_id}` | deleteTenant | 删除租户 |
| RBAC | POST | `/roles` | createRole | 在当前租户下创建角色 |
| RBAC | GET | `/roles/{role_id}` | getRole | 获取角色详情 |
| RBAC | PUT | `/roles/{role_id}` | updateRole | 更新角色 |
| RBAC | DELETE | `/roles/{role_id}` | deleteRole | 删除角色 |
| RBAC | POST | `/rolebindings` | createRoleBinding | 创建角色绑定 |
| RBAC | GET | `/rolebindings/{rolebinding_id}` | getRoleBinding | 获取角色绑定详情 |
| RBAC | DELETE | `/rolebindings/{rolebinding_id}` | deleteRoleBinding | 删除角色绑定 |
| Authz | POST | `/authz` | checkAuthorization | 检查主体是否有权限执行某操作 |

## 认证方式

所有接口需要通过以下认证方式之一访问：
- **JwtAuth**: 使用 IAM 颁发的 JWT 访问令牌
- **AccessKeyAuth**: 使用 AccessKey/SecretKey 进行 HMAC-SHA256 签名

## 基础路径

```
/serverless/v1
```

