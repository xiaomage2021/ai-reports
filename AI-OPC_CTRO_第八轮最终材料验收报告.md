# AI-OPC CTRO 第八轮最终材料验收报告

> **审查对象：** Telegram Monitor V2.2 第八轮最终整改材料  
> **审查身份：** AI-OPC CTRO（独立技术审查官）  
> **最终决策人：** CEO  
> **审查日期：** 2026-11-03  
> **最终结论：** CONDITIONAL（有条件不通过）  
> **材料来源：** Dropbox 文件《Telegram Monitor 提交 CTRO 第八轮审核.md》

---

## 文档说明

本文件整理的是第八轮 CTRO 审查报告。当前可恢复的历史记录包含报告结论、逐项技术发现、验收意见及第九轮整改要求，但未能重新取得此前助手回复的逐字原文。因此，本文件是**依据已保存的审查记录重建的完整审查整理版**，不是逐字导出的原始聊天记录。

本文件不包含 Dropbox 中第八轮原始材料全文；审查对象仍是该原始材料。

---

# 一、最终审查结论

<badge>CONDITIONAL</badge>

**第八轮材料有明显进展，但尚未达到最终验收标准。**

第八轮已针对第七轮发现的多项关键缺陷补充了事务伪代码、状态约束及恢复逻辑，尤其是：

1. 入站任务使用 128-bit 随机 `task_id`。
2. 入站 Worker 使用 `claim_token`，并在续租及最终提交时核验任务身份和状态。
3. Reaper 回收任务时清除旧 Worker 的 Claim 信息。
4. Worker 最终提交时将 cooldown、outbound 插入及 inbound 完成放入同一事务。
5. MTProto 对账增加三次扫描，并要求最终扫描完整后才允许将任务标记为 `SENT`。
6. `button_actions`、`outbound_queue` 等表增加了部分租约和执行状态字段。
7. 入站事务在数据库 COMMIT 后再投递内存队列，降低了提交前唤醒造成的丢失风险。

但是，第八轮仍有一个关键 P0 问题没有真正闭环：

**出站任务 CLAIMED 超时恢复没有区分“尚未调用 Telegram API”与“Telegram 已成功接收但本地没有收到回执”两种情况。** 当前 Reaper 对所有过期 CLAIMED 任务直接转入 `RETRY_WAIT`，在第二种情况下可能导致重复发送。

此外，按钮执行恢复、跨进程写入协调、备份恢复演练、数据清理实际实现及 MTProto 实际验证仍不充分。

因此：

- 不批准 `PASSED`。
- 不批准第八轮材料作为完整最终架构定稿。
- 不批准 Telegram Monitor V2.2 正式生产准入。
- 不批准出站发送模块进入正式实现。
- 不要求推倒重建现有架构。
- 可以由 CEO 单独批准在隔离环境中开展局部原型开发，但必须使用模拟 Telegram API、虚拟数据库和测试数据。
- 第九轮应继续围绕未闭环事项定点整改，不得将设计说明或测试计划表述为测试通过。

---

# 二、审查材料与证据边界

根据已读取的 Dropbox 第八轮文件，材料包含：

- 七张 SQLite 数据表 DDL；
- 六段事务伪代码；
- 状态流转图；
- P0/P1 整改对照表；
- 12 组隔离测试方案；
- 第八轮验收提示词。

文件信息：

| 项目 | 内容 |
|---|---|
| 文件名 | `Telegram Monitor 提交 CTRO 第八轮审核.md` |
| Dropbox 路径 | `/网盘中继/Telegram Monitor 提交 CTRO 第八轮审核.md` |
| 文件大小 | 45,353 bytes |
| 提取正文长度 | 约 36,457 字符 |
| 最后修改时间 | 2026-11-03 21:43（北京时间） |
| 审查方式 | 对提交的 DDL、事务伪代码、状态图及测试方案进行静态审查 |

**证据边界：**

- 本轮审查属于材料级静态技术审查。
- 12 组隔离测试全部标记为 `NOT_RUN`。
- 未据此认定任何测试已经通过。
- 未执行真实并发测试、真实 Telegram API 调用、生产数据库操作或恢复演练。
- 设计伪代码通过静态审查，不等于实现代码已经通过验证。

---

# 三、第八轮相对第七轮的整改进展

## 3.1 入站事务一：消息接收与入库

第八轮材料采用：

- `task_id = secrets.token_hex(16)`，即 128-bit 随机标识；
- 在 `BEGIN IMMEDIATE` 事务内检查 `runtime_state` 的 `fencing_token`；
- `fencing_token` 不存在或为空时拒绝入库，不再默认使用 `0`；
- 使用 `INSERT OR IGNORE` 写入 `message_dedup`；
- 如果检测到重复消息则回滚；
- 创建 `inbound_queue` 任务；
- 数据库 `COMMIT` 成功后才投递内存队列；
- 如果内存队列已满，只记录警告，不把已经成功入库的任务错误报告为失败，由 Reaper 扫描兜底。

**审查意见：通过设计层面的定点复核。**

第七轮曾存在 COMMIT 后 QueueFull 异常处理仍尝试 ROLLBACK，可能把已入库任务报告为失败的问题。第八轮调整为先 COMMIT、后投递队列，且队列满时不再将数据库事务误报为失败，方向正确。

仍需在隔离测试中验证：队列满、进程崩溃、Reaper 扫描及重复唤醒等边界场景。

## 3.2 入站事务二：Claim、续租与 Reaper 回收

第八轮材料补充：

- Claim 时生成 `claim_token = token_hex(16)`；
- Renew 使用 `task_id`、`worker_id`、`claim_token` 和 `status='PROCESSING'` 作为更新条件；
- Reaper 将超时任务重新置为 `PENDING` 时清除 `worker_id`、`claim_token`、`claimed_at`；
- Reaper 将超过重试上限的任务置为 `FAILED` 时，同样清除旧 Claim 信息；
- Reaper 在事务 COMMIT 后才投递内存队列。

**审查意见：通过设计层面的定点复核。**

第七轮中 FAILED 分支残留旧 Claim 信息、以及 Reaper 在 COMMIT 前投递内存队列的问题，已得到针对性修复。

但实际实现仍需通过隔离并发测试确认，不得仅凭伪代码认定运行时行为已经通过。

## 3.3 入站事务三：Worker 决策、Fencing 与最终提交

第八轮材料补充：

- 使用入站任务保存的 `receive_token` 与当前 `fencing_token` 比较；
- stale 分支的 UPDATE 绑定 `task_id`、`worker_id`、`claim_token` 和 `status='PROCESSING'`；
- Claim 查询增加 `status='PROCESSING'` 条件；
- cooldown 写入、outbound 任务插入、inbound 任务完成放在同一事务中；
- inbound 的 `COMPLETED` UPDATE 绑定 `task_id`、`worker_id`、`claim_token`、`status='PROCESSING'`；
- 对最终 UPDATE 检查 `rowcount`；
- payload 深拷贝后再添加指纹字段。

**审查意见：通过设计层面的定点复核。**

上述修改针对第七轮发现的旧 Worker 越权提交、stale 更新覆盖新状态，以及 inbound 完成更新缺少 Claim 身份约束等问题，方向正确。

注意：只有实际 SQL 条件、事务边界和 rowcount 检查在代码中一致落实，并通过并发隔离测试后，才能将此项提升为实现级通过。

## 3.4 MTProto 对账：三次扫描与完整扫描约束

第八轮材料补充：

- 对账扫描间隔为 `[5, 15, 30]`；
- 每次扫描重新检查目标时间窗口；
- 只有唯一匹配且 `final_scan_complete` 为真，才允许将任务标记为 `SENT`；
- 对账状态 UPDATE 增加 `WHERE status='UNKNOWN_ERROR'` 条件。

**审查意见：设计方向明显改善，但尚未完成实现验证。**

第七轮中“只要发现唯一匹配就标记 SENT，即使窗口未扫描完整”的风险已得到针对性修正。

但仍有以下限制：

1. 每次扫描最多读取 100 条消息，材料没有证明所有场景下都能完整覆盖目标窗口。
2. 分页逻辑、Telethon `offset_date` 方向及边界语义尚未通过真实或隔离环境验证。
3. 零宽字符指纹是否能在 Telegram 实际消息内容中稳定保留，尚未验证。
4. 对账 UPDATE 没有检查 `rowcount`，可能出现日志显示已完成、实际状态却未更新的情况。
5. 三次扫描目前是伪代码设计，不是已执行的测试结果。

因此，此项不能标记为 `PASSED`。

---

# 四、P0 级阻断问题

## P0-05：出站任务 CLAIMED 超时恢复可能造成重复发送

<badge color="danger">BLOCKED</badge>

### 4.1 第八轮已经补充的内容

第八轮事务四提交了：

- `claim_outbound_task()`；
- 原子领取 `PENDING` 或到期 `RETRY_WAIT` 任务；
- `reap_timeout_outbound_tasks()`；
- `retry_count`、`max_retry_count`；
- `claim_expires_at`；
- 超时任务进入 `RETRY_WAIT` 或 `DEAD` 的处理。

这些内容改善了出站任务领取与超时回收的结构。

### 4.2 核心缺陷

当前 Reaper 对过期 `CLAIMED` 任务采用类似以下逻辑：

```sql
UPDATE outbound_queue
SET status='RETRY_WAIT',
    worker_id=NULL,
    claim_token=NULL,
    claimed_at=NULL,
    claim_expires_at=NULL,
    retry_count=retry_count+1
WHERE status='CLAIMED'
  AND claim_expires_at < ?
  AND retry_count < max_retry_count;
```

该逻辑没有区分以下两种完全不同的崩溃位置：

**场景 A：领取任务后、调用 Telegram API 之前崩溃**

- Worker 已领取任务；
- 尚未向 Telegram 发出请求；
- Worker 崩溃；
- 本地可以安全地将任务重新放入重试队列。

**场景 B：Telegram 已接收消息，但本地没有收到或保存成功回执时崩溃**

- Worker 已领取任务；
- 已经调用 Telegram API；
- Telegram 服务端可能已经成功接收并发送消息；
- Worker 在收到响应或写入本地 SENT 状态前崩溃；
- Reaper 将任务直接放入 `RETRY_WAIT`；
- 新 Worker 再次发送，可能造成客户收到重复消息。

场景 B 是本系统必须重点防范的“外部副作用结果不确定”问题。

### 4.3 第八轮材料缺失的关键环节

第八轮事务四没有提交完整的：

- 出站 Worker 续租函数；
- Telegram API 实际发送伪代码；
- 发送开始状态，例如 `SENDING`；
- `send_started_at` 等发送开始时间字段；
- API 明确成功后的条件提交；
- API 明确失败后的重试处理；
- API 超时或回执丢失时进入 `UNKNOWN_ERROR` 的代码；
- `UNKNOWN_ERROR` 经 MTProto 对账后再决定 `SENT`、`MANUAL_REVIEW` 或其他安全状态的完整闭环。

材料中的文字说明提到“已发起请求遇弱网则交由 UNKNOWN_ERROR 确界核验”，但实际事务伪代码没有完整实现这条路径。

### 4.4 必须整改的状态机

第九轮必须明确区分：

| 状态/结果 | 含义 | 允许的后续处理 |
|---|---|---|
| `PENDING` | 等待领取 | Worker 可领取 |
| `CLAIMED` | 已领取，尚未开始外部发送 | 租约过期后可安全回收 |
| `SENDING` | 已进入外部 API 调用阶段 | 不能仅因租约过期就盲目重发 |
| `SENT` | 已取得明确成功证据 | 不得再次发送 |
| `RETRY_WAIT` | 已确认本次未产生外部副作用，可重试 | 到期后重新领取 |
| `UNKNOWN_ERROR` | 请求结果不确定 | 必须先对账，不得直接重发 |
| `MANUAL_REVIEW` | 自动对账无法安全判定 | 人工核验 |
| `DEAD` | 达到重试上限或明确终止 | 不再自动发送 |

必须形成以下闭环：

1. 领取任务时进入 `CLAIMED`。
2. 即将调用 Telegram API 前，先在数据库中以条件 UPDATE 将状态改为 `SENDING`，并写入发送开始时间。
3. API 明确成功：使用 `task_id`、`worker_id`、`claim_token`、当前状态等条件更新为 `SENT`，检查 `rowcount`。
4. API 明确失败且能够确认没有产生外部副作用：进入 `RETRY_WAIT`。
5. API 超时、连接中断、响应丢失或进程在发送后崩溃：进入 `UNKNOWN_ERROR`。
6. `UNKNOWN_ERROR` 必须先执行 MTProto 对账。
7. 唯一匹配且完整扫描：可以判定 `SENT`。
8. 无法完整扫描、匹配不唯一或证据不足：进入 `MANUAL_REVIEW`，不得盲目重发。
9. 只有能够证明外部发送未发生时，才允许重新进入 `RETRY_WAIT`。
10. 对 CLAIMED 与 SENDING 的租约回收必须使用不同逻辑。

**本项是第八轮未通过的核心 P0 阻断项。**

---

# 五、P1 级未闭环问题

## P1-01：按钮执行恢复缺少完整租约与 Reaper 实现

第八轮事务六已经包含：

- 按钮执行 Claim；
- `execution_token`；
- 外部 API 调用；
- 完成或失败后的状态写入；
- DDL 增加 `executing_at`、`execution_token`。

但仍存在以下缺口：

1. DDL 没有 `execution_expires_at`。
2. 状态图声称 `EXECUTING` 崩溃后可以回收，但没有提交对应 Reaper 伪代码。
3. 外部 API 结果未知时，材料倾向于直接视为 `FAILED`。
4. 如果 API 已经执行成功，但本地没有收到回执，直接允许再次执行可能重复执行 BAN、MUTE 等操作。
5. 最终 UPDATE 的条件和 `rowcount` 检查仍需明确。
6. 没有充分说明如何对未知结果进行幂等核验。

**整改要求：**

- 增加执行租约到期字段；
- 补充按钮执行 Reaper；
- 区分明确失败与结果未知；
- 对 BAN/MUTE 等操作增加执行前后状态核验或可证明的幂等处理；
- 最终状态更新必须绑定按钮任务 ID、执行者、execution_token 和预期状态；
- 检查 UPDATE `rowcount`；
- 对无法确认的外部副作用进入人工核验，不得直接盲目重试。

## P1-02：AsyncWriteQueue 与 flock 的运行机制仍未提交完整证据

第八轮解释了：

- `flock` 用于进程间互斥；
- `asyncio.Lock` 用于同一进程内的异步互斥。

这个概念区分是正确的，但材料没有提交完整的运行机制代码。

缺失内容包括：

- flock 获取锁的代码；
- 锁释放代码；
- 异常退出时的释放与清理；
- 进程启动时的互斥处理；
- 所有数据库写入口统一遵守同一写锁的证明；
- 锁竞争、进程异常退出和锁文件处理的隔离测试。

**审查意见：部分通过，不能认定跨进程写入协调已经完成。**

整改时应保持 KISS，不要为了补足锁机制而引入 Redis、RabbitMQ 或其他不必要的基础设施。重点是明确所有写事务的统一入口和锁边界。

## P1-03：备份与恢复只有描述，缺少完整恢复演练

第八轮材料补充了：

- 带时间戳的备份文件命名；
- 备份文件大小大于 0 的检查；
- `PRAGMA integrity_check`。

但这还不能证明系统能够恢复。

`integrity_check=ok` 只能说明 SQLite 数据库结构完整性检查通过，不能单独证明：

- 备份文件内容完整；
- 恢复流程正确；
- WAL 数据已正确处理；
- 恢复后的业务数据一致；
- 索引、约束和关键状态没有丢失；
- 恢复后系统能够继续正常处理任务。

第八轮 TC-11 主要是备份与 integrity check，并不构成完整恢复演练。

**整改要求：**

1. 提交完整备份函数；
2. 明确 WAL checkpoint 或一致性备份方式；
3. 检查备份失败、磁盘空间不足、文件损坏等异常；
4. 实现备份保留策略；
5. 在独立临时目录恢复备份；
6. 对恢复后的数据库执行 `integrity_check`；
7. 验证关键业务表、任务数量、关键状态及必要索引；
8. 提交恢复前后数据一致性结果；
9. 不得在生产数据库上进行未经授权的恢复或覆盖操作。

## P1-04：数据清理策略方向合理，但缺少实际清理实现

第八轮的数据保留规则较第七轮合理：

- 可以考虑清理 `COMPLETED`；
- 可以考虑清理 `DROPPED_COOLDOWN`；
- 保留 `FAILED`；
- 保留 `MANUAL_REVIEW`。

但仍缺少：

- 实际清理 SQL；
- 定时清理任务；
- 清理批次与事务边界；
- 清理日志；
- 清理失败后的恢复方式；
- 清理前后的数量核验；
- 对关联表和外键关系的保护。

**审查意见：策略方向可接受，尚未达到实现验收。**

不得在没有明确保留期限、关联关系和恢复策略的情况下执行批量删除。

### admin_audit_logs 的“永久保存、防篡改”表述不准确

SQLite 普通表本身不具备防篡改能力。拥有数据库文件写权限的主体可以修改或删除记录。

因此：

- “永久保存”需要明确保留期限、容量和归档机制；
- “防篡改”需要额外机制，例如外部只追加日志、权限隔离、哈希链或独立存储；
- 仅仅把记录写入 SQLite 表，不能声称已经实现防篡改。

第九轮应修正文档中的安全承诺，不得将普通数据库表描述为具备其本身不具备的安全属性。

## P1-05：MTProto 对账仍需补足结果校验和实际验证

第八轮已补充三次扫描及 `final_scan_complete` 条件，属于重要进展。

但仍需解决：

- 对账 UPDATE 必须检查 `rowcount`；
- 日志必须区分“扫描完成”“状态更新成功”和“状态更新未命中”；
- 每次最多 100 条的分页逻辑必须能够覆盖完整时间窗口；
- 必须验证 Telethon `offset_date` 的方向及边界语义；
- 必须验证零宽字符指纹在实际消息读取中的保留情况；
- 对账失败、权限不足、消息不可见、扫描不完整时必须进入安全状态；
- 不得因为只找到一条匹配就忽略扫描完整性要求。

**审查意见：设计逻辑有改善，但全部测试仍为 NOT_RUN，不能判定通过。**

## P1-06：出站 Claim UPDATE 应补齐到期条件

第八轮出站领取逻辑的 SELECT 会检查：

- `PENDING`；
- 或 `RETRY_WAIT AND next_retry_at <= now`。

但后续 UPDATE 只检查 `PENDING` 或 `RETRY_WAIT`，没有将 `next_retry_at <= now` 一并放入 UPDATE 条件。

由于当前设计使用 `BEGIN IMMEDIATE`，写事务在事务期间会串行，因此不能夸大为已经证明存在必然并发竞态。

但为提高条件更新的可审计性，建议将 SELECT 的资格条件完整带入 UPDATE，形成清晰的条件更新（CAS）逻辑，并检查 `rowcount`。

## P1-07：入站 Reaper 的条件更新可进一步加强

第八轮 Reaper 使用事务锁保护 SELECT 与 UPDATE，降低了事务期间被其他写事务改变状态的可能性。

但 UPDATE 仍主要按 `task_id IN (...)` 进行，没有完整重新绑定：

- `worker_id`；
- `claim_token`；
- `claimed_at`；
- 预期状态。

在 `BEGIN IMMEDIATE` 保护下，不能据此断言存在确定的并发漏洞。

但为了让代码本身更容易审计、降低未来重构时的风险，建议在 UPDATE 中补齐 Claim 身份与预期状态条件，并检查 `rowcount`。

---

# 六、七张 DDL 与六段事务伪代码总体评价

## 6.1 DDL 总体评价

| 数据表/模块 | 第八轮审查意见 | 主要判断 |
|---|---|---|
| `message_dedup` | CONDITIONAL | 128-bit task_id 方向合理；仍需核验去重保留策略及清理关联 |
| `inbound_queue` | CONDITIONAL | Claim Token、receive_token、重试字段及 Reaper 清理有所改善；需隔离并发验证 |
| `user_cooldown` | CONDITIONAL | 与 outbound 插入、inbound 完成放在同一事务是正确方向；UPSERT 与 rowcount 仍需验证 |
| `outbound_queue` | BLOCKED | CLAIMED/SENDING 未充分区分，发送结果不确定时可能重复发送 |
| `button_actions` | CONDITIONAL | 有 execution_token、executing_at，但缺少完整租约到期与崩溃回收闭环 |
| `runtime_state` | CONDITIONAL | fencing_token 缺失时拒绝入库的处理更明确；仍需运行时验证 |
| `admin_audit_logs` | CONDITIONAL | 普通 SQLite 表不能真正防篡改；保留与外部审计机制未闭环 |

## 6.2 六段事务伪代码评价

| 事务 | 审查结论 | 说明 |
|---|---|---|
| 事务一：入站消息接收与入库 | 设计层面通过 | COMMIT 后投递队列，QueueFull 不再误报事务失败 |
| 事务二：入站 Claim、续租与 Reaper | 设计层面通过 | 过期回收时清除旧 Claim 信息，提交后再唤醒 |
| 事务三：Worker 决策与最终提交 | 设计层面通过 | Fencing、Claim 身份、状态条件及同事务提交有所补强 |
| 事务四：出站 Claim 与超时恢复 | BLOCKED | 没有完整实现 CLAIMED/SENDING/UNKNOWN_ERROR 发送状态机 |
| 事务五：MTProto 对账 | CONDITIONAL | 三次扫描与完整扫描条件已补充，但分页、rowcount、指纹仍待验证 |
| 事务六：按钮执行 | CONDITIONAL | 有执行 Claim 与 Token，但租约、Reaper、未知结果幂等恢复不完整 |

---

# 七、隔离测试与证据等级

第八轮材料列出了 12 组隔离测试方案，但全部标记为：

`NOT_RUN`

这是正确、诚实的证据标记。

**NOT_RUN 不等于 PASSED，也不等于 FAILED；它表示尚无测试执行结果。**

本轮不得将以下事项描述为已经验证通过：

- 双 Worker 竞争；
- 旧 Worker 失去 Claim 后无法提交；
- Fencing Token 切换；
- Reaper 并发回收；
- 出站 API 回执丢失；
- UNKNOWN_ERROR 对账；
- 按钮操作崩溃恢复；
- SQLite 写入锁跨进程互斥；
- WAL 备份与恢复；
- 数据保留与清理；
- MTProto 分页和完整扫描；
- 零宽字符指纹稳定性。

建议后续证据按以下层次记录：

| 证据层级 | 含义 | 本轮可否据此认定 |
|---|---|---|
| 设计说明 | 文档描述了预期行为 | 可以确认设计意图，不能确认实现 |
| 静态材料审查 | DDL、SQL、伪代码逻辑审查 | 可以确认材料中存在相应逻辑 |
| 隔离单元测试 | 使用虚拟数据库和模拟 API 执行 | 需提供真实测试结果 |
| 隔离集成/并发测试 | 多 Worker、异常注入、恢复测试 | 需提供日志与可复现证据 |
| 真实环境验证 | 真实 Telegram、真实凭证或生产环境 | 必须另行授权，不属于本轮 |

---

# 八、第八轮验收结果汇总

| 审查项 | 结论 | 是否阻断正式准入 |
|---|---|---|
| 入站消息事务与去重 | 设计层面通过，待测试 | 否，需验证 |
| 入站 Claim 与动态续租 | 设计层面通过，待测试 | 否，需验证 |
| Reaper 清除旧 Claim | 设计层面通过，待测试 | 否，需验证 |
| Worker Fencing 与最终提交 | 设计层面通过，待测试 | 否，需验证 |
| cooldown/outbound/inbound 原子事务 | 设计层面通过，待测试 | 否，需验证 |
| MTProto 三次扫描 | CONDITIONAL | 是，未完成验证 |
| 出站 CLAIMED 超时恢复 | BLOCKED | **是，P0** |
| SENDING 与 UNKNOWN_ERROR 状态机 | 未闭环 | **是，P0** |
| 按钮执行租约与崩溃恢复 | CONDITIONAL | 是，P1 |
| AsyncWriteQueue/flock 实际机制 | CONDITIONAL | 是，需补证据 |
| 备份与独立恢复演练 | CONDITIONAL | 是，P1 |
| 数据清理实际实现 | CONDITIONAL | 是，P1 |
| admin_audit_logs 防篡改承诺 | 不成立/需修正 | 是，需修正文档 |
| 12 组隔离测试 | 全部 NOT_RUN | 是，不能据此准入 |

---

# 九、最终裁定与准入边界

## 9.1 最终裁定

**CONDITIONAL（有条件不通过最终验收）**

第八轮材料已经修复或改善了第七轮多项设计缺陷，但出站任务外部副作用不确定时的恢复机制仍然缺失，这是影响消息可靠性和客户体验的 P0 问题。

按钮执行恢复、备份恢复、清理机制及跨进程写入协调也尚未达到完整实现验收标准。

## 9.2 当前不批准事项

在 CEO 另行明确授权前：

- 不批准 Telegram Monitor V2.2 正式生产准入；
- 不批准将第八轮标记为 PASSED；
- 不批准出站发送模块正式实现；
- 不批准连接真实 Telegram 账号或使用真实凭证进行测试；
- 不批准在生产数据库上执行迁移、清理、恢复或写入；
- 不批准生产部署；
- 不得将 NOT_RUN 测试描述为已通过。

## 9.3 可以考虑的有限工作

CEO 可以单独批准在隔离环境中进行局部原型开发，但必须满足：

- 使用虚拟数据库或独立测试数据库；
- 使用模拟 Telegram API，不连接真实 Telegram；
- 使用虚拟账号、虚拟消息和测试数据；
- 不读取、复制或使用真实生产凭证；
- 不修改生产数据库；
- 不部署到生产环境；
- 只验证本报告列出的局部状态机和恢复逻辑。

---

# 十、提交 CTO 的第九轮定点整改任务

**任务对象：** Gemini（CTO）  
**审查方：** ChatGPT（CTRO）  
**最终决策人：** CEO  
**整改性质：** 定点整改，不推倒重建  
**当前结论：** CONDITIONAL

## 10.1 P0：出站发送状态机必须闭环

请重点提交：

1. 明确 `CLAIMED` 与 `SENDING` 的区别。
2. 增加发送开始时间字段，例如 `send_started_at`。
3. 补充出站 Worker 续租函数。
4. 补充调用 Telegram API 前的状态转换事务。
5. 补充 API 明确成功后的条件提交逻辑。
6. 补充 API 明确失败且可确认未产生副作用时的重试逻辑。
7. 补充超时、连接中断、回执丢失时进入 `UNKNOWN_ERROR` 的逻辑。
8. 补充 `UNKNOWN_ERROR` 的 MTProto 对账闭环。
9. 明确只有在能够证明没有发送时才允许重试。
10. `SENDING` 过期不得直接盲目重发。
11. 所有关键 UPDATE 必须绑定任务身份、Worker、Claim Token 和预期状态。
12. 检查并处理 `rowcount`，防止日志与实际数据库状态不一致。

目标状态链路：

```text
PENDING
  |
  v
CLAIMED
  |
  | 发送前条件提交
  v
SENDING
  |
  +---- API 明确成功 ----------------> SENT
  |
  +---- 明确未产生副作用的失败 -----> RETRY_WAIT
  |
  +---- 结果不确定 -----------------> UNKNOWN_ERROR
                                         |
                                         v
                                      MTProto 对账
                                         |
                      +------------------+------------------+
                      |                  |                  |
                      v                  v                  v
                    SENT          MANUAL_REVIEW     证明未发送
                                                        |
                                                        v
                                                    RETRY_WAIT
```

## 10.2 P1：按钮执行租约与恢复

请提交：

- `execution_expires_at` 或等效租约字段；
- 按钮执行续租逻辑（如确有需要）；
- EXECUTING 超时 Reaper；
- 明确失败与结果未知的状态区分；
- BAN/MUTE 等外部副作用的幂等核验；
- 最终 UPDATE 的身份条件和 rowcount 检查；
- 无法确认结果时转人工核验的路径。

## 10.3 P1：备份与恢复实际代码

请提交：

- 可执行的备份函数；
- WAL 一致性处理；
- 备份失败处理；
- 文件完整性检查；
- 备份保留策略；
- 独立目录恢复流程；
- 恢复后 `integrity_check`；
- 关键业务数据一致性核验；
- 一次隔离恢复演练的实际日志和结果。

仅有备份文件大小大于 0 或 `integrity_check=ok`，不足以证明恢复能力。

## 10.4 P1：数据清理实际实现

请提交：

- 清理 SQL；
- 清理任务调度；
- 明确保留期限；
- 事务边界；
- 清理日志；
- 失败处理；
- 清理前后数量核验；
- 关联表完整性保护。

保留 `FAILED` 和 `MANUAL_REVIEW` 等关键状态记录，除非另有经过审查的归档方案。

同时修正 `admin_audit_logs` 的“永久保存、防篡改”表述。若没有外部只追加日志、权限隔离或哈希链等机制，不得宣称 SQLite 普通表可以防篡改。

## 10.5 MTProto 对账补充

请提交：

- `rowcount` 检查；
- 完整分页实现；
- `offset_date` 方向与边界验证；
- 时间窗口扫描完成判定；
- 零宽字符指纹的隔离验证；
- 扫描不完整、权限不足、匹配不唯一时的安全状态处理。

不得以“设计上会扫描三次”代替实际测试结果。

## 10.6 第九轮提交范围

为避免材料膨胀，本轮只要求提交：

1. 相关 DDL 差异；
2. 事务四：出站发送完整状态机；
3. 事务五：MTProto 对账修订；
4. 事务六：按钮执行恢复修订；
5. 备份与恢复代码；
6. 数据清理代码；
7. 状态流转图；
8. P0/P1 整改对照表；
9. 隔离测试用例与实际执行结果。

不得为了第九轮整改而重写整个架构。

## 10.7 安全边界

在 CEO 明确授权前：

- 不连接真实 Telegram；
- 不使用真实 Telegram 凭证；
- 不发送真实消息；
- 不对真实用户执行 BAN/MUTE；
- 不操作生产数据库；
- 不部署生产环境；
- 不执行生产数据清理；
- 不进行未经授权的 Git 提交、推送或其他破坏性操作。

---

# 十一、CTRO 最终意见

第八轮的主要进步是真正开始把任务身份、Fencing Token、Claim Token、事务边界和对账完整性落实到具体伪代码中，而不再只停留在概念说明。

但系统最重要的外部副作用仍然是 Telegram 消息发送。只要“请求可能已经成功、但本地没有拿到回执”这一情况没有被状态机明确处理，系统就不能声称具备可靠的自动重试能力。

**本轮最重要的判断不是“是否增加了更多字段”，而是：当 Worker 在任意一步崩溃时，系统是否能够证明消息没有发送、已经发送，或必须暂停等待人工核验。**

在 P0-05 完整闭环、按钮执行恢复补齐，并取得必要的隔离测试证据之前，CTRO 不建议批准正式生产准入。

---

## 对话健康提醒

本报告是基于第八轮 Dropbox 材料进行的静态技术审查整理版。第八轮所列 12 组隔离测试均为 `NOT_RUN`；本报告没有将设计说明、伪代码或测试计划认定为实际测试通过。未执行真实 Telegram API 验证、真实并发测试、生产数据库操作或恢复演练。

**归档说明：** 本 Markdown 文件是依据当前可恢复的第八轮审查记录重建的完整整理版，并非此前助手回复的逐字导出。