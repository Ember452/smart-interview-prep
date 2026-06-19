# 安全工程知识库

> 面试高频问题与答案要点。覆盖应用安全/认证授权/基础设施安全/合规方向。

---

## 1. 应用安全

### 1.1 Web 主要漏洞
- **SQL 注入防御** — 成因：拼接用户输入到 SQL（`"SELECT * FROM users WHERE name='" + input + "'"`→`' OR '1'='1`）。防御：参数化查询（PreparedStatement/#{}占位符，永远不拼接 SQL）、ORM（带参数绑定）、最小权限（DB 账号只给必需权限）
- **XSS 三兄弟** — 反射型（URL 参数直接回显，<script>alert(1)</script>）；存储型（恶意脚本存入 DB，其他用户打开页面触发）；DOM 型（前端 JS 取 URL hash 插入 DOM 不做转义）。防御：输出编码（HTML Entity/JS String 编码）、输入校验+过滤、CSP（Content Security Policy 白名单限制脚本来源）
- **CSRF 成因与防御** — 用户登录了银行 → 访问恶意网站 → 恶意网站悄悄发请求到银行（浏览器自动带 Cookie）→钱转了。防御：CSRF Token（服务端生成随机 token 放入表单/header，攻击者无法伪造）；SameSite Cookie（Strict/Lax 限制跨站发送 Cookie）；Referer/Origin 头校验
- **SSRF 成因与防御** — 服务端根据用户提供的 URL 发起请求（如 Webhook 回调、图片直传外部 URL）。攻击者传内网地址（http://169.254.169.254 AWS 元数据/内网服务）。防御：URL 白名单（只允许对外部域）；禁止内网 IP 段（127.0.0.0/8 10.0.0.0/8 等）；禁用 302 跟随跳转避免绕过检测

### 1.2 密码学基础
- **哈希算法安全特性** — 确定性（相同输入相同输出）、单向（不可逆）、抗碰撞（找不到两个不同输入产生相同哈希，MD5/SHA1 已被碰撞攻破→换 SHA-256/SHA-3）、雪崩效应（输入轻微变化输出剧烈变化）。密码存储不加盐的 SHA-256 仍危险（彩虹表反查）
- **密码存储最佳实践** — 加盐哈希：bcrypt（salt 自动管理/工作因子可调）/Argon2（2015 密码哈希竞赛冠军，内存硬开销抗GPU）。不加盐/直接用 SHA/MD5 是不合格的
- **对称 vs 非对称加密** — 对称（AES/ChaCha20）：加解密同一密钥、快、适合大批量数据加密。非对称（RSA/ECC）：公钥加密私钥解密、慢、适合密钥交换/签名。混合：用非对称传输对称密钥（TLS 握手），对称加密实际数据

---

## 2. 认证与授权

- **OAuth2.0 授权码流程** — 1. User 在第三方应用点"用 Google 登录"→2. 跳转到 Google 授权页面→3. 用户授权→4. Google 返回 Authorization Code→5. 第三方用 Code+ClientSecret 换 Access Token+Refresh Token。PKCE（Proof Key for Code Exchange）保护移动/SPA 无 client_secret 场景防授权码拦截
- **JWT 安全问题** — 不存敏感数据在 payload（payload 是 base64 解码可读）；token 过期用短 access token（15min）+ refresh token；token 注销需黑名单（Redis 存已注销 jti）；密钥泄露风险（对称 HMAC 密钥一旦泄露所有 token 可伪造→换 RS256 非对称签名，私钥只签发端知道）
- **RBAC vs ABAC** — RBAC 角色访问控制（管理员/编辑/只读，简单好维护权限集有限时）；ABAC 属性访问控制（用户属性+资源属性+环境属性+动作组合判定，灵活但策略复杂度高。主流：RBAC 主体+ABAC 补充细粒度策略）

---

## 3. 基础设施安全

- **密钥管理** — 密钥不硬编码/不放入代码仓库（立即泄露）。KMS/HSM 托管（AWS KMS/HashiCorp Vault）自动轮换、审计日志、访问控制；应用仅通过 SDK 间接调用密钥，不直接持有密钥值
- **零信任核心原则** — 永不默认信任（内网≠安全）、最小权限（仅给必要的）、持续验证（每次请求都要鉴权+设备+网络位置等动态上下文）、假设已被攻破（横向移动防护+微隔离）
- **容器安全基线** — 非 root 运行（USER 1000）、只读根文件系统（ReadOnlyRootFilesystem true）、禁用特权容器（privileged false）、资源限制防爆炸半径扩散（CPU/Memory limits）、镜像供应链安全（签名验证+SBOM 物料清单扫描已知漏洞）
- **mTLS（双向 TLS）** — 服务端验证客户端证书 + 客户端验证服务端证书，双向身份确认。适用场景：服务间通信（内部微服务互调不经过网关时）、Zero Trust 架构节点间通信。证书生命周期管理（签发/轮换/吊销 CRL/OCSP）是最大运维负担 → Service Mesh（Istio/Linkerd）自动证书管理简化
- **安全响应头** — HSTS（Strict-Transport-Security：强制 HTTPS 访问，max-age=31536000; includeSubDomains）；X-Frame-Options（DENY/SAMEORIGIN 防 Clickjacking）；CSP（Content-Security-Policy 限制脚本/样式/图片加载源，防 XSS/数据注入）；X-Content-Type-Options（nosniff 阻止浏览器 MIME 嗅探）

---

## 4. API 安全

- **API 鉴权最佳实践** — Bearer Token（OAuth2 短生命周期 Access Token，Header 传递）+ API Key（机器对机器调用，限 IP 白名单）+ mTLS（高安全场景双向证书）。永远不在 URL 中传递 token（会进入访问日志/浏览器历史）
- **API 限流与防护** — 单 IP/单用户 QPS 上限（令牌桶/滑动窗口）；请求体大小限制（防 DoS）；参数校验与类型强制（防类型混淆攻击）；批量接口单独限流（导出/批量查询比普通接口更严格）
- **敏感数据处理** — 日志脱敏（手机号/身份证/银行卡号/密码永不落盘，用掩码或 hash）；响应数据最小化（不返回不必要字段）；传输加密（HTTPS 强制+HSTS）；存储加密（敏感字段 AES 加密或应用层加密+密钥分离）