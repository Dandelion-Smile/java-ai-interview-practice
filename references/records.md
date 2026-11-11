# 学习记录约定

此文件规定后续实际运行时的记录方式；设计阶段不创建任何正式学习批次。

## 文件结构

记录保存在 SKILL.md 指定的固定目录，不随聊天工作目录变化，不存进 skill 安装目录：

```text
java-ai/
  state.json                 题目、批次映射、反馈和排期的唯一权威记录
  sessions/
    B0001.md                 已交付批次全文，包含答案与复习题安排
  backups/
    state-r000001.json       写入前的可恢复快照
```

`state.json` 中的批次也保存完整 Markdown 正文，`sessions/` 是便于用户阅读的派生副本。副本缺失可从 state 恢复，不据副本倒推覆盖 state。没有新出题就不创建新 session。

## 状态字段

采用 UTF-8 JSON。顶层字段：

| 字段 | 含义 |
|---|---|
| schema_version | 初始为 1，格式变化须显式迁移 |
| revision | 每次成功写入递增 |
| timezone | Asia/Shanghai |
| source | 题源 ID、路径、SHA-256、页数 |
| next_question_number / next_batch_number | 下一可用全局编号 |
| questions | 以永久题目 ID 为键的字典 |
| batches | 以批次 ID 为键的字典 |
| feedback_events | 追加保存反馈及其更正 |
| question_revisions | 答案、分类、来源等变更记录 |
| configuration | 每批题量、新旧配额、范围配额、记录目录及其他已确认偏好 |

永久题目 ID 使用 `Q0001`、`Q0002`；批次使用 `B0001`、`B0002`。编号终身不重新分配。批次日期是独立字段，不把“第几天”用作主键。

每道题的字段：

| 字段 | 含义 |
|---|---|
| id / topic_key / title | 永久 ID、规范知识点键、当前题干 |
| primary_scope / related_scopes / subtopic | 主范围、关联范围、细分主题 |
| priority / priority_reason | P0 基础核心 / P1 基础补充，以及简短选题依据；默认不生成进阶题 |
| selection_basis | frequency_reason（常见面试考点依据）、frequency_evidence_type（qualitative 或实际统计来源）、foundational_reason（对应的基础能力）、difficulty（默认 foundation） |
| sources | 每项记录 source_id、chapter、original_heading、pdf_pages；页码从 1 开始 |
| external_references | 实际核实过的官方链接、适用版本和核实日期 |
| answer_version / answer | 当前答案版本、口述版、理解说明、追问与易错点 |
| first_issued_at / last_issued_at | 首次、最近出题时间 |
| occurrence_ids | 该题历次出现的唯一 ID |
| last_rating / rating_basis | null/会/模糊/不会，以及自评或作答评估 |
| success_stage | 0=尚无连续确认，1=下一次间隔 1 天，2=3 天，3 及以上=7 天 |
| next_review_date | 下次复习日期；初次未评估可暂定次日 |

`topic_key` 按“主范围.主题.核心考点”命名，例如 `foundation.database.transaction-acid`、`java.concurrent.atomicinteger-cas`。topic_key 用于辅助查重，仍须比较语义，不能认为键不同就是新题。

新建题目时：`last_rating=null`、`success_stage=0`。出现次数、展示次数不能代替有效反馈次数。

每个批次包含：ID、上海日期及时间、题量与新旧数量、完整交付正文，以及依次排列的 entries。每个 entry 包含：

```json
{
  "occurrence_id": "B0001-01",
  "position": 1,
  "question_id": "Q0001",
  "kind": "new",
  "review_timing": null,
  "answer_version": 1,
  "assessment_status": "pending",
  "effective_feedback_id": null
}
```

`kind` 为 new/review；复习时 `review_timing` 为 due/early/unassessed。每个批次不能重复 question_id 或等价知识点。题目 last_rating 与某次出现是否已评估是不同概念：旧题有“会”的评级，也可能本次复习尚未作答。

每条反馈事件保存 event_id、occurrence_id、question_id、原始用户内容、反馈时间、rating、basis、关键缺漏、supersedes（被更正事件或 null）、修改前后掌握阶段及复习日期。

反馈更正应从该题的有效反馈序列重算掌握阶段与排期；同一 occurrence 的修订不能重复增加成功次数。后续已有反馈时，不以修订后的旧反馈直接覆盖最新有效状态。

## 持久化流程

1. 先准备题目、批次正文或反馈变更，不在生成答案或等待用户时长期占用写锁。
2. 写入前用独占创建（例如 Python `open(path, 'x')`）取得记录目录 `.write.lock`，写入操作标识与时间。锁已存在时读取当前状态并短暂重试；仍被占用则停止本次写入并报告。不能擅自删除来源不明的锁。
3. 持锁后重新读取最新 state，核对 revision。若发生变化，释放锁，基于最新记录重做去重、编号和排期；不要用旧快照覆盖。
4. 保存旧 state 到带 revision 的 backups 文件。将新 state 写入同目录临时文件，解析并核验引用、唯一性、计数、页码范围及排期字段，再用原子替换写入 state.json。
5. 从已提交 state 导出本批 sessions Markdown，重新读取两者并确认一致，最后释放本操作持有的锁。state 首次创建也须遵循独占锁流程。
6. state 已提交但 Markdown 导出或展示中断时，下次恢复同一批正文，不新增编号或重复入账。导出失败时明确记录已入账但可读副本尚未生成，修复后交付。

若文件损坏，先保留故障文件、检查备份和完整性；仅在确认没有丢失已交付记录时恢复。无法确认时报告可恢复范围，不能假装恢复完整，也不能静默归零。

## 输出前检查

- 每道新题同时通过“面试高频”与“基础”筛选，留有两项具体依据；定性筛选不能伪装成统计数据。
- 答案与追问也保持基础难度，不依靠源码细节、冷门参数或复杂推导完成回答。
- 新题均有已核对的 PDF 来源，页码在 1～166；不把外部补充说成 PDF 原文。
- 默认 10 道主问题，追问不计数；实际配额变化有说明。
- 新题无历史语义重复，复习题沿用原 ID，批内序号映射可唯一解析。
- 未反馈项目保持 pending，未学习的题没有被计作掌握。
- 已交付全文可以从 state 重建，同一天普通重试返回已有批次。
