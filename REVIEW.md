# 湖表在线备份可恢复性代码审查报告（REVIEW.md）

- 审查对象：`lake.py`、`backup.py`、`reclaim.py`（实现、样例与原断言均未改动）
- 契约依据：`CONTRACT.md`；证据入口：`evidence_gate.py record|verify|demo`
- 证据文件：`review_cases.json`（审查用例与源码定位）、`review.json`（由原入口 `lab.exercise` 实际重放生成）
- 审查范围：正常完成、受控复制失败、提交/回收与备份的可控交错；不含任意进程崩溃恢复，不实现生产修复
- 结论摘要：**当前基线不满足"成功备份可恢复出声明版本完整行集合"的发布承诺，建议阻断发布（NO-GO）**。静止场景与真实死对象回收场景成立；但存在两类可稳定复现的可恢复性缺陷（版本选择与目录快照脱节、陈旧候选回收无裁决），且摘要核验全部通过，说明"摘要正确"不足以证明可恢复。

## 1. 审查方法与判定口径

1. 保留全部生产源码、`samples.json`、`lab.py`、`checks.py` 及原断言，不修改实现。
2. 用例只通过 `lab.exercise(spec)` 的原入口重放；每个检查点回调在对应源码操作之后同步执行，不引入线程。
3. 期望行（`expected_rows`）由 `lab.py` 按输入批次的覆盖语义独立计算（`lab.py:23-31`），不调用 `Lake.read`；实际行来自对带完成标记副本执行 `backup.restore` 后的 `restored.read()`（`lab.py:77-83`）。
4. 判定不只看摘要或文件存在：必须同时满足 `complete=true`、`declared_version == catalog_version`、`actual_rows == expected_rows` 才算可恢复承诺成立。
5. 用必要单因与组合对照区分触发因子、掩盖因子与合法交错：每个缺陷都配一个仅改变关键时点的对照用例。

## 2. 源码事实（含行号）

### 2.1 版本选择与目录快照之间存在无保护窗口

- `backup.py:20-24`：备份在一个短事务里读取 head 版本与 manifest 条目，随即 `c.commit()` 释放事务；版本与对象集合只保存在进程局部变量 `version`、`entries` 中。
- `backup.py:25`：发出 `backup.selected` 事件后，选择事务已经结束。
- `backup.py:26`：`clone_catalog` 经 `lake.py:109` 的 SQLite online backup 复制目录；此时复制到的是**调用当下**的 head，可能已不是第 22 行选定的版本。
- `backup.py:29-40`：载荷对象却按第 23 行旧 `entries` 复制并逐文件核验摘要（`backup.py:36`）。
- `backup.py:41-42`：回执 `complete.json` 写入的是旧 `version`；完成标记并不校验目录快照与回执是否同一版本。
- `backup.py:54-63`：`restore` 信任完成标记，克隆目录的 head 即恢复后的当前版本；若目录 head 引用了未复制对象，`lake.py:97` 的 `read_bytes()` 抛 `FileNotFoundError`（契约允许的缺文件表现）。
- 提交侧 `lake.py:83-87` 在单个 `BEGIN IMMEDIATE` 事务内插入新 snapshot/manifest 并更新 head，提交成功即对外发布新版本，备份无法暂停它。

### 2.2 回收规划与裁决分离，sweep 不做活性复检

- `reclaim.py:4-6`：`plan` 只按规划瞬间 head 的 manifest 计算 live 集合，凡不在 live 中的数据文件都成为候选；候选只是文件名列表，不记录规划时版本。
- `reclaim.py:7`：规划事务在返回候选前即提交，不持有任何跨调用保护。
- `reclaim.py:11-20`：`sweep` 对每个候选文件开新事务（`reclaim.py:14-15`），仅以 `path.exists()`（`reclaim.py:17`）为条件直接 `unlink`（`reclaim.py:18`），**不复检该对象是否已被新 head 或在途备份引用**，也没有任何备份登记/钉取协议。
- 内容寻址复用使风险成立：`lake.py:79-81` 同摘要载荷不换名；`return` 批次与 `initial` 的 north 载荷摘要相同（均为 `2c32600e…jsonl`），旧候选可被新版本再次引用。
- `lake.py:90-102`：前台 `read` 用 `BEGIN IMMEDIATE` 写事务包裹文件读取，但其生命周期仅覆盖单次 read 调用，无法跨载荷 I/O 借给备份使用；契约亦明确禁止把这种长事务作为备份保护方案。

### 2.3 复制失败的残留与责任

- `backup.py:9-10`：目标目录在任何复制前创建；`backup.py:33-34` 在指定序号复制前抛受控 `OSError`。
- `backup.py:45-47`：异常路径只追加 `backup.failed` 事件并重新抛出；`backup.py:48-49` 的 `finally` 仅保证写 `events.json`。
- 失败后没有完成标记（`backup.py:42` 不可达），`restore` 在 `backup.py:54` 读 `complete.json` 即抛 `FileNotFoundError`，拒绝恢复不完整目录——这部分符合契约；但已创建的目标目录、已复制对象和目录副本没有任何清理或责任归属。

## 3. 用例重放结果（case:ID 与 review_cases.json 一一对应）

下表"实际"列均取自 `review.json` 的真实重放输出；"独立期望"由批次覆盖语义计算。对象简名：`N0=2c32600e…jsonl`（initial/return 的 north）、`S0=2b0e713c…jsonl`（south）、`N2=6731eb0f…jsonl`（change 的 north）。

### case:C01-quiescent-baseline（静止基线，单因空白对照）

- 触发顺序：提交 initial(v1) → selected(v1) → catalog → 复制 S0、N0 → complete；无交错。
- 独立期望：`complete=true`，声明=目录=1，行 `[["north","n1",10],["north","n2",0],["south","s1",20]]`。
- 实际：完全一致，`restore_error=null`。
- 结论：静止状态下承诺成立，作为所有交错用例的基线；证明缺陷需要交错因子激活，而非路径恒错。

### case:C02-commit-between-selection-and-catalog（原失败复现，单因：提交落在选择/快照窗口）

- 触发顺序：initial(v1) → selected(v1，条目 N0、S0) → **在 `backup.py:25` 之后、`backup.py:26` 之前提交 change(v2)**（`lake.py:86-87` 更新 head=2）→ catalog 复制到 head=2 的目录 → 仍按 v1 条目复制 N0、S0 并通过摘要核验 → complete 回执声明 v1。
- 独立期望：声明版本 1 的完整行集合（同 C01），恢复目录当前版本=1。
- 实际：`complete=true`、`declared_version=1`、`catalog_version=2`、`actual_rows=null`、`restore_error=FileNotFoundError`。恢复目录 head=2 的 manifest 引用 N2，而 N2 从未复制，`lake.py:97` 抛错。
- 对照定位：与 C03/C04 仅改变提交检查点。提交发生在 catalog 之后即无害，故**必要触发因子是"提交位于选择事务提交（`backup.py:24`）与目录快照（`backup.py:26`）之间"**，而非提交本身。
- 关键观察：两个已复制文件摘要核验全部通过，完成标记正常写入——摘要正确不能替代可恢复性。该用例即 `checks.py repro` 稳定展示的 `BACKUP_RESTORE_MISMATCH`，证据中原失败保留。

### case:C03-commit-after-catalog-control（合法交错对照）

- 触发顺序：selected(v1) → catalog（head=1）→ **在 `backup.py:27` 之后提交 change(v2)** → 复制 v1 对象 → complete。
- 独立期望/实际：声明=目录=1，实际行等于期望，`restore_error=null`。
- 结论：目录快照一旦先于提交取得，源表继续前进不影响已冻结副本；与 C02 对照把缺陷窗口缩小到 selected→catalog 之间，同时证明"不暂停提交"本身可与可恢复性兼容。

### case:C04-commit-mid-copy-control（合法交错对照）

- 触发顺序：selected(v1) → catalog → 复制 S0 后（`backup.py:40` 检查点）提交 change(v2)，N2 由 `lake.py:79-80` 新写入，不动既有对象 → 复制 N0 → complete。
- 独立期望/实际：声明=目录=1，实际行等于期望。
- 结论：载荷复制期间提交是合法交错；内容寻址（`lake.py:81`）保证新版本只增文件、不改旧文件。与 C02 配合证明问题不在"复制期间有提交"，而在目录快照时点。

### case:C05-sweep-dead-then-rereference-legal（回收+再次引用的合法交错对照）

- 触发顺序：initial(v1) → change(v2，N0 成为死对象) → selected(v2，条目 N2、S0) → catalog(head=2) → **catalog 之后**依次 plan（候选 N0）、sweep（删除 N0）、提交 return(v3，按 `lake.py:79` 重新写出 N0 并再次引用) → 复制 v2 对象 → complete。
- 独立期望：v2 行 `[["north","n3",30],["south","s1",20]]`，恢复目录版本=2。
- 实际：声明=目录=2，实际行等于期望，无错误。
- 结论：规划时为死对象的候选被删、随后载荷被再次引用，只要裁决发生在备份目录快照取得之后且对象不在备份所需集合内，即为合法交错；用于与 C06 对照，区分"回收行为本身"与"回收裁决缺失"。

### case:C06-stale-candidate-sweep-during-inflight-backup（组合缺陷：陈旧候选 + 在途备份 + 再次引用）

- 触发顺序：initial(v1) → change(v2) → **plan 暂存候选 [N0]**（`reclaim.py:6`，此时 N0 不被 head=2 引用）→ selected(v2，所需 N2、S0) → selected 后提交 return(v3)：head=3 再次引用 N0（`lake.py:81`，文件已存在不重写）→ catalog 复制到 head=3 → 复制 S0 成功 → **`after_file:0` 检查点执行 sweep，陈旧候选列表未经复检直接 `unlink` N0（`reclaim.py:17-18`）** → 复制 N2（成功）→ complete 回执声明 v2。
- 独立期望：v2 完整行集合（N3/S1），恢复目录当前版本=2。
- 实际：`complete=true`、`declared_version=2`、`catalog_version=3`、`actual_rows=null`、`restore_error=FileNotFoundError`。恢复目录 head=3 引用 N0，但 N0 既未被 v2 备份复制、又已在源端删除。
- 因子分解：
  - 与 C08 对照：回收真正的死对象（无再次引用、无在途引用）完全合法 → 回收不是充分因。
  - 与 C07 对照：同一陈旧候选缺陷在无备份交错时单独即可损坏源表 head → 在途备份不是必要因。
  - 本用例的必要组合是"规划后引用关系变化 + sweep 不裁决"；备份交错决定了损坏外溢为一份**带完成标记却不可恢复的副本**，且全程摘要核验通过，具有强掩盖性。

### case:C07-stale-candidate-sweep-corrupts-head-single-factor（单因缺陷：回收裁决损坏当前 head）

- 触发顺序（全部在备份开始前的 `before` 中）：initial(v1) → change(v2) → plan 得候选 [N0] → 提交 return(v3)，head=3 再次引用 N0 → sweep 按旧候选删除 N0。
- 独立期望：head=3 可读，行同 initial（N1/N2/S1）。
- 实际（lab 重放）：备份在 `backup.py:35` 读 N0 前即抛 `FileNotFoundError`，`complete=false`、`backup_error=FileNotFoundError`、`restore_error=FileNotFoundError`。独立脚本另证：源表 `lake.version()=3` 而 `lake.read()` 直接抛 `FileNotFoundError`（`lake.py:97`），数据目录仅剩 S0、N2。
- 结论：这是**不依赖任何备份交错的独立回收缺陷**——规划与裁决之间没有版本固定，sweep 的删除决策完全基于过期列表。它同时是 C06 的单因子证据。

### case:C08-collect-truly-dead-during-backup-legal（合法回收对照）

- 触发顺序：initial(v1) → change(v2) → selected(v2) 时执行 `collect`：plan/sweep 在同一同步序列内完成，候选 N0 此刻确实不被任何版本的 head 引用且不在 v2 备份所需集合中 → 正常复制完成。
- 独立期望/实际：声明=目录=2，v2 行完全匹配，无错误。
- 结论：备份期间执行回收本身合法，契约允许的交错无需暂停回收；问题仅在候选过期且无裁决（对照 C06/C07）。

### case:C09-copy-failure-first-file（复制失败边界：序号 0）

- 触发顺序：selected(v1) → catalog → `before_file:0` 处 `backup.py:33-34` 抛 `OSError` → `backup.failed` → finally 写 events。
- 独立期望：`backup_error=OSError`、无完成标记、`restore` 拒绝不完整目录。
- 实际：完全符合；`complete=false`，`restore_error=FileNotFoundError`（缺 `complete.json`，拒绝点 `backup.py:54`）。
- 结论：失败检测与恢复拒绝在首个文件边界成立；残留责任仍未定义（见第 5 节 R4）。

### case:C10-copy-failure-after-one-file-residual（复制失败：已产生部分载荷残留）

- 触发顺序：同 C09，但 `fail_at=1`：S0 已在 `backup.py:38` 落盘后，N0 复制前失败。
- 独立期望：无完成标记、恢复被拒绝；失败不得污染源表。
- 实际：`complete=false`、`OSError`、恢复被拒绝；源表 `read()` 仍返回完整 v1 行（源不受污染，符合预期）。独立目录检查证实残留：目标下留存 `catalog.db`、`data/S0`、`events.json`，无 `complete.json`；代码中没有任何清理路径（`backup.py:45-49`）。
- 结论：拒绝恢复正确，但"目标目录可直接清理"的文档表述把责任留给了运维且无接口约定；重试还会因 `backup.py:9` 的 `exist_ok=False` 直接失败，必须人工清理。

### case:C11-copy-failure-overlapping-commit-control（失败与提交组合对照）

- 触发顺序：selected(v1) 后立即提交 change(v2)（复刻 C02 的窗口因子），首个文件复制前注入失败。
- 实际：`backup_error=OSError`、`complete=false`、恢复被拒绝；未产生完成标记，故 C02 类串版被失败前置截断。
- 结论：复制失败是确定性终止因子，不会因交错而伪造成功；与 C02 对照说明"成功标记 + 错版本"只能经由完成路径产生，失败路径不存在该风险。

### case:C12-copy-failure-after-reclaim-rereference（失败与回收组合对照）

- 触发顺序：复刻 C06 的 plan→return 再引用→sweep 删除 N0，在序号 1 注入复制失败（先删后败）。
- 实际：`backup_error=OSError`、`complete=false`、恢复被拒绝；events 以 `backup.failed` 收尾。
- 结论：回收损坏与注入失败叠加时，失败语义仍确定；但残留目录同时包含过期 catalog 副本与部分载荷，且源端 N0 已被 C07 类缺陷删除——失败路径没有释放任何保护（现状下也无保护可释放），残留物与源损坏的责任边界在代码中均无归属。

## 4. 根因综合：取得、释放与裁决的责任现状

| 责任点 | 契约要求 | 现状（源码证据） | 对应用例 |
|---|---|---|---|
| 版本选择的保护取得 | 选定可读版本后，正常提交/回收不得令备份失败或串版 | 选择在 `backup.py:24` 即提交释放，无钉取/登记；目录快照在 `backup.py:26` 另取，两者无版本绑定 | C02 vs C03/C04 |
| 保护的释放 | 完成或失败后明确释放 | 无保护实体，故无释放动作；失败只写事件（`backup.py:46-49`） | C09-C12 |
| 候选回收的裁决顺序 | 备份所需文件不得被回收；不得暂停回收 | `sweep` 仅查文件存在性（`reclaim.py:17`），不查 head manifest 复检、不查在途备份 | C06/C07 vs C08 |
| 完成标记的含义 | 标记=声明版本完整可恢复副本 | 标记只证明回执内文件摘要正确（`backup.py:36`），不证明目录与回执同版本、不证明行集合完整 | C02、C06 |
| 失败残留责任 | 拒绝恢复不完整目录，并明确清理责任 | 拒绝恢复成立（`backup.py:54`）；残留无清理、重试被 `backup.py:9` 阻断，责任未定义 | C10、C12 |
| 事务持有范围 | 不跨载荷 I/O 持目录写事务；不得用前台 read 长事务兜底 | 备份侧已满足（事务仅覆盖 `backup.py:20-24`）；`lake.py:92` 的 read 写事务不可复用，修复必须另选短事务机制 | 全部 |

## 5. 最小兼容修复建议（仅建议，本轮不实现）

以下建议保持现有接口签名、样例与断言不变，不暂停提交或回收，且不在载荷复制期间持有目录写事务。

- **R1 备份保护的取得与释放（针对 C02、C06）**：在 catalog.db 新增仅内部使用的登记表（如 `backup_pins(version, objects_json, created_at)`）。备份在选择版本后用一个**短写事务**插入钉取记录并提交即完成取得；`finally` 中用另一个短事务删除该记录完成释放（含失败路径）。载荷复制期间不持有任何目录事务，前台提交与回收照常进行。
- **R2 版本选择与目录快照绑定（针对 C02）**：钉取事务内同时读定 head 版本与 manifest；`clone_catalog`（`backup.py:26`）完成后增加一次性一致性校验——克隆目录的 head 必须等于钉取版本、其 manifest 对象集合必须等于回执将声明的集合，不一致即按失败处理且不写 `complete.json`。`restore` 侧增加防御性校验：目录 head 与回执版本一致才返回 Lake。该方案不依赖把 `lake.py:92` 的读事务延长到复制阶段。
- **R3 回收裁决顺序（针对 C06、C07）**：`plan` 保持现状（仅产生候选）；把活性复检放到 `sweep` 的每个短删除事务内——删除前同时核对当前 head 的全量 manifest（不限于规划时版本）与所有有效 `backup_pins` 的对象并集，命中即跳过该文件并在返回值中标注 skipped。规划/裁决仍可与提交、备份交错，无需任何暂停。
- **R4 失败残留与责任（针对 C09、C10、C12）**：在 `backup` 的异常路径中，于 `events.json` 落盘后由备份任务自身负责清理目标目录（或提供显式 `abort(destination)` 入口并在文档中规定重试前必须调用）；明确责任归属：备份任务负责取得/释放钉取与清理自身残留，回收者只服从 R3 的裁决，提交者无新增责任。
- **R5 证据与文档**：完成标记的语义在文档中改为"目录快照版本、manifest 集合与回执三者一致且全部对象摘要核验通过"；保留 `checks.py repro` 作为修复前的回归证据，修复后再以同入口转为通过断言。

## 6. 发布意见

- **NO-GO（阻断在线备份的可恢复承诺发布）**，直到 R1-R3 落地并有回归覆盖；R4 可与修复同批补齐文档与责任约定。
- 可继续发布的范围：静止备份（C01）、目录快照后的提交交错（C03、C04）、真实死对象回收（C05、C08）、复制失败的检测与恢复拒绝（C09-C12 的失败语义部分）。
- 必须告知维护方的现存风险：
  1. 选择与快照窗口内的提交可产生"标记成功、恢复即缺文件"的副本（C02，即公开 repro，证据已保留）。
  2. 规划后被再次引用的对象可被过期候选删除，单独损坏源表 head（C07），并可在备份期间外溢为不可恢复副本（C06）。
  3. 复制失败后的目标残留无自动清理、无重试通道（C10）。
- 所有结论均以 `review.json` 的原入口重放结果为准；建议方案的预测效果未计入当前证据。
