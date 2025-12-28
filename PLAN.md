# Claude Relay Service 项目计划

**日期**: 2025-12-28
**项目路径**: ~/claude-relay-service-original
**版本**: 1.1.249

---

# 第一部分：Fork 仓库与工作流设置

## 目标
将项目 Fork 到用户的 GitHub 账号 (xsha511)，建立长期维护的工作流程。

## 采用策略
**策略二：main 是你的版本**
- main 分支 = 用户的生产版本
- 包含用户所有的定制修改
- 定期从上游合并原作者的更新

## 执行步骤

### 步骤 1：Fork 仓库
```bash
gh repo fork Wei-Shaw/claude-relay-service --clone=false
```

### 步骤 2：修改本地 remote
```bash
# 将 origin 指向用户的 Fork
git remote set-url origin https://github.com/xsha511/claude-relay-service.git

# 添加上游 remote
git remote add upstream https://github.com/Wei-Shaw/claude-relay-service.git
```

### 步骤 3：处理现有本地改动
当前本地修改：
- `docker-compose.yml` (修改) - 部署配置
- `package-lock.json` (修改) - 依赖锁定
- `crs-compose.sh` (新增) - compose 生成脚本

处理方式：提交到 main 分支作为部署配置
```bash
git add .
git commit -m "Local deployment configuration"
git push origin main
```

### 步骤 4：验证设置
```bash
git remote -v
# 应显示：
# origin    https://github.com/xsha511/claude-relay-service.git
# upstream  https://github.com/Wei-Shaw/claude-relay-service.git
```

## 后续工作流程

### 用户提需求时
1. 用户描述需求
2. Claude 编写代码并提交到 main
3. 用户测试验证

### 同步上游更新时
1. 用户询问是否有更新
2. Claude 检查上游变更，分析内容
3. Claude 建议是否合并
4. 用户决定后，Claude 执行合并并解决冲突

---

# 第二部分：安全审计报告

## 执行摘要

对 Claude Relay Service 进行了全面的第三方安全审计，涵盖后门检测、认证授权、输入验证三大领域。

### 总体评估

| 类别 | 评级 | 说明 |
|------|------|------|
| 后门/恶意代码 | ✅ 安全 | 未发现后门、数据外泄或恶意代码 |
| 认证与授权 | ⚠️ 需改进 | 存在多个中高危漏洞 |
| 输入验证 | ⚠️ 中等 | 部分验证不完整 |

---

## 一、后门与恶意代码审计 ✅ 通过

### 检查结果

| 检查项 | 结果 | 说明 |
|--------|------|------|
| 动态代码执行 | ✅ 安全 | eval() 仅用于 Redis Lua 脚本 |
| 隐蔽网络请求 | ✅ 安全 | 仅连接官方 API (claude.ai, anthropic.com, google.com) |
| 硬编码后门地址 | ✅ 安全 | 无可疑 IP 或域名 |
| 隐藏管理员账号 | ✅ 安全 | 凭据随机生成，无预设后门 |
| 恶意加密/混淆 | ✅ 合法 | 加密仅用于保护敏感数据 |
| 系统命令执行 | ✅ 合法 | 仅用于进程管理脚本 |

### 结论
**项目是合法的 API 中转服务，未发现恶意代码或后门。**

---

## 二、认证与授权安全问题

### 严重级别问题 (6个)

| # | 问题 | 位置 | 风险 |
|---|------|------|------|
| 1 | **JWT 密钥管理不安全** | config/config.js:15 | 默认密钥硬编码，可伪造会话 |
| 2 | **API Key 从查询参数提取** | src/middleware/auth.js:165 | 日志泄露 API Key |
| 3 | **LDAP 密码明文存储** | src/services/ldapService.js | 凭据暴露 |
| 4 | **未使用密码哈希** | src/services/userService.js | 无本地密码验证 |
| 5 | **加密密钥管理不安全** | config/config.js:18 | 固定盐 'salt'，密钥明文存储 |
| 6 | **管理员初始凭据问题** | data/init.json | 明文存储 |

### 高危级别问题 (6个)

| # | 问题 | 位置 | 风险 |
|---|------|------|------|
| 7 | 未检查 JWT 签名算法 | src/middleware/auth.js | Session 固定攻击 |
| 8 | API Key 激活逻辑缺陷 | src/services/apiKeyService.js:236 | 无限延长有效期 |
| 9 | 管理员 Session 缺少完整性检查 | src/middleware/auth.js:1392 | 会话篡改 |
| 10 | OAuth State 验证缺陷 | src/routes/admin/claudeAccounts.js:72 | CSRF 攻击 |
| 11 | OAuth Code 未验证 | src/utils/oauthHelper.js:150 | Code 注入 |
| 12 | Socket 身份验证缺陷 | src/middleware/auth.js:869 | HTTP Keep-Alive 请求混淆 |

### 中危级别问题 (11个)

- Session 过期时间过长 (24小时)
- 密码强度验证不足 (仅检查长度)
- API Key 格式验证宽松
- API Key 哈希存储 (Rainbow table 风险)
- 管理员登录无暴力破解防护
- OAuth Session TTL 过长 (10分钟)
- 粘性会话设置不当
- Session 序列化不安全
- Client Restriction 可伪造 (User-Agent)
- 过度详细的错误消息
- CSP 策略过宽松 (unsafe-inline, unsafe-eval)

---

## 三、输入验证与注入漏洞

### 低风险 ✅

| 类别 | 评估 | 说明 |
|------|------|------|
| SQL 注入 | ✅ 无风险 | 使用 Redis，无 SQL 查询 |
| 命令注入 | ✅ 无风险 | 不使用 exec()/spawn() 执行用户输入 |
| 路径遍历 | ✅ 无风险 | 不处理文件系统路径 |

### 中等风险 ⚠️

| 类别 | 问题 | 位置 |
|------|------|------|
| XSS | sanitizeHtml() 定义但未广泛使用 | src/routes/admin/apiKeys.js |
| SSRF | 代理主机名未验证内网地址 | src/utils/proxyHelper.js |
| NoSQL 注入 | SCAN 命令通配符注入风险 | src/routes/admin/apiKeys.js:908 |

### 安全措施已实现 ✅

- Webhook URL 严格验证 (禁止内网/元数据服务)
- AES-256-CBC 加密敏感数据
- 安全头配置完整 (X-Frame-Options, X-XSS-Protection, HSTS)
- Socket 身份验证双重防护

---

## 四、关键漏洞详情

### 1. API Key 从 URL 查询参数提取 [严重]

```javascript
// src/middleware/auth.js:165-201
const candidates = [
  req.headers['x-api-key'],
  req.query?.key  // ❌ 严重风险：会出现在日志、浏览器历史、Referer 中
]
```

**影响**: API Key 可能通过服务器日志、代理日志、浏览器历史泄露

### 2. 加密密钥使用固定盐 [严重]

```javascript
// src/services/claudeAccountService.js
this.ENCRYPTION_SALT = 'salt'  // ❌ 固定盐值

// scripts/data-transfer-enhanced.js
const key = crypto.scryptSync(config.security.encryptionKey, 'salt', 32)
```

**影响**: 降低密钥派生函数安全性

### 3. 代理配置缺少内网验证 [中危]

```javascript
// src/utils/proxyHelper.js:90
const socksUrl = `socks5h://${auth}${proxy.host}:${proxy.port}`
// ❌ proxy.host 未验证是否为内网地址
```

**影响**: 可能被用于 SSRF 攻击内网服务

---

## 五、修复建议

### 立即修复 (严重级)

1. **移除查询参数 API Key 支持**
```javascript
// 删除 req.query?.key
const candidates = [
  req.headers['x-api-key'],
  req.headers['authorization']
]
```

2. **使用随机盐**
```javascript
const salt = crypto.randomBytes(16)
const key = crypto.scryptSync(password, salt, 32)
```

3. **加强密钥管理**
- 使用密钥管理服务 (AWS KMS, HashiCorp Vault)
- 强制修改默认密钥

### 高优先级

4. **添加代理地址验证**
```javascript
const dangerousHosts = ['localhost', '127.0.0.1', '169.254.169.254']
if (dangerousHosts.includes(proxy.host)) {
  throw new Error('Proxy cannot point to internal address')
}
```

5. **OAuth State 完整验证**
```javascript
if (req.query.state !== oauthSession.state) {
  throw new Error('Invalid OAuth state')
}
```

6. **管理员登录添加速率限制**

### 中等优先级

7. **全面应用 sanitizeHtml()**
8. **缩短 Session 过期时间至 2-4 小时**
9. **移除 CSP 中的 unsafe-inline**
10. **添加密码复杂性要求**

---

## 六、总结

### 安全优点 ✅

- 无后门或恶意代码
- 敏感数据 AES-256 加密存储
- Webhook URL 完整 SSRF 防护
- 完整的安全响应头
- Socket 身份验证创新防护

### 需要改进 ⚠️

- 认证系统存在多个漏洞
- 密钥管理需要加强
- 输入验证应用不一致
- CSP 策略过于宽松

### 风险评级

| 级别 | 数量 |
|------|------|
| 严重 | 6 |
| 高危 | 6 |
| 中危 | 11 |
| 低危 | 3 |

**建议**: 在生产环境使用前，优先修复所有严重和高危问题。

---

# 第三部分：安全改进计划（后续执行）

## 优先级排序

### P0 - 立即修复（影响安全性）
| 序号 | 问题 | 修改文件 | 工作量 |
|------|------|----------|--------|
| 1 | 移除 URL 查询参数 API Key | src/middleware/auth.js | 小 |
| 2 | 使用随机盐替换固定盐 | src/services/claudeAccountService.js, scripts/data-transfer-enhanced.js | 中 |
| 3 | 添加代理地址内网验证 | src/utils/proxyHelper.js | 小 |

### P1 - 高优先级（建议尽快修复）
| 序号 | 问题 | 修改文件 | 工作量 |
|------|------|----------|--------|
| 4 | OAuth State 完整验证 | src/routes/admin/claudeAccounts.js | 小 |
| 5 | 管理员登录速率限制 | src/routes/admin/auth.js | 中 |
| 6 | 缩短 Session 过期时间 | config/config.js | 小 |

### P2 - 中等优先级
| 序号 | 问题 | 修改文件 | 工作量 |
|------|------|----------|--------|
| 7 | 全面应用 sanitizeHtml() | 多个文件 | 中 |
| 8 | 加强 CSP 策略 | src/middleware/security.js | 中 |
| 9 | 密码复杂性要求 | src/services/userService.js | 小 |

## 执行方式
用户可以随时要求 Claude 执行以上任何改进，例如：
- "帮我修复 P0 的问题"
- "修复 API Key 查询参数的问题"
- "按优先级修复所有安全问题"

---

*计划更新 - 2025-12-28*
