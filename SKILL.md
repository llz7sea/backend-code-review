---
name: backend-code-review-score
description: Use when a user explicitly requests backend code review, Code Review scoring, periodic backend quality statistics, contributor or commit analysis, or a backend HTML review scoring report; do not use for ordinary backend coding, self-review, debugging, frontend review, or mobile review.
---

# 后端代码审查评分

## 概述

这是一个面向 Java、BladeX/Saber 及通用后端项目的证据驱动代码审查 Skill：在明确触发后，按稳定问题码表识别和去重问题，执行 100 分制质量评分、人员/团队聚合和风险门禁，并使用内置唯一 HTML 模板生成可复核的审查评分报告。

## 触发边界

仅在以下情况使用：

- 用户明确要求后端代码审查、Code Review、审查评分、打分或周期性审查。
- 用户要求审查当前修改、本人提交、指定时间段、指定分支、指定人员、目录/模块或当前项目全部提交。
- 用户要求按后端人员、提交、分支、目录或模块统计质量问题并生成报告。
- 用户明确调用 `$backend-code-review-score` 或要求生成后端 HTML 审查评分报告。

不要在以下情况主动使用：

- 普通后端编码、修复、重构、调试或开发阶段自审，且用户没有要求评分报告。
- 只要求解释代码、排查问题或给出优化建议。
- 前端、原生移动端、H5、Hybrid、uni-app、Taro 或其他非后端审查。

## 审查范围确认

支持四种审查类型：

1. **当前修改代码审查**：覆盖未暂存、已暂存和未跟踪文件，并明确实际纳入的范围。
2. **本人提交代码审查**：按作者、提交、分支和基线确定完整提交范围。
3. **全局时间段代码审查**：按起止时间、分支、作者、目录和提交记录完整汇总。
4. **当前项目全部提交审查**：按当前项目可追溯提交和分支范围完整审查。

用户没有提供必要的类型、基线、作者、时间、分支、目录或模块条件，且无法从当前上下文唯一推断时，只询问缺失的关键范围；不得额外询问“是否进入审查”。禁止抽样审查后推断整体结论。

## 主流程

1. 读取目标仓库就近的 `AGENTS.md`、后端项目入口规则、框架/编码/项目规则和本 Skill 相关 references。
2. 固化审查范围与 Git 证据：`git status --short --branch`、工作区/暂存区/基线 diff、提交日志、作者信息和必要的历史代码基线；保护用户已有改动。
3. 先读取 [references/issue-catalog.md](references/issue-catalog.md)，按 `id`、`signals`、`rootCauseKey`、位置指纹和描述归一规则识别、合并和排除重复问题。
4. 读取 [references/scoring-model.md](references/scoring-model.md)，从 100 分开始按问题级别和影响范围扣分；执行单人分、多人贡献加权、低分尾部、风险封顶和唯一最终风险分状态判定。
5. 读取 [references/ai-coding-rate.md](references/ai-coding-rate.md)，统计新增/删除行、有效文件、代码总量占比和整体/人员 AI 编码率。AI 编码率不是质量分，不影响门禁。
6. 读取 [references/report-generation.md](references/report-generation.md)，将脱敏证据、问题、分数、人员统计和隐藏审查 JSON 写入唯一模板 [templates/CODE_REVIEW_REPORT_TEMPLATE.html](templates/CODE_REVIEW_REPORT_TEMPLATE.html)。
7. 生成到审查项目根目录 `review-output/backend-code-review-report-YYYYMMDD-HHmmss.html`；目标文件已存在时追加随机码或短 UUID，只新增不覆盖。
8. 生成后执行静态门禁：占位符、六章标题顺序、动态内容转义、表格结构、代码证据、状态/分数一致性，以及整个 `reviewDataJson` 的逐层字段集合、字段类型、允许值、内容来源和派生一致性；任一层缺字段、多字段、类型错误、占位内容或静默丢字段时禁止输出报告。

## 证据和报告要求

- 每个扣分问题必须有人员/提交、文件和行号、根因码、影响、扣分、修复建议和可复核证据；证据不足只能标记“待确认”，不得扣分。
- 发现的问题代码片段必须脱敏并写入问题详情；不得因代码展示策略而隐藏已确认问题证据。
- 报告必须包含六章：审查概览、提交日志、问题清单、问题详情、问题跟踪、结论与后续动作。
### 隐藏 JSON 严格合同

- 严格合同作用于整个 `<script id="reviewDataJson" type="application/json" hidden>` JSON，不只作用于 `persons[].issues`。顶层、`summary`、`persons[]`、顶层 `issues[]`、`persons[].issues[]` 必须逐层使用 [references/report-generation.md](references/report-generation.md) 定义的唯一字段集合。
- 每一层对象的字段集合必须精确匹配合同：不得缺少字段，不得增加合同外字段，不得静默忽略未知字段，也不得因 DTO、序列化器或 JSON 解析器会忽略未知字段而放行。字段类型、枚举、数组元素和空值规则必须逐项校验；校验失败时禁止生成、保存或下载报告。
- 所有合同字段都必须显式生成。必填业务内容必须来自实际 Git 证据、问题事实源、评分算法或 AI 编码率计算，不得使用 `null`、空字符串、占位文本、默认零、伪造值或重复字段绕过完整性校验；只有详细合同明确允许为空的字段才能使用规定的空值。
- `summary.codeType` 固定为 `backend`。`persons[].issues[]` 每项必须且只能包含 `id`、`level`、`deduction`、`person`、`commit`、`location`、`mustFix`、`summary` 八个字段；无问题人员必须显式写 `issues: []`，且 `issueCount === issues.length`。
- 顶层 `issues[]` 是问题和评分的唯一事实源。首次生成、渲染、编辑、删除、撤销、改派及保存前，必须从当前顶层问题完整重建人员问题快照、问题数量、扣分、人员分、团队聚合分、风险封顶和最终风险分；删除的问题及其扣分不得残留。
- 可见 HTML、顶层问题、人员问题快照、人员统计和 `summary` 的问题数、分数、结论必须一致。运行时派生字段或计算临时字段不得序列化进最终隐藏 JSON。
- 如确需新增 `nodeId` 或其他业务字段，必须先修改详细合同，并同步所有生成器、模板校验器、DTO、持久化结构和读取方；合同未变更前，该字段属于多余字段，不得临时生成或依赖解析器静默接收。
- 不加载远程资源。动态文本、代码、commit、作者、路径和隐藏 JSON 必须完成 HTML、JSON 和脚本上下文安全转义。

## 输出边界

报告正文、分数、问题清单、代码片段、门禁结论和验证明细全部只写入 HTML 文件。报告成功生成后，控制台或对话只输出：

```text
后端代码审查报告已生成：<报告绝对路径>
```

报告生成失败或审查范围缺失时，如实说明阻塞原因；不得伪造报告路径，也不得在对话中展开审查正文。


