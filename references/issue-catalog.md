# 后端代码审查问题码表

## 一、文档定位

本文是 AI 代码扫描的**问题分类字典（码表）**。AI 在触发代码审查时，对照本表识别问题、分配唯一编号、映射级别、执行去重与合并。

- 评分规则、级别定义、报告输出参见 `scoring-model.md`，两文档配合使用。
- 每条问题拥有稳定 `id`（如 `SEC-001`），AI 命中后必须输出该 `id`，作为跨报告的稳定引用。
- 本表覆盖后端为主（Java / BladeX 场景），兼顾前端与项目通用项。

## 二、编号方案

| 前缀 | 分类 | 说明 |
| --- | --- | --- |
| SEC | 安全与漏洞 | 注入、密钥、反序列化、路径穿越、鉴权、越权 |
| PRV | 隐私与合规 | 脱敏、审计、留存、跨境、隐私最小化 |
| BIZ | 业务逻辑正确性 | 条件、计算、金额、时间、状态、幂等、事务、回滚 |
| ROB | 健壮性 | NPE、越界、异常、超时、重试、限流、资源释放 |
| CON | 并发 | 竞态、锁、线程池、非线程安全 |
| CACHE | 缓存 | 穿透、击穿、雪崩、一致性 |
| PERF | 性能 | N+1、循环远程调用、慢查询、无分页 |
| LOG | 日志 | 敏感落日志、吞异常、级别、上下文 |
| CFG | 配置 | 硬编码、环境混用、开关残留 |
| DEP | 依赖 | 已知 CVE、雪花版本、冗余依赖 |
| API | 接口 | 参数校验、错误码、版本兼容、幂等键 |
| QUAL | 代码质量 | 复制粘贴、死代码、复杂度、命名 |
| TEST | 测试 | 覆盖不足、边界用例缺失 |

编号一旦分配不再复用；新增条目末尾追加，废弃条目标记 `deprecated: true` 但保留编号。

## 三、条目字段结构

每条问题以列表项组织，字段固定如下（AI 输出时必须按同结构给出命中信息）：

```yaml
- id: SEC-001                                    # 唯一编号，稳定不变
  title: 明文密码/Token/AccessKey/SecretKey 硬编码 # 中文标题
  level: 阻断                                    # 建议级别：阻断/严重/主要/次要/提示；AI 可依上下文在 ±1 级内调整
  rootCauseKey: SEC-001:credential-hardcoded    # 去重合并键，AI 必须输出
  signals:                                       # 识别线索，仅供参考，AI 可依上下文自行补充
    regex:
      - (?i)(password|passwd|token|access[_-]?key|secret[_-]?key|api[_-]?key)\s*[:=]\s*["'][^"'${}]{4,}["']
    keywords: [ "new SecretKeySpec(\"", "BCrypt.hashpw(\"" ]
    whitelist: [ "${ENV_", "@Value(\"${", "Nacos 加密配置" ]
  description: 有明确证据证明生产真实凭证、外部可访问凭证或高权限凭证被硬编码，泄露即事故。
  fixHint: 迁移到配置中心或环境变量，敏感项统一加密下发。
  downgradeToInfo:
    - "Druid monitor、Actuator、Swagger、测试工具等内网监控 / 开发测试账号出现在 yml 中，且无证据表明为生产真实高权限凭证或外部可访问。"
    - "application-dev.yml、application-test.yml、local.yml、示例配置或模板配置中的默认账号密码，未证明会进入生产环境。"
    - "仅凭字段名 password、secret、accessKey 命中，但缺少 profile、生效环境、外部暴露范围或真实凭证证据。"
```

- **id**：`前缀-三位数字`，AI 输出的每个问题必须携带。
- **title**：中文，简短一句。
- **level**：建议级别，五档：阻断/严重/主要/次要/提示。
- **rootCauseKey**：去重合并主键，格式 `<id>:<归一化根因特征>`；AI 必须输出。
- **signals**：识别线索，`regex` / `keywords` / `whitelist` 三选；**仅供参考**，AI 应结合语义判断，不得只做机械匹配。
- **description**：一句话说明 WHY。
- **fixHint**：一句修复方向（≤ 30 字）。

## 四、去重与合并规则

AI 每次扫描按下列规则计算与合并，保证多次扫描同一份代码结果稳定。

### 4.1 三层键定义

```
rootCauseKey        = "<id>:<归一化根因特征>"   例：SEC-001:credential-hardcoded
locationFingerprint = <文件路径> + "#" + <类名/方法名>   （不含行号）
descriptionNorm     = 去除变量名、行号、字面量后取前 8 位哈希
```

### 4.2 合并与扣分规则

| 编号 | 规则 | 说明 |
| --- | --- | --- |
| R1 | 根因合并 | 同一 `rootCauseKey` 只算 1 条问题；`locations[]` 记录所有 `locationFingerprint`，扣分不因数量线性叠加 |
| R2 | 影响面上浮 | 同一 `rootCauseKey` 命中位置 ≥ 3 处，可在该 id 建议级别范围内上浮 1 档（如 主要 → 严重），不得超过 id 允许上限 |
| R3 | 描述归一 | `rootCauseKey` 相同、`descriptionNorm` 不同的多次记录，视为同一问题，不重复扣分 |
| R4 | 多类别命中 | 同一位置同时命中多个 id，按最高级别 id 扣分，其他 id 仅列为「关联问题」，不再重复扣 |
| R5 | 扫描幂等 | 同一份代码重复扫描，`rootCauseKey` 集合应完全一致；不一致视为扫描不稳定，需回归 |
| R6 | 证据不足 | 无法明确判定的疑似项，标记为「待确认」，不进入扣分池 |
| R7 | 白名单豁免 | 命中条目 `signals.whitelist` 时不入问题池 |
| R8 | 级别冲突 | 同 id 在不同证据下级别不一致时，取该次扫描中的最高级别 |

### 4.3 AI 输出结构约定

AI 在报告中每个问题至少输出：

```yaml
id: SEC-002
rootCauseKey: SEC-002:sql-injection-string-concat
level: 阻断
locations:
  - src/main/java/.../OrderMapper.java#OrderMapper.listByCondition
  - src/main/java/.../ReportMapper.java#ReportMapper.export
occurrenceCount: 2
mergedFrom: 5           # 归一前的原始命中次数
evidence: <脱敏代码片段>
fixHint: 使用 #{} 参数化查询，禁止 ${} 拼接
```


### 4.4 误报排除、降级与证据边界

- 命中安全类关键词但缺少上下文证据时，不得直接按阻断、严重或主要扣分；应先降级为“提示”或标记为“待确认”。
- **租户相关业务边界已从代码审查安全问题中移除**：不得仅因 `tenantId`、租户上下文、跨租户参数、租户字段透传、租户过滤条件缺失、`@NonDS` 使用、`TenantUtil.setTenantId`、租户上下文未恢复 / 未清理、租户业务策略与通用安全习惯不一致，或组织 / 用户 / 部门 / 角色等查询没有出现 `tenant_id` 条件，就输出租户类安全结论；不得扣分，不得列为阻断原因、必须修复项或复审要求，默认不进入问题清单。
- FeignClient / RPC / 内部 Client 声明的接口通常视为内部服务调用契约；仅凭请求体中存在 `tenantId`、`initiator`、`userId`、`ownerId` 等字段，不得判定为高危越权、租户类安全问题或身份伪造。
- `tenantId`、租户上下文、租户查询条件、租户字段是否参与审批、授权、跨租户判断，均按业务规则处理，默认不输出为代码审查问题；`initiator`、`userId`、`ownerId` 等非租户字段如果只是业务参数、内部契约或业务边界定义问题，也不作为安全漏洞扣分，除非用户明确要求输出业务建议，否则不进入问题清单，确需提醒时只能作为“提示”级且不扣分的业务说明。
- 只有能证明非租户类身份字段或资源主键来自外部不可信入口，并且可直接绕过授权、冒充他人或修改他人资源，且缺少项目统一鉴权 / 网关 / 拦截器 / 上下文校验保护时，才可按对应安全问题升级。
---

## 五、码表条目

### 5.1 SEC · 安全与漏洞

```yaml
- id: SEC-001
  title: 明文密码/Token/AccessKey/SecretKey 硬编码
  level: 阻断
  rootCauseKey: SEC-001:credential-hardcoded
  signals:
    regex:
      - (?i)(password|passwd|token|access[_-]?key|secret[_-]?key|api[_-]?key)\s*[:=]\s*["'][^"'${}]{4,}["']
    keywords: [ "new SecretKeySpec(\"", "BCrypt.hashpw(\"" ]
    whitelist: [ "${ENV_", "@Value(\"${", "Nacos 加密配置" ]
  description: 有明确证据证明生产真实凭证、外部可访问凭证或高权限凭证被硬编码，泄露即事故。
  fixHint: 迁移到配置中心或环境变量，敏感项统一加密下发。
  downgradeToInfo:
    - "Druid monitor、Actuator、Swagger、测试工具等内网监控 / 开发测试账号出现在 yml 中，且无证据表明为生产真实高权限凭证或外部可访问。"
    - "application-dev.yml、application-test.yml、local.yml、示例配置或模板配置中的默认账号密码，未证明会进入生产环境。"
    - "仅凭字段名 password、secret、accessKey 命中，但缺少 profile、生效环境、外部暴露范围或真实凭证证据。"

- id: SEC-002
  title: SQL 注入
  level: 阻断
  rootCauseKey: SEC-002:sql-injection-string-concat
  signals:
    regex:
      - \$\{[^}]+\}\s*(WHERE|AND|OR|ORDER|GROUP|LIMIT)
      - (SELECT|UPDATE|DELETE|INSERT)[\s\S]{0,200}?\+\s*\w+
    keywords: [ "Statement.execute", "createStatement()", "${", "concat(" ]
    whitelist: [ "#{", "MyBatis @Param", "Prepared" ]
  description: SQL 通过字符串拼接或 ${} 传入外部变量，可被注入。
  fixHint: 一律 #{} 参数化；动态列名用白名单枚举校验。

- id: SEC-003
  title: 命令注入
  level: 阻断
  rootCauseKey: SEC-003:command-injection
  signals:
    keywords: [ "Runtime.getRuntime().exec(", "ProcessBuilder(", "new ProcessBuilder" ]
    regex:
      - exec\s*\(\s*[^)]*\+
  description: Runtime/ProcessBuilder 拼接外部参数导致 shell 注入。
  fixHint: 使用数组形式传参，参数白名单校验，禁止 shell 展开。

- id: SEC-004
  title: 表达式注入（SpEL/OGNL/EL/Groovy）
  level: 阻断
  rootCauseKey: SEC-004:expression-injection
  signals:
    keywords: [ "SpelExpressionParser", "Ognl.parseExpression", "GroovyShell", "ScriptEngine.eval(" ]
  description: 用户输入进入表达式引擎，可执行任意代码。
  fixHint: 禁止执行用户输入表达式；必要时用 SimpleEvaluationContext + 白名单。

- id: SEC-005
  title: SSRF / 任意请求转发
  level: 严重
  rootCauseKey: SEC-005:ssrf
  signals:
    keywords: [ "RestTemplate.getForObject(", "HttpClient.execute(", "URL(", "OkHttpClient" ]
    regex:
      - \.(getForObject|exchange|execute|newCall)\s*\(\s*[^)]*(?:request|params|url)
  description: 服务器根据外部输入拼接 URL 发起请求，可访问内网。
  fixHint: 目标域名/IP 白名单，禁止 127/10/192.168/169.254 内网段。

- id: SEC-006
  title: XSS 输出编码缺失
  level: 严重
  rootCauseKey: SEC-006:xss-missing-encoding
  signals:
    keywords: [ "innerHTML", "v-html", "dangerouslySetInnerHTML", "th:utext" ]
    whitelist: [ "DOMPurify.sanitize", "HtmlUtils.htmlEscape", "th:text" ]
  description: 用户内容未转义直接渲染到 HTML/JS 上下文。
  fixHint: 用 htmlEscape/DOMPurify；富文本走白名单过滤器。

- id: SEC-007
  title: CSRF 防护缺失
  level: 严重
  rootCauseKey: SEC-007:csrf-missing
  signals:
    keywords: [ ".csrf().disable()", "csrfTokenRepository(null)" ]
  description: 状态变更接口未启用 CSRF Token 或 SameSite Cookie 策略。
  fixHint: 启用 Spring Security CSRF；Cookie 加 SameSite=Lax/Strict。

- id: SEC-008
  title: 路径穿越
  level: 阻断
  rootCauseKey: SEC-008:path-traversal
  signals:
    regex:
      - new\s+File\s*\([^)]*(?:request|param|input|fileName)
      - Paths\.get\s*\([^)]*(?:request|param|input)
    keywords: [ "../", "..\\\\", "getCanonicalPath" ]
  description: 使用未校验的文件名/路径参数访问本地文件。
  fixHint: 归一化后必须 startsWith(允许根目录)，禁止 ../。

- id: SEC-009
  title: 任意文件上传/下载
  level: 阻断
  rootCauseKey: SEC-009:arbitrary-file-upload-download
  signals:
    keywords: [ "MultipartFile", "transferTo(", "FileCopyUtils.copy(" ]
  description: 上传未限制类型/大小/存储路径，或下载未校验归属。
  fixHint: 白名单扩展名 + MIME 校验，独立存储目录，下载做归属校验。

- id: SEC-010
  title: 不安全反序列化
  level: 阻断
  rootCauseKey: SEC-010:unsafe-deserialization
  signals:
    keywords: [ "ObjectInputStream.readObject", "XMLDecoder", "Yaml.load(", "SnakeYaml", "Hessian" ]
    whitelist: [ "Yaml.loadAs(", "SafeConstructor" ]
  description: 反序列化外部数据可触发 gadget 链 RCE。
  fixHint: 禁止反序列化不可信输入；用 JSON + 白名单类型。

- id: SEC-011
  title: XXE / XML 外部实体
  level: 严重
  rootCauseKey: SEC-011:xxe
  signals:
    keywords: [ "DocumentBuilderFactory.newInstance()", "SAXParserFactory", "XMLInputFactory" ]
    whitelist: [ "setFeature(\"http://apache.org/xml/features/disallow-doctype-decl\", true)" ]
  description: XML 解析器未关闭 DTD 与外部实体加载。
  fixHint: disallow-doctype-decl=true；关闭 external-general-entities。

- id: SEC-012
  title: 权限绕过 / 未鉴权接口
  level: 阻断
  rootCauseKey: SEC-012:auth-bypass
  signals:
    keywords: [ "@ApiOperationSupport", "permitAll()", "SkipAuth" ]
    regex:
      - @(RequestMapping|GetMapping|PostMapping)[^\n]*\n\s*(public\s+)?[^{]*\{
  description: 敏感接口未加鉴权注解，或用 permitAll() 打开。
  fixHint: 默认拒绝，白名单放行；鉴权注解走统一切面。

- id: SEC-013
  title: 非租户类越权访问 / 非租户类数据归属校验缺失
  level: 阻断
  rootCauseKey: SEC-013:idor-object-ownership
  signals:
    regex:
      - selectById\s*\(\s*(?:request|param|dto)\.
    keywords: [ "getById(id)", "removeById(" ]
  description: 非租户类资源查询、更新或删除只按外部可控主键执行，且有明确证据证明可绕过授权访问或修改他人资源。
  fixHint: 结合统一鉴权、当前用户上下文和资源归属校验。
  excludeFromReview:
    - "租户字段、租户过滤条件、tenantId 透传、@NonDS 使用、TenantUtil.setTenantId、租户上下文恢复 / 清理、跨租户业务判断、租户上下文切换、组织 / 用户 / 部门 / 角色查询条件等内容，不再作为本码表安全问题输出或扣分，默认不进入问题清单。"
    - "问题摘要、分类、阻断原因、必须修复项和复审要求中不得使用租户类安全结论；除非用户明确要求业务建议，否则不要把租户相关内容写入问题清单。"
  downgradeToInfo:
    - "FeignClient / RPC / 内部 Client 接口中仅声明或透传 initiator、userId、ownerId 等字段，未证明外部可控或调用方不可信。"
    - "initiator、userId、ownerId 等字段参与审批、授权或人员归属判断，但问题本质只是边界定义不明确、内部契约语义不清、上下文缺失或缺少补充说明时，只按提示处理。"
    - "只看到 DTO / Request 字段传递，未看到按该字段直接越权查询、修改、删除、审批或冒充操作。"
- id: SEC-014
  title: 弱加密 / 弱哈希算法
  level: 严重
  rootCauseKey: SEC-014:weak-crypto
  signals:
    keywords: [ "MD5", "SHA1", "DES", "RC4", "ECB", "PKCS1Padding" ]
    whitelist: [ "MD5 仅作校验非敏感", "SHA-256", "AES/GCM" ]
  description: 使用已破解或不适合场景的算法。
  fixHint: 密码用 BCrypt/Argon2；对称加密用 AES/GCM；签名用 SHA-256+。

- id: SEC-015
  title: 不安全随机数
  level: 主要
  rootCauseKey: SEC-015:insecure-random
  signals:
    keywords: [ "new Random(", "Math.random()" ]
    whitelist: [ "SecureRandom" ]
  description: 涉密场景（Token/密码/验证码）使用 java.util.Random。
  fixHint: 安全场景一律 SecureRandom。

- id: SEC-016
  title: Open Redirect
  level: 严重
  rootCauseKey: SEC-016:open-redirect
  signals:
    keywords: [ "sendRedirect(", "redirect:", "Location header" ]
    regex:
      - sendRedirect\s*\(\s*[^)]*(?:request|param|url)
  description: 重定向目标来自用户输入且未校验域名。
  fixHint: 白名单校验目标 host；对相对路径做归一化。

- id: SEC-017
  title: JWT / Session 处理不当
  level: 严重
  rootCauseKey: SEC-017:jwt-session-mishandling
  signals:
    keywords: [ "alg\":\"none\"", "verify=False", "SecretKeySpec(\"secret\"" ]
  description: JWT 允许 none 算法、密钥固定或过期时间过长。
  fixHint: 强制 HS256/RS256，密钥来自配置中心，合理过期 + 刷新。
```

### 5.2 PRV · 隐私与合规

```yaml
- id: PRV-001
  title: 个人信息 / 敏感数据未脱敏
  level: 严重
  rootCauseKey: PRV-001:pii-not-masked
  signals:
    keywords: [ "身份证", "手机号", "phone", "idCard", "bankCard", "log.info", "System.out.println" ]
    whitelist: [ "DesensitizedUtil", "@Sensitive", "MaskUtil" ]
  description: 姓名、身份证、手机号、银行卡等未脱敏输出到日志或返回给前端。
  fixHint: 输出前统一 MaskUtil；日志级别拉高时也不得原文落盘。

- id: PRV-002
  title: 审计日志缺失
  level: 主要
  rootCauseKey: PRV-002:audit-missing
  signals:
    keywords: [ "@Log", "@OperationLog", "AuditLog" ]
  description: 关键操作（登录、授权、资金、数据导出）无审计记录。
  fixHint: 关键接口加统一审计切面，记录操作人/对象/前后值/来源 IP。

- id: PRV-003
  title: 数据留存 / 删除策略缺失
  level: 主要
  rootCauseKey: PRV-003:retention-policy-missing
  signals:
    keywords: [ "TTL", "expireAt", "logic delete", "deleted=1" ]
  description: 用户数据无留存期限或删除后仍可追溯到明文。
  fixHint: 明确留存期限；到期物理删除或不可逆脱敏。

- id: PRV-004
  title: 敏感接口暴露 / 隐私接口未做权限约束
  level: 严重
  rootCauseKey: PRV-004:privacy-api-exposed
  signals:
    keywords: [ "/export", "/download", "/list", "@GetMapping" ]
  description: 导出、批量查询、隐私字段接口对外开放且无权限约束。
  fixHint: 敏感接口默认关闭，需白名单开通 + 二次授权 + 审计。
```

### 5.3 BIZ · 业务逻辑正确性

```yaml
- id: BIZ-001
  title: 条件分支错误
  level: 主要
  rootCauseKey: BIZ-001:branch-logic-error
  signals:
    regex:
      - if\s*\([^)]*==\s*null[^)]*&&
      - \s(&&|\|\|)\s*!
    keywords: [ "TODO", "FIXME" ]
  description: 逻辑运算符使用错误、null 判定顺序颠倒、分支缺失。
  fixHint: 明确前置条件；补充 else / default 分支；用测试用例覆盖每条分支。

- id: BIZ-002
  title: 计算逻辑错误
  level: 主要
  rootCauseKey: BIZ-002:computation-error
  signals:
    regex:
      - /\s*0(?!\.\d)
      - %\s*0(?!\.\d)
  description: 常量误算、除零、单位换算错误、四舍五入误用。
  fixHint: 提炼公式常量为具名量；除零前置校验；四舍五入用 HALF_EVEN 或明确规则。

- id: BIZ-003
  title: 金额精度错误（Double / Float 误用）
  level: 严重
  rootCauseKey: BIZ-003:money-precision
  signals:
    regex:
      - (double|float|Double|Float)\s+(price|amount|money|fee|total|cost)
    keywords: [ ".doubleValue()", ".floatValue()" ]
    whitelist: [ "BigDecimal", "MoneyUtil" ]
  description: 金额使用 double/float，或 BigDecimal 未指定 scale/rounding。
  fixHint: 金额一律 BigDecimal，new BigDecimal(String)；显式 scale 与 rounding。

- id: BIZ-004
  title: 时间 / 时区 / 日期边界错误
  level: 主要
  rootCauseKey: BIZ-004:time-timezone-boundary
  signals:
    keywords: [ "new Date(", "SimpleDateFormat", "System.currentTimeMillis" ]
    whitelist: [ "ZonedDateTime", "Instant", "LocalDateTime + ZoneId" ]
  description: 混用本地时间与 UTC，SimpleDateFormat 非线程安全，跨月/闰年错误。
  fixHint: 用 java.time；对外统一 UTC，展示层再本地化；日期区间左闭右开。

- id: BIZ-005
  title: 数量 / 边界处理错误
  level: 主要
  rootCauseKey: BIZ-005:boundary-error
  signals:
    regex:
      - <\s*=\s*size\s*\(\s*\)
      - i\s*<\s*list\.size\s*\(\s*\)\s*\+
  description: off-by-one、负数量、上限未校验、空集合边界错。
  fixHint: 明确闭区间；负数、零、超大值单独用例；空集合走 fast-path。

- id: BIZ-006
  title: 幂等性缺失
  level: 严重
  rootCauseKey: BIZ-006:idempotency-missing
  signals:
    keywords: [ "@PostMapping", "insert", "create", "pay", "refund" ]
    whitelist: [ "requestId", "idempotencyKey", "唯一约束", "分布式锁" ]
  description: 支付、下单、扣款、消息消费等写接口未做幂等控制。
  fixHint: 幂等键（requestId/bizNo）+ 唯一索引 / 状态机 / 分布式锁三选。

- id: BIZ-007
  title: 事务边界错误
  level: 严重
  rootCauseKey: BIZ-007:transaction-boundary
  signals:
    keywords: [ "@Transactional" ]
    regex:
      - @Transactional[^)]*\)[\s\S]{0,200}RemoteCall|HttpClient|kafkaTemplate
  description: 事务方法内包含远程调用、MQ 发送、长耗时操作；或 @Transactional 未生效（this. 调用）。
  fixHint: 事务只做本地 DB；远程/MQ 出事务；避免 this. 内部调用绕过代理。

- id: BIZ-008
  title: 回滚逻辑错误
  level: 严重
  rootCauseKey: BIZ-008:rollback-logic
  signals:
    keywords: [ "rollbackFor = Exception.class", "TransactionAspectSupport.currentTransactionStatus", "catch (Exception" ]
  description: 只声明 rollbackFor=RuntimeException；catch 后未 rollback；补偿逻辑遗漏。
  fixHint: rollbackFor=Exception.class 或明确异常；补偿走 Saga/TCC。

- id: BIZ-009
  title: 状态流转错误
  level: 主要
  rootCauseKey: BIZ-009:state-transition
  signals:
    keywords: [ "status", "state", "updateStatus" ]
  description: 状态跳变未校验前置状态、允许非法回退、缺终态判定。
  fixHint: 用状态机集中管理 allowedTransitions；UPDATE 带 WHERE status=? 前置校验。

- id: BIZ-010
  title: 参数组合不合法未处理
  level: 主要
  rootCauseKey: BIZ-010:param-combination-invalid
  signals:
    keywords: [ "@Valid", "BindingResult" ]
  description: 单字段合法但组合语义非法（如 startTime > endTime、pageSize 与 pageNum 边界）未校验。
  fixHint: 增加跨字段校验器（@AssertTrue / ConstraintValidator）。

- id: BIZ-011
  title: 唯一性约束缺失
  level: 主要
  rootCauseKey: BIZ-011:uniqueness-missing
  signals:
    keywords: [ "insert", "save(" ]
  description: 业务唯一（手机号、bizNo）仅靠应用先查后写，无 DB 唯一索引。
  fixHint: 关键业务唯一必须 DB 唯一索引，应用层捕获唯一冲突转业务码。
```

### 5.4 ROB · 健壮性

```yaml
- id: ROB-001
  title: 空值 / NPE
  level: 主要
  rootCauseKey: ROB-001:npe
  signals:
    keywords: [ ".get(", ".stream()", ".getName()" ]
    whitelist: [ "Optional", "@NotNull", "Objects.requireNonNull" ]
  description: 未判空直接调用属性 / 方法；集合返回 null 而非空集合。
  fixHint: 集合返回空实现；对可空对象用 Optional 或前置校验。

- id: ROB-002
  title: 数组 / 集合越界
  level: 主要
  rootCauseKey: ROB-002:index-out-of-bounds
  signals:
    regex:
      - \.get\s*\(\s*\d+\s*\)
      - \[\s*\d+\s*\]
  description: 硬编码下标或 size-1 未防空集合。
  fixHint: 访问前判 size；避免硬编码下标；用迭代器 / stream。

- id: ROB-003
  title: 类型转换异常
  level: 主要
  rootCauseKey: ROB-003:cast-exception
  signals:
    keywords: [ "Integer.parseInt(", "Long.parseLong(", "(Xxx)" ]
  description: 强制类型转换无 instanceof 判断；字符串转数字未捕获。
  fixHint: 转换前 instanceof / NumberUtils.isDigits；提供默认值。

- id: ROB-004
  title: 缺少异常兜底
  level: 主要
  rootCauseKey: ROB-004:missing-exception-handler
  signals:
    keywords: [ "throws Exception", "// TODO handle" ]
  description: 关键路径无异常兜底或吞异常继续执行导致数据不一致。
  fixHint: 明确失败策略：抛出 / 兜底默认值 / 补偿；全局异常处理器兜底。

- id: ROB-005
  title: 超时控制缺失
  level: 严重
  rootCauseKey: ROB-005:timeout-missing
  signals:
    keywords: [ "RestTemplate", "HttpClient", "FeignClient", "Jedis" ]
    whitelist: [ "setConnectTimeout", "setReadTimeout", "connectTimeout=", "requestTimeout" ]
  description: HTTP / RPC / DB / Redis 调用未设置连接与读取超时。
  fixHint: 客户端全局默认超时；关键接口显式覆盖，慢下游必须熔断。

- id: ROB-006
  title: 重试边界缺失 / 无限重试
  level: 严重
  rootCauseKey: ROB-006:retry-boundary
  signals:
    keywords: [ "@Retryable", "while(true)", "retryTemplate.execute" ]
  description: 重试无最大次数、无退避、无幂等前提，导致雪崩或重复副作用。
  fixHint: 明确 maxAttempts + 指数退避；仅对幂等操作重试。

- id: ROB-007
  title: 限流 / 熔断缺失
  level: 严重
  rootCauseKey: ROB-007:rate-limit-circuit-breaker
  signals:
    keywords: [ "Sentinel", "Resilience4j", "@SentinelResource", "RateLimiter" ]
  description: 高危接口、外部依赖调用未加限流 / 熔断。
  fixHint: 网关或方法级加 Sentinel/Resilience4j；下游异常隔离。

- id: ROB-008
  title: 资源未释放（IO / Connection / Thread）
  level: 主要
  rootCauseKey: ROB-008:resource-leak
  signals:
    keywords: [ "new FileInputStream(", "getConnection(", "new Thread(" ]
    whitelist: [ "try-with-resources", "@Cleanup" ]
  description: 未使用 try-with-resources；异常路径未 close；线程池未 shutdown。
  fixHint: 一律 try-with-resources；Executor 集中管理 + 关闭钩子。

- id: ROB-009
  title: 分页缺失 / 批量操作无上限
  level: 严重
  rootCauseKey: ROB-009:pagination-batch-unbounded
  signals:
    keywords: [ "selectList(", "findAll()", "batchInsert" ]
  description: 查询无分页、批量操作无最大条数、导出无限制。
  fixHint: 强制分页；批量 ≤ 1000/批；导出走异步 + 记录中心。
```

### 5.5 CON · 并发

```yaml
- id: CON-001
  title: 并发更新覆盖（Lost Update）
  level: 严重
  rootCauseKey: CON-001:lost-update
  signals:
    keywords: [ "updateById", "set balance = " ]
    whitelist: [ "version", "@Version", "SELECT ... FOR UPDATE" ]
  description: 先查后写未加乐观锁或行锁，导致并发覆盖。
  fixHint: 加 version 乐观锁或 SELECT FOR UPDATE；关键更新原子 SQL。

- id: CON-002
  title: 分布式锁误用
  level: 严重
  rootCauseKey: CON-002:distributed-lock-misuse
  signals:
    keywords: [ "redisTemplate.opsForValue().setIfAbsent", "Redisson", "tryLock" ]
    whitelist: [ "lockWatchdog", "leaseTime", "唯一 requestId" ]
  description: setnx 未设置过期、无锁续约、释放锁未判断持有者。
  fixHint: 用 Redisson 或 setnx+expire 原子；释放需比对 requestId。

- id: CON-003
  title: 竞态条件（Race Condition）
  level: 严重
  rootCauseKey: CON-003:race-condition
  signals:
    keywords: [ "if (map.get(", "containsKey", "putIfAbsent" ]
  description: check-then-act 未原子化，并发下产生不一致。
  fixHint: 用 computeIfAbsent / AtomicXxx / 锁；或改为唯一约束兜底。

- id: CON-004
  title: 线程池 / 队列配置不当
  level: 严重
  rootCauseKey: CON-004:thread-pool-misconfig
  signals:
    keywords: [ "Executors.newFixedThreadPool", "Executors.newCachedThreadPool", "LinkedBlockingQueue()" ]
    whitelist: [ "ThreadPoolExecutor(", "有界队列" ]
  description: 使用 Executors 工厂类导致无界队列 / 无界线程数，OOM 风险。
  fixHint: 手工 ThreadPoolExecutor，有界队列 + 拒绝策略 + 线程命名。

- id: CON-005
  title: 非线程安全类误用
  level: 主要
  rootCauseKey: CON-005:non-thread-safe-misuse
  signals:
    keywords: [ "SimpleDateFormat", "HashMap", "ArrayList" ]
    whitelist: [ "ThreadLocal", "ConcurrentHashMap", "DateTimeFormatter" ]
  description: 共享 SimpleDateFormat / HashMap 等在并发下出错。
  fixHint: 用线程安全替代类；或 ThreadLocal 包裹。

- id: CON-006
  title: 死锁 / 锁顺序不一致
  level: 严重
  rootCauseKey: CON-006:deadlock
  signals:
    keywords: [ "synchronized (a)", "lock.lock()", "@Transactional" ]
  description: 多资源加锁顺序不一致，或事务持锁等待外部资源。
  fixHint: 全局锁顺序约定；缩短事务持锁时间；提前排序主键。
```

### 5.6 CACHE · 缓存

```yaml
- id: CACHE-001
  title: 缓存穿透
  level: 严重
  rootCauseKey: CACHE-001:cache-penetration
  signals:
    keywords: [ "redisTemplate.opsForValue().get(", "cache.get(" ]
    whitelist: [ "空值缓存", "BloomFilter" ]
  description: 恶意查询不存在的 key，穿透打到 DB。
  fixHint: 空值也缓存（短 TTL）；或前置 BloomFilter。

- id: CACHE-002
  title: 缓存击穿
  level: 严重
  rootCauseKey: CACHE-002:cache-breakdown
  signals:
    keywords: [ "get 缓存失败即查 DB" ]
    whitelist: [ "singleflight", "分布式锁", "永不过期 + 异步刷新" ]
  description: 热 key 过期瞬间大量并发直击 DB。
  fixHint: 互斥锁重建缓存；热点 key 永不过期 + 异步刷新。

- id: CACHE-003
  title: 缓存雪崩
  level: 严重
  rootCauseKey: CACHE-003:cache-avalanche
  signals:
    keywords: [ "expire(", "TTL" ]
    whitelist: [ "TTL + 随机抖动" ]
  description: 大量 key 同一时间失效，DB 被打爆。
  fixHint: TTL 加随机抖动；热数据分级 TTL；集群多副本。

- id: CACHE-004
  title: 缓存一致性 / 更新时序错误
  level: 严重
  rootCauseKey: CACHE-004:cache-consistency
  signals:
    keywords: [ "先更新缓存后更新 DB", "@CachePut", "@CacheEvict" ]
    whitelist: [ "先更新 DB 后删除缓存", "延迟双删" ]
  description: 缓存与 DB 更新顺序错，或删除缓存失败无补偿。
  fixHint: 先写 DB 后删缓存 + 消息补偿；对强一致场景走事务外发件箱。

- id: CACHE-005
  title: 缓存无过期时间
  level: 主要
  rootCauseKey: CACHE-005:cache-no-ttl
  signals:
    regex:
      - opsForValue\(\)\.set\s*\([^,]+,[^,]+\)\s*;
    keywords: [ "SET key value" ]
  description: 缓存写入未指定 TTL，Redis 内存无限增长。
  fixHint: 一律指定 TTL；持久数据放独立 db 或走单独策略。
```

### 5.7 PERF · 性能

```yaml
- id: PERF-001
  title: N+1 查询
  level: 主要
  rootCauseKey: PERF-001:n-plus-one
  signals:
    keywords: [ "for (", "forEach(", "stream().map(x -> mapper.selectById" ]
  description: 循环内单条查询/RPC，导致 N+1。
  fixHint: 改批量 IN 或一次 join；数据结构预聚合。

- id: PERF-002
  title: 循环内远程调用
  level: 主要
  rootCauseKey: PERF-002:remote-call-in-loop
  signals:
    keywords: [ "for (", "restTemplate", "feignClient", "kafkaTemplate.send" ]
  description: 循环内同步 RPC / HTTP / MQ，延迟叠加。
  fixHint: 批量接口；异步并行；合并发送。

- id: PERF-003
  title: 大对象 / 大集合无分页
  level: 主要
  rootCauseKey: PERF-003:large-payload
  signals:
    keywords: [ "selectList(", "findAll(", "readAllBytes" ]
  description: 一次加载全表 / 大文件 / 大列表到内存。
  fixHint: 强制分页/流式读取；游标或 Chunk 处理。

- id: PERF-004
  title: 索引缺失 / 慢查询
  level: 主要
  rootCauseKey: PERF-004:missing-index
  signals:
    keywords: [ "LIKE '%", "OR ", "SELECT *", "ORDER BY" ]
  description: 查询字段无索引、前导模糊、OR 拼接、多列排序未复合索引。
  fixHint: 依访问模式建组合索引；避免前导 %；OR 改 UNION。

- id: PERF-005
  title: SELECT *
  level: 次要
  rootCauseKey: PERF-005:select-star
  signals:
    regex:
      - SELECT\s+\*\s+FROM
    keywords: [ "select *" ]
  description: 使用 SELECT *，字段变更影响面大、带宽浪费。
  fixHint: 明列查询；实体懒加载大字段。

- id: PERF-006
  title: 无意义序列化 / 深拷贝
  level: 次要
  rootCauseKey: PERF-006:redundant-serialization
  signals:
    keywords: [ "JSON.parseObject(JSON.toJSONString(", "SerializationUtils.clone" ]
  description: 用 JSON 往返实现"深拷贝"或类型转换，CPU 浪费。
  fixHint: MapStruct/BeanUtils.copyProperties；必要时手工映射。
```

### 5.8 LOG · 日志

```yaml
- id: LOG-001
  title: 敏感信息落日志
  level: 严重
  rootCauseKey: LOG-001:sensitive-in-log
  signals:
    keywords: [ "log.info", "log.debug", "System.out" ]
    regex:
      - log\.(info|debug|warn|error)\s*\([^)]*(password|token|idCard|phone|bankCard)
  description: 日志打印密码、Token、身份证、手机号等敏感字段。
  fixHint: 统一脱敏工具；敏感字段过滤器；生产禁 DEBUG。

- id: LOG-002
  title: 异常堆栈吞掉
  level: 主要
  rootCauseKey: LOG-002:swallow-exception
  signals:
    regex:
      - catch\s*\([^)]+\)\s*\{\s*\}
    keywords: [ "e.printStackTrace()", "log.error(\"error\")" ]
  description: catch 空块、只打印 e.getMessage()、未记录堆栈或未上抛。
  fixHint: log.error("msg", e) 记录堆栈；不可恢复的必须上抛。

- id: LOG-003
  title: 日志级别误用
  level: 次要
  rootCauseKey: LOG-003:log-level-misuse
  signals:
    keywords: [ "log.info", "log.error" ]
  description: 高频循环用 info / error；异常路径用 info；调试信息用 warn。
  fixHint: 循环体尽量 debug；error 只用于真正异常；建立级别使用规范。

- id: LOG-004
  title: 日志缺少上下文（traceId / bizNo）
  level: 主要
  rootCauseKey: LOG-004:log-missing-context
  signals:
    keywords: [ "MDC", "traceId", "requestId" ]
  description: 日志缺 traceId/bizNo，无法链路排查。
  fixHint: 统一 MDC 注入 traceId / userId / tenantId；网关入口埋点。
```

### 5.9 CFG · 配置

```yaml
- id: CFG-001
  title: 环境变量 / 配置硬编码
  level: 主要
  rootCauseKey: CFG-001:config-hardcoded
  signals:
    regex:
      - "http[s]?://[a-zA-Z0-9.-]+(:\\d+)?"
      - "jdbc:[a-z]+://"
    keywords: [ "localhost:", "127.0.0.1:", "10.", "192.168." ]
    whitelist: [ "@Value(\"${", "${ENV_" ]
  description: URL、端口、目录、DB 连接串等硬编码到源码。
  fixHint: 迁移到 application-*.yml / 配置中心；按环境隔离。

- id: CFG-002
  title: 生产 / 测试环境配置混用
  level: 严重
  rootCauseKey: CFG-002:env-config-mixed
  signals:
    keywords: [ "application-dev.yml", "application-prod.yml", "if (env == \"prod\")" ]
  description: 代码里做环境判断走不同分支；测试数据源出现在生产 profile。
  fixHint: 使用 Spring profile 隔离；代码内禁止环境判断。

- id: CFG-003
  title: 敏感配置未加密
  level: 严重
  rootCauseKey: CFG-003:sensitive-config-unencrypted
  signals:
    keywords: [ "password:", "secret:", "accessKey:" ]
    whitelist: [ "ENC(", "配置中心加密" ]
  description: 生产真实凭证、外部可访问凭证或高权限凭证明文写入 application.yml / bootstrap.yml。
  fixHint: 生产凭证走配置中心、环境变量或加密下发。
  downgradeToInfo:
    - "Druid monitor、Actuator、Swagger、测试工具等内网监控 / 开发测试账号出现在 yml 中，且无证据表明为生产真实高权限凭证或外部可访问。"
    - "application-dev.yml、application-test.yml、local.yml、示例配置或模板配置中的默认账号密码，未证明会进入生产环境。"
    - "仅凭字段名 password、secret、accessKey 命中，但缺少 Java 配置绑定、profile、生效环境、外部暴露范围或真实凭证证据。"
- id: CFG-004
  title: 开关 / 调试代码残留
  level: 次要
  rootCauseKey: CFG-004:debug-code-left
  signals:
    keywords: [ "System.out.println", "printStackTrace", "// debug", "if (true)", "if (false)" ]
  description: printStackTrace / System.out / 强制 true/false 分支留存。
  fixHint: 上线前删除；用统一日志；开关走配置中心。
```

### 5.10 DEP · 依赖

```yaml
- id: DEP-001
  title: 已知 CVE 版本依赖
  level: 严重
  rootCauseKey: DEP-001:vulnerable-dependency
  signals:
    keywords: [ "log4j-core:2.14", "fastjson:1.2", "shiro:1.7", "spring-web:5.2" ]
  description: 依赖版本存在公开 CVE 且社区已发布修复版。
  fixHint: 升级到修复版；不可升级则临时缓解（关闭功能 / WAF 规则）。

- id: DEP-002
  title: 依赖未固定版本 / 雪花版本
  level: 主要
  rootCauseKey: DEP-002:unpinned-dependency
  signals:
    keywords: [ "LATEST", "RELEASE", "+" ]
  description: pom / gradle 中使用 LATEST / + / SNAPSHOT，构建不可复现。
  fixHint: 固定版本；BOM 统一管理；生产禁 SNAPSHOT。

- id: DEP-003
  title: 冗余依赖 / 版本冲突
  level: 次要
  rootCauseKey: DEP-003:redundant-conflicting-dependency
  signals:
    keywords: [ "mvn dependency:tree", "duplicate class" ]
  description: 引入未用依赖或存在版本冲突（同类库多版本共存）。
  fixHint: mvn dependency:analyze 清理；BOM 收敛版本。
```

### 5.11 API · 接口

```yaml
- id: API-001
  title: 参数未校验
  level: 主要
  rootCauseKey: API-001:param-validation-missing
  signals:
    keywords: [ "@RequestBody", "@RequestParam" ]
    whitelist: [ "@Valid", "@Validated", "@NotNull", "@NotBlank" ]
  description: 入参未使用 @Valid / @Validated；关键字段无非空 / 长度 / 范围约束。
  fixHint: DTO 加 JSR-303 注解；@Valid 强制生效；全局 BindingResult 处理。

- id: API-002
  title: 返回结构 / 错误码不规范
  level: 主要
  rootCauseKey: API-002:response-inconsistent
  signals:
    keywords: [ "return \"success\"", "throw new RuntimeException" ]
    whitelist: [ "R.ok", "R.fail", "ResultCode" ]
  description: 返回结构不统一；错误码零散未收敛；抛 RuntimeException 直接返回 500。
  fixHint: 全局统一 R<T>；错误码枚举集中管理；业务异常走 BusinessException。

- id: API-003
  title: 版本兼容破坏
  level: 严重
  rootCauseKey: API-003:api-breaking-change
  signals:
    keywords: [ "public interface", "@RequestMapping" ]
  description: 已发布接口删除字段、改类型、改必填、改错误码，破坏调用方。
  fixHint: 破坏性变更升 v2；旧版本保留过渡期；对外 SDK 变更走评审。

- id: API-004
  title: 幂等键 / 请求 ID 缺失
  level: 主要
  rootCauseKey: API-004:request-id-missing
  signals:
    keywords: [ "@PostMapping", "requestId", "traceId" ]
  description: 写接口未强制传 requestId/idempotencyKey；日志链路无关联标识。
  fixHint: 网关注入 traceId；写接口显式要求 idempotencyKey。
```

### 5.12 QUAL · 代码质量

```yaml
- id: QUAL-001
  title: 大段复制粘贴
  level: 主要
  rootCauseKey: QUAL-001:copy-paste-code
  signals:
    keywords: [ "// copy from", "// same as" ]
  description: 多处代码高度相似，导致修改遗漏、行为分叉。
  fixHint: 抽取公共方法 / 模板方法；用工具类或策略模式。

- id: QUAL-002
  title: 无用方法 / 依赖 / 死代码
  level: 次要
  rootCauseKey: QUAL-002:dead-code
  signals:
    keywords: [ "@Deprecated", "// unused", "if (false)" ]
  description: 未被调用的方法、类、字段、依赖长期留存。
  fixHint: 定期清理；IDE 提示 unused；@Deprecated 设定移除时间。

- id: QUAL-003
  title: 重复查询 / 重复远程调用
  level: 主要
  rootCauseKey: QUAL-003:duplicate-io
  signals:
    keywords: [ "selectById", "restTemplate" ]
  description: 一次业务处理内多次相同查询 / RPC，浪费性能。
  fixHint: 局部缓存或参数传递；批量合并。

- id: QUAL-004
  title: 上帝类 / 方法过长
  level: 主要
  rootCauseKey: QUAL-004:god-class-long-method
  signals:
    keywords: [ "public class", "public void" ]
  description: 单类 > 800 行、单方法 > 80 行、职责混杂。
  fixHint: 按职责拆分；提炼 Service / Handler；单一职责。

- id: QUAL-005
  title: 圈复杂度过高
  level: 主要
  rootCauseKey: QUAL-005:high-cyclomatic-complexity
  signals:
    keywords: [ "if", "else if", "switch", "case" ]
  description: 单方法圈复杂度 > 15，测试和维护成本高。
  fixHint: 提炼子方法；用策略/责任链替代深嵌套 if。

- id: QUAL-006
  title: 魔法值 / 硬编码常量
  level: 次要
  rootCauseKey: QUAL-006:magic-number
  signals:
    regex:
      - if\s*\(.*==\s*\d{2,}\s*\)
  description: 业务字面量散落各处，含义不清、修改易遗漏。
  fixHint: 提炼为常量或枚举；集中在 Constants / Enum。

- id: QUAL-007
  title: 命名 / 语义模糊
  level: 次要
  rootCauseKey: QUAL-007:naming-ambiguous
  signals:
    regex:
      - \b(temp|tmp|data|obj|info|flag|foo|bar|test)\d?\b
  description: 变量 / 方法 / 类使用 tmp/data/obj/flag 等无语义命名。
  fixHint: 命名体现业务含义；避免缩写；review 时统一约定。
```

---

## 六、AI 使用流程

AI 触发代码审查时，按以下步骤对照本表：

1. **对照识别**：扫描代码文件，逐段与本表 signals 对照，命中即候选。
2. **归一根因**：为每个候选计算 `rootCauseKey`，同键合并。
3. **计算位置指纹**：为每个 `rootCauseKey` 汇总 `locations[]`。
4. **描述归一去重**：对同键多描述做 `descriptionNorm` 归一，取最详尽一条作为主描述。
5. **级别裁定**：按本表 `level` 建议，可依上下文 ±1 级调整，多类别命中同位置按最高级别扣分（R4）。
6. **影响面上浮**：`locations` ≥ 3 处时按 R2 上浮一档。
7. **输出**：按 scoring-model.md 报告结构输出，每条问题携带 `id / rootCauseKey / level / locations / mergedFrom / fixHint`。
8. **幂等校验**：重复扫描时对比上次 `rootCauseKey` 集合，不一致必须给出差异说明（新代码 / 误报修正）。

## 七、扩展与维护

- **新增条目**：末尾追加，编号连续，不复用已废弃编号。
- **废弃条目**：保留编号并标 `deprecated: true`，说明原因与替代 id。
- **级别调整**：以本表 `level` 为建议基线；某项目/团队约定不同的，在项目根 `AI_RULES.md` 里做覆盖说明，不直接改本文件。
- **命中率反馈**：AI 生成报告时应记录本次命中的 id 集合，用于后续调整 signals 与级别。





