# Catalog 全量回归与 Wren 进程内集成

## 目标

以 ygg Catalog 为唯一事实源，在 KEX 进程内集成维护版 Wren Core/SDK，用于语义检索、规划辅助和候选 SQL；所有候选仍须经过 KEX 的 Catalog、AST、权限、契约、安全、只读执行和审计门禁。使用正式 50 条年报基线和固定 seed 的 20 条样本验证质量变化。

## 阶段

- [x] 阶段 1：确认根因与现有回归能力
- [ ] 阶段 2：新增回归测试并修复 SchemaContext/linking/编译边界
- [ ] 阶段 3：增强 SQL 校验可观测性
- [ ] 阶段 4：增强 Catalog 全量回归脚本和报告
- [ ] 阶段 5：运行定向测试、静态检查和真实 Catalog 回放
- [ ] 阶段 6：确认全部问题来源与去重口径
- [ ] 阶段 7：升级全链路回归覆盖、并发执行和失败分类
- [ ] 阶段 8：执行当前 Catalog 全量真实 Doris 回归
- [ ] 阶段 9：按根因分组修复并重复全量回归至通过
- [ ] 阶段 10：固化报告、命令和质量门禁
- [x] 阶段 11：完成 Wren 集成架构设计并取消独立 Docker 服务
- [x] 阶段 12：验证维护版 Wren Core/SDK 的 Python 3.12 进程内兼容性
- [x] 阶段 13：实现 Catalog→MDL 类型边界、编译器和 adapter
- [x] 阶段 14：在 `_build_drafts()` 接入影子语义检查并保留全部现有门禁
- [x] 阶段 15：运行定向测试、静态检查和固定 20 条回归
- [ ] 阶段 16：共享 Ollama 无其他任务时补跑修复后的最终 20 条报告
- [x] 阶段 17：复审正式 50 条智能统计回归样本，明确问题中的年份条件并同步修正示例 SQL
- [x] 阶段 18：将 50 条基线中的人员身份语义占位符替换为 Catalog 真实物理码值
- [x] 阶段 19：从 50 条候选 SQL 基线中移除由后置权限改写阶段注入的修订、区域和单位谓词
- [x] 阶段 20：将正式 v4 50 条结构化基线接入回归脚本，并分离候选 SQL 与最终权限 SQL 的评估口径
- [x] 阶段 21：使用正式 v4 基线固定 seed=42 运行 10 条真实 LLM+Doris 回归并整理失败批注稿
- [x] 阶段 22：下载 Qwen3.8 27B Q4_K_M，并用相同 10 条与 Qwen3 30B 对比
- [x] 阶段 23：快速对齐回归 replan 与 Planning 物理字段约束，将 Qwen3.8 同组通过率从 4/10 提升至 9/10
- [x] 阶段 24：固定 Qwen3.8 27B 为智能统计基座，执行正式 v4 全部 50 条并输出逐题规划/总耗时与失败批注稿
- [x] 阶段 25：按用户批注修正候选覆写、表头等列、OffStaffPerson 与编外人员契约口径，并复跑 50 条
- [x] 阶段 26：按独立审查收紧等列表头白名单并修复编外人员 metric-rule 残余路径，完成定向复跑

## 关键约束

- 保留应用层表和字段白名单，不以 Doris 账号权限代替业务安全边界。
- `allowed_tables` 表示当前 Catalog 已发布且允许执行的全部表。
- linking 命中表只影响相关性选择，不影响最终安全白名单。
- 不修改前端，不提交或推送 Git。
- SQL 比较以结构和业务语义为主，不要求文本完全一致。
- 每条问题必须覆盖 Planning LLM、SQL 生成、AST/Catalog/plan contract、权限改写、只读 Doris 执行与结果表头检查。
- 本轮输入范围收窄为单位年报与人员年报 sampleQuestions，即 `dwd_annual_organization` 和 `dwd_annual_person`。
- 规划输出与最终 SQL/结果列数量、顺序、alias 必须精确一致。
- 不部署 WrenAI Docker/HTTP 服务，不使用弃用的 GenBI Classic API。
- Wren 不持有 Doris 凭据、不执行 SQL，也不能产出或绕过 `ValidatedSql`。
- 若维护版 Wren 不提供 Python 3.12 可进程内调用的 Core/SDK，则停止 Wren 产品代码集成并报告兼容性结论，不伪造适配层。
- Wren Core 仅运行非阻断影子语义检查；不得采用其转换后的嵌套 SQL，主链路始终使用原始 KEX 候选 SQL。
- 本轮仅修订 50 条回归样本的问题与示例 SQL；问题和 SQL 的年份条件必须明确且完全一致，默认使用 2026 年表达原先模糊的单年条件。
- 人员身份条件直接使用当前 Catalog `termMappings` 物理值：事业人员 `InstPerson`、运动员 `AthPerson`、机关工人 `OrgPerson`、离休人员 `LeavePerson`、编外人员 `OffStaffPerson`；不再保留对应语义占位符。
- `revision_id`、`tenant_id`、`organization_id` 的当前修订和授权范围过滤不属于 Planning LLM 或 SQL 生成合同；正式基线候选 SQL 不包含这些谓词，由后置权限改写阶段统一覆写注入。

## 错误记录

| 错误 | 次数 | 处理 |
|---|---:|---|
| 查询命中模板表但该表不在 linking 构造的 allowed_tables | 1 | 拆分执行白名单和候选表 |
| Wren 官方 API 调研首次遇到上游临时不可用 | 1 | 使用官方仓库、包索引和维护文档交叉验证 |
| `wren-core` 包名不存在 | 1 | 确认官方发行包为 `wren-core-py==0.7.3`，导入名为 `wren_core` |
| Wren 20 条复测误用 DashScope，规划阶段全部 403/429 | 1 | 使用临时 ENV_FILE 保留现有连接配置，仅覆盖 LLM 为 71 Ollama 本地 30B 后重跑 |
| 最终 post-fix 20 条两次超过 10/15 分钟工具上限 | 2 | 发现其他会话持续运行 seed=42 的 20 条任务并争用同一 30B；不终止他人进程，保留为阶段 16 |
| 年份一致性校验脚本首次使用错误的 shell 换行转义 | 1 | 改为单行 Python 断言后通过，文档内容未受影响 |
| 身份码校验脚本中的 Markdown 反引号被 shell 当作命令替换 | 1 | 使用 `chr(96)` 构造代码围栏后重跑通过，文档内容未受影响 |
| 最终测试撞上并发写入中的 `permission_rewriter.py`，短暂缺少 `_expression_depth` 定义 | 1 | 确认该文件由其他任务继续写完后不做覆盖，原命令重跑 33 项通过 |
| Ollama 更新只替换主程序、runner 包未完成 | 1 | 使用完整 0.32.14 安装包补全 `/usr/local/lib/ollama` 并重启，30B 推理恢复 |
| Qwen3.8 OpenAI 兼容响应 content 为空 | 1 | 按 Ollama 官方文档使用 `reasoning_effort: none`，Planning JSON 恢复正常 |
| #15 候选 SQL 的括号别名 revision 谓词未清理 | 1 | 扩展候选清理匹配“括号 + 表别名”，定向测试通过；E2E 因 PG/Ygg 服务停止未补跑 |
| Ygg 中文 metric_code 不符合 KEX 标识规范 | 1 | remote normalizer 对非 ASCII code 生成稳定 `catalog_<sha256[:12]>` 标识，显示名保持中文 |
| #4 单元素列表条件误判 | 1 | 合同解析器使用拆包后的单值匹配 SQL，`['OffStaffPerson']` 正确识别 |
| 编外人员被识别为必需指标 | 1 | 人员身份字典标签从 semantic metric contract 排除，按过滤条件处理 |
| 等列表头规则范围过宽 | 1 | 仅允许用户明确批准的 #26/#27/#32/#46/#47/#50 使用等列数兜底 |
