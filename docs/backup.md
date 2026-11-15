# 在线备份维护说明

本说明描述仓库**当前实现的实际行为与边界**。审查发现的缺陷、用例证据与修复建议见 `REVIEW.md`（用例以 `case:ID` 引用）；建议项 R1–R4 尚未实现，本说明不得把建议效果表述为现有保证。

## 目录与对象模型（现状）

- 目录库由 `catalog.db`（head、snapshots、manifest 三张表）与 `data/` 下的不可变对象组成；对象文件名为载荷内容的 SHA-256 摘要加 `.jsonl`，相同载荷跨分区、跨版本复用同一对象，再次引用不换名（`lake.py:79-81`）。
- 提交在单个 `BEGIN IMMEDIATE` 事务内发布新版本（`lake.py:67-87`），只新增对象、不修改既有对象；未提及的分区沿用上一版本。
- 历史读只在所需对象文件仍存在时可用；未知版本抛 `ValueError`（`lake.py:48-49`），对象缺失抛 `FileNotFoundError`（`lake.py:97`）。
- 前台 `read` 虽以 `BEGIN IMMEDIATE` 开启写事务（`lake.py:92`），但其生命周期仅覆盖单次读取调用，**不能**作为在线备份跨载荷复制的保护手段。

## 备份与恢复的实际语义（现状）

1. `backup.backup` 在一个短事务中读取当前 head 版本与 manifest 条目后立即提交释放（`backup.py:20-24`）；随后才复制 SQLite 目录（`backup.py:26`），再按先前读到的条目复制对象并逐文件核验摘要（`backup.py:35-39`），最后写 `complete.json` 与 `events.json`（`backup.py:42-49`）。
2. **完成标记的实际含义仅限**：回执所列对象均已复制且摘要核验通过。它**不证明**目录快照与回执声明的是同一版本，也**不证明**声明版本的完整行集合可恢复。静止状态下两者恰好一致（case:C01）；存在交错时可产生带完成标记却不可恢复的副本（case:C02、case:C06）。
3. `backup.restore` 只接受含 `complete.json` 的目录，缺失完成标记时在读取标记处抛 `FileNotFoundError` 并拒绝恢复（`backup.py:54`）；恢复目录的当前版本取自克隆目录的 head，而非回执声明版本。
4. 载荷复制期间不持有目录写事务，提交与回收无需、也不得被暂停；目录快照取得之后的提交（case:C03、case:C04）与真实死对象回收（case:C05、case:C08）已验证为合法交错，不影响已冻结副本。

## 回收的实际语义与风险（现状）

- `reclaim.plan` 仅按规划瞬间 head 的 manifest 计算非 live 对象候选（`reclaim.py:4-6`），返回纯文件名列表，不记录规划版本；规划事务在返回前即提交（`reclaim.py:7`）。
- `reclaim.sweep` 对候选逐个开短事务，仅以文件仍存在为条件直接删除（`reclaim.py:14-20`），**不复检**对象是否已被新 head 或在途备份引用，系统中也没有备份登记/钉取机制。
- 因此：规划之后若提交使旧载荷被再次引用（内容寻址复用），按旧候选执行 sweep 会删除当前 head 仍在引用的对象，单独即可使源表当前版本不可读（case:C07）；与在途备份交错时还会产生声明版本与目录版本不一致、恢复缺文件的"成功"副本（case:C06）。回收规划时确实无引用的死对象（case:C08）不在此风险内。

## 复制失败与残留（现状）

- `fail_at` 注入的受控失败在对应文件复制前抛 `OSError`（`backup.py:33-34`）；异常路径只记录 `backup.failed` 事件并重新抛出（`backup.py:45-47`），不写完成标记，恢复会被拒绝。源表不受失败污染。
- 失败目标目录会残留已建目录、目录副本、已复制对象与 `events.json`（case:C10）；代码没有自动清理路径，且目标已存在时重新备份会在 `backup.py:9` 直接失败。**重试前必须由人工或运维流程清空目标目录**；残留清理与保护释放的责任归属尚待 R4 落地。

## 核验与证据入口

- `python3 checks.py verify`：正常提交、历史读、静止备份与复制失败的原断言回归。
- `python3 checks.py repro`：经原入口稳定展示 `BACKUP_RESTORE_MISMATCH` 并退出 1，这是保留的缺陷证据，不是待修好的测试。
- `python3 checks.py demo`：展示静止场景的版本、事件与恢复行。
- `python3 evidence_gate.py record|verify|demo`：按 `review_cases.json` 经 `lab.exercise` 原入口重放生成/核验 `review.json`；verify 同时重跑 `checks.py verify`、校验源码定位与报告引用，并要求证据中保留原失败。
- 发布口径：在 R1–R3（备份钉取与释放、目录快照一致性校验、sweep 活性裁决）落地前，不得对外承诺"成功备份即可恢复声明版本完整行集合"；当前发布意见为 NO-GO，详见 `REVIEW.md` 第 6 节。
