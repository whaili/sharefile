# Bearer Token Scope 与 IAM 联合权限控制方案

## 一、双层权限控制：为什么需要 IAM + Bearer Token Scope

### 1.1 核心概念

| 组件 | 职责 | 典型场景 |
|------|------|---------|
| **IAM（身份与访问管理）** | 长期身份管理 + 资源级策略控制 | 子账号管理、精细化资源授权、审计追踪 |
| **Bearer Token Scope** | API 访问凭证 + 粗粒度权限标记 | API 认证、临时授权、第三方应用接入 |

### 1.2 为什么需要两层控制

**权限计算公式**：
```
最终权限 = IAM 策略 ∩ Token Scope（取交集，最小权限原则）
```

**核心价值**：

| 问题场景 | 仅有 Token Scope 的问题 | 仅有 IAM 的问题 | 双层方案的解决 |
|---------|---------------------|----------------|--------------|
| **权限细化** | 无法精确到资源级别<br>（如限制特定 bucket） | 策略过于重量级<br>（每次都要修改策略） | Token Scope 粗粒度 + IAM 细粒度 = 灵活组合 |
| **临时降权** | Token 权限无法小于账号权限 | 修改 IAM 策略流程重，且难回收 | 创建受限 Token（Scope 小于 IAM 策略），自动过期 |
| **Token 泄露防护** | Token 拥有账号全部权限，泄露风险高 | 无 Token 机制，只能用 AK/SK | Token Scope 限制最大权限，即使泄露影响可控 |
| **审计与溯源** | 只能追踪到 Token，无法关联操作人 | 无 Token 粒度的审计 | Token ID + 子账号 UID + 操作记录 = 完整审计链 |

### 1.3 协作流程

```
用户请求 → Bearer Token 验证 → 提取子账号 UID → IAM 策略检查 → 执行操作
            ↓                                      ↓
         ✓ Token 有效性                         ✓ 资源级权限
         ✓ Scope 权限                          ✓ 条件约束（IP/时间）
         ✓ 过期时间                            ✓ 显式拒绝策略
```

---

## 二、业界主流云厂商实践对比

### 2.1 权限模型差异

各家云厂商在 IAM 和 Scope 的职责划分上有显著差异：

| 云厂商 | Scope 职责 | IAM 职责 | 权限计算 |
|--------|-----------|---------|---------|
| **Google Cloud** | **服务级 + 粗粒度操作**<br>定义可访问哪些 GCP 服务及大致操作级别<br>例：`https://www.googleapis.com/auth/cloud-platform.read-only` | **资源级细粒度授权**<br>精确到具体资源的权限<br>例：允许读取特定 bucket | IAM Binding ∩ Token Scope |
| **AWS** | **完整策略约束**<br>Session Policy 可完整定义资源、操作、条件<br>（类似 IAM Policy 的 JSON 格式） | **长期身份策略**<br>User/Role 的权限边界 | IAM Policy ∩ Session Policy |
| **Azure** | **API 权限声明**<br>声明访问特定资源服务器的权限<br>例：`https://storage.azure.com/user_impersonation` | **资源级 RBAC**<br>角色分配到具体资源<br>例：Storage Blob Data Reader | RBAC ∩ Token Scope |
| **阿里云/腾讯云/火山引擎** | **完整策略约束**<br>STS 申请时可指定 Policy 参数<br>（JSON 格式，支持资源级限制） | **RAM/CAM/IAM 策略**<br>子账号的长期权限策略 | IAM Policy ∩ STS Policy |

### 2.2 典型示例对比

#### Google Cloud (GCP)

**Scope 粒度**：服务级 + 操作级别
```
https://www.googleapis.com/auth/compute.readonly          # Compute Engine 只读
https://www.googleapis.com/auth/devstorage.read_write     # Cloud Storage 读写
https://www.googleapis.com/auth/cloud-platform            # 所有 GCP 服务完全访问
```

**IAM 粒度**：资源级
```json
{
  "role": "roles/storage.objectViewer",
  "members": ["user:alice@example.com"],
  "condition": {
    "expression": "resource.name.startsWith('projects/_/buckets/my-bucket')"
  }
}
```

**权限计算**：
- Access Token 的 Scope 包含 `devstorage.read_write` → 可以访问 Cloud Storage 读写 API
- IAM 绑定了 `storage.objectViewer` → 只能查看 `my-bucket` 的对象
- **最终权限**：只能读取 `my-bucket`（Scope 允许读写，但 IAM 只允许读）

---

#### AWS

**Session Policy 粒度**：完整策略（资源级）
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": ["arn:aws:s3:::my-bucket/*"]
  }]
}
```

**IAM Policy 粒度**：完整策略（资源级）
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*"],
    "Resource": ["arn:aws:s3:::my-bucket/*"]
  }]
}
```

**权限计算**：
- Session Policy 允许 `s3:GetObject`
- IAM Policy 允许 `s3:*`（所有 S3 操作）
- **最终权限**：只能 `s3:GetObject`（取交集）

---

#### 阿里云/腾讯云/火山引擎

**STS Policy 粒度**：完整策略（资源级）
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Action": ["oss:GetObject"],
    "Resource": ["acs:oss:*:*:my-bucket/*"]
  }]
}
```

**RAM/CAM Policy 粒度**：完整策略（资源级）
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Action": ["oss:*"],
    "Resource": ["acs:oss:*:*:my-bucket/*"]
  }]
}
```

**权限计算**：同 AWS，取交集

### 2.3 设计思路总结

| 设计思路 | 代表厂商 | 核心特点 | 适用场景 |
|---------|---------|---------|---------|
| **分层授权**<br>（粗粒度 Scope + 细粒度 IAM） | Google Cloud<br>Azure | Scope 定义服务和操作级别（粗）<br>IAM 控制具体资源（细）<br>**职责分离明确** | OAuth 2.0 授权<br>第三方应用集成 |
| **双策略模型**<br>（动态策略 + 静态策略） | AWS<br>阿里云<br>腾讯云<br>火山引擎 | 两层都是完整 Policy<br>**关键区别是生命周期**：<br>• IAM Policy：长期静态策略<br>• Session Policy：临时动态策略 | 临时降权<br>跨账号访问<br>权限委托 |

### 2.4 双策略模型的必要性

**问题**：既然两层都是资源级策略，为什么需要两层？

**答案**：核心价值在于**临时降权**和**动态授权**

**典型场景**：

```
管理员账号的 IAM Policy（长期权限）:
{
  "Action": ["s3:*"],                    # 所有 S3 操作
  "Resource": ["arn:aws:s3:::*"]         # 所有 bucket
}

给第三方应用生成临时凭证时，指定 Session Policy（临时权限）:
{
  "Action": ["s3:GetObject"],            # 仅读取
  "Resource": ["arn:aws:s3:::public-bucket/*"]  # 仅这个 bucket
}

最终权限（取交集）:
- 只能读取 public-bucket 的对象
- 有效期 1 小时后自动失效
```

**价值**：
- ✅ 无需修改 IAM Policy（保持管理员长期权限不变）
- ✅ 临时凭证自动过期，无需手动回收
- ✅ 不同场景可以生成不同权限的临时凭证
- ✅ 审计时可以区分是用主凭证还是临时凭证访问

**对比单层方案**：
- ❌ 如果只有 IAM：需要修改策略来实现临时降权，流程重且难回收
- ❌ 如果只有 Token Scope：Scope 粒度太粗（如 `s3:read`），无法限制到具体 bucket

### 2.5 共同特点

尽管实现细节不同，但所有云厂商都遵循以下原则：

- ✅ **双层权限控制**：IAM（长期身份）+ Token（临时凭证）
- ✅ **取交集原则**：最终权限 = IAM ∩ Token Scope/Policy
- ✅ **临时性**：Token 有过期时间（15min ~ 24h）
- ✅ **最小权限**：Token 权限不能超过 IAM 策略范围

---

## 三、七牛现状与 Bearer Token Scope 设计方案

### 3.1 七牛 IAM 体系现状

**已具备的 IAM 能力**：

#### 用户管理
主账号、子账号、用户信息管理

![IAM 用户管理](pic/IAM-用户.png)

#### 用户组
用户分组、批量授权

![IAM 用户组](pic/IAM-用户组.png)

#### 策略管理
策略列表、预设策略、自定义策略

![IAM 策略列表](pic/IAM-策略列表.png)

#### 策略详情
资源级权限定义（Resource、Action、Effect）

![IAM 策略详情](pic/IAM-策略详情.png)

**IAM 策略示例**（基于现有体系）：
```json
{
  "version": "1",
  "statement": [
    {
      "effect": "allow",
      "action": ["kodo:GetObject", "kodo:PutObject"],
      "resource": ["qrn:qiniu:kodo::12345:bucket/my-bucket/*"]
    }
  ]
}
```

### 3.2 Bearer Token Scope 设计方案

#### 推荐方案：分层授权模型（参考 GCP）

**为什么选择分层授权而不是双策略模型？**

| 对比项 | GCP 分层授权 | AWS 双策略模型 | 七牛现状 |
|--------|------------|--------------|---------|
| **Scope 粒度** | 服务级+操作级（粗） | 完整 Policy（细） | Bearer Token 已有 Scope 字段 |
| **实现复杂度** | 低（Scope 是字符串数组） | 高（需要完整 Policy 解析器） | 现有系统已支持简单 Scope |
| **用户易用性** | 高（勾选框选择） | 低（需要编写 JSON Policy） | 符合现有用户体验 |
| **IAM 集成** | 现有 IAM 无需改造 | 现有 IAM 无需改造 | 已有完善的 IAM 体系 |

**设计原则**：
- Token Scope 采用**粗粒度权限标记**（如 `kodo:read`, `kodo:write`）— 服务级+操作级
- IAM 策略提供**细粒度资源控制**（如具体 bucket/路径）— 资源级
- Token 归属于特定子账号，验证时关联 IAM 策略
- **职责明确分离**：Scope 控制"能访问哪些服务的哪类操作"，IAM 控制"能访问哪些具体资源"

#### Scope 格式

```
<service>:<action>

示例：
kodo:read          # Kodo 读权限
kodo:write         # Kodo 写权限
cdn:refresh        # CDN 刷新权限
mikustream:*       # MikuStream 所有权限
*                  # 全局权限（慎用）
```

#### 验证流程

```python
def validate_request(token, action, resource):
    # 第一层：验证 Bearer Token
    token_info = bearer_service.validate(token)
    if not token_info["valid"]:
        return {"allowed": False, "reason": "Invalid token"}

    # 提取子账号 UID 和 Token Scope
    sub_account_uid = token_info["uid"]
    token_scope = token_info["scope"]

    # 第二层：检查 Token Scope（粗粒度）
    if not check_scope_match(token_scope, action):
        return {"allowed": False, "reason": "Token scope insufficient"}

    # 第三层：检查 IAM 策略（细粒度）
    iam_allowed = iam.evaluate_policy(
        uid=sub_account_uid,
        action=action,
        resource=resource
    )
    if not iam_allowed:
        return {"allowed": False, "reason": "IAM policy denied"}

    return {"allowed": True}
```

#### 权限匹配逻辑

| Token Scope | 请求 Action | IAM Policy 允许的 Resource | 最终结果 |
|-------------|------------|-------------------------|---------|
| `kodo:read` | `kodo:GetObject` | `bucket-a/*` | ✅ 允许（都通过） |
| `kodo:write` | `kodo:GetObject` | `bucket-a/*` | ❌ 拒绝（Scope 不匹配） |
| `kodo:read` | `kodo:GetObject` | `bucket-b/*` | ❌ 拒绝（IAM 资源不匹配） |
| `kodo:*` | `kodo:DeleteBucket` | `bucket-a` | ✅ 允许（都通过） |


---

## 四、Token 业务前缀命名方案

### 重要说明：前缀 vs 权限

**Token 前缀命名仅作为助记符，与权限控制无关。**

| 概念 | 作用 | 示例 |
|------|------|------|
| **Token 前缀**<br>（如 `kd_`, `qn_`） | 助记作用，帮助用户识别 Token 用途<br>**不参与权限验证** | `kd_xxx` 表示这是用于 Kodo 的 Token |
| **Token Scope**<br>（如 `["kodo:read", "kodo:write"]`） | 权限标记，验证时检查<br>**真正控制权限** | 该 Token 可以读写 Kodo |

**为什么会产生关联？**

在创建 Token 时，用户通过勾选业务和权限：
1. **系统根据选择自动设置 Scope**（如勾选 Kodo 读写 → Scope 包含 `kodo:read`, `kodo:write`）
2. **系统根据选择自动生成前缀**（如仅选 Kodo → 前缀为 `kd_`）

这样**一次操作同时完成了 Scope 设置和前缀生成**，简化了用户体验。但需要明确：
- ✅ 权限验证时检查的是 **Scope 字段**
- ❌ 前缀只是字符串标识，**不参与权限判断**

**示例说明**：

```json
{
  "token": "kd_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",  // 前缀 kd_ 仅助记
  "scope": ["kodo:read", "kodo:write"],           // 这个才控制权限
  "sub_account_uid": "sub_user_456"
}

// 验证时：
// 1. 检查 scope 是否包含所需权限 ✓
// 2. 前缀 kd_ 不参与验证 ✗
```

即使 Token 前缀是 `kd_`（暗示 Kodo），如果 Scope 中包含 `cdn:refresh`，该 Token 仍然可以访问 CDN 服务。

---

### 4.1 业界主流命名规范

| 厂商 | 前缀格式 | 示例 | 说明 |
|------|---------|------|------|
| **GitHub** | `ghp_`, `gho_`, `ghs_` | `ghp_abc123xyz` | 按 Token 类型区分 |
| **Stripe** | `sk_live_`, `sk_test_` | `sk_live_abc123` | 按环境区分 |
| **OpenAI** | `sk-proj-`, `sk-org-` | `sk-proj-abc123` | 按项目/组织区分 |
| **AWS** | `AKIA` (固定) | `AKIAIOSFODNN7EXAMPLE` | 固定前缀 |

### 4.2 七牛推荐方案：业务前缀

#### Token 格式

```
单业务 Token：<业务缩写>_<随机32位字符串>
多业务 Token：qn_<随机32位字符串>

示例：
kd_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6    # 仅 Kodo
cd_x1y2z3a4b5c6d7e8f9g0h1i2j3k4l5m6    # 仅 CDN
qn_m1n2o3p4q5r6s7t8u9v0w1x2y3z4a5b6    # Kodo + CDN + MikuStream（多业务）
```

#### 业务前缀定义

| 业务 | 前缀 | 说明 |
|------|------|------|
| **Kodo 对象存储** | `kd_` | Kodo |
| **CDN 内容分发** | `cd_` | CDN |
| **MikuStream 流媒体** | `mk_` | MikuStream |
| **LAS 实时音视频** | `la_` | LAS |
| **AI Inference** | `ai_` | AI Inference |
| **多业务组合** | `qn_` | 选择 2 个及以上业务时使用七牛统一前缀 |

#### 前缀规则

| 场景 | 前缀 | 示例 |
|------|------|------|
| 选择 1 个业务 | 该业务的两字母缩写 | 选择 Kodo → `kd_xxx` |
| 选择 2+ 个业务 | 七牛统一前缀 `qn_` | 选择 Kodo + CDN → `qn_xxx` |
| 前缀是否可修改 | **不可修改**（系统自动生成） | 保证安全性，便于泄漏检测 |

### 4.3 用户创建 Token 流程

**界面设计**：

![用户创建Token界面](pic/用户创建token2.png)

**动态前缀示例**：

| 用户选择的业务 | Token 前缀 | 完整 Token 示例 |
|--------------|----------|---------------|
| 仅 Kodo | `kd_` | `kd_a1b2c3d4e5f6...` |
| 仅 CDN | `cd_` | `cd_x1y2z3a4b5c6...` |
| Kodo + CDN | `qn_` | `qn_m1n2o3p4q5r6...` |
| Kodo + CDN + MikuStream | `qn_` | `qn_p7q8r9s0t1u2...` |

**创建后展示**：

```
Token 创建成功！请妥善保管，不会再次显示完整 Token。

Token 值：
┌─────────────────────────────────────────────────────┐
│ kd_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8             │
│ [复制]                                              │
└─────────────────────────────────────────────────────┘

Token 信息：
• 名称：production-api-token
• 业务：Kodo 对象存储（kd_）
• 权限：kodo:read, kodo:write
• 有效期至：2026-02-04 16:30:00
• 所属账号：sub_user_456
```

### 4.4 多业务一个 Token

**设计原则**：一个 Token 可以包含多个业务的 Scope

**示例**：
```json
{
  "token": "qn_a1b2c3d4e5f6...",
  "description": "全业务开发者 Token",
  "scope": [
    "kodo:read",
    "kodo:write",
    "cdn:refresh",
    "mikustream:stream:publish",
    "ai-inference:predict"
  ],
  "sub_account_uid": "sub_user_456",
  "expires_at": "2026-03-04T16:30:00Z"
}
```

**验证时自动匹配**：
- Kodo 服务验证：检查 `kodo:read` 或 `kodo:write`
- CDN 服务验证：检查 `cdn:refresh`
- MikuStream 服务验证：检查 `mikustream:stream:publish`

**优点**：
- ✅ 用户无需为每个业务创建单独 Token
- ✅ 减少 Token 管理复杂度
- ✅ 灵活组合不同业务的权限

---
