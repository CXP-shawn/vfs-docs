# AUDIT.md — docs/01~05 命题表（本 agent 产出）

> **来源声明**：本文件**不是**上游审计 agent 的产出。本次构建未能从同工作区数据库或本地 downloads 目录取得 AUDIT.md，经用户明确授权后，由本 agent（builder）基于仓库内仅有的两份事实来源自行生成：
>
> - `README.md`（顶层，自带"已基于 shell + execute_js + file 工具真实验证"的声明）
> - `docs/experiments/2026-04-20-sandbox-fs.md`（Run ID `Rmo7btrg0-126701172164598`，2026-04-20，40+ 探针原始记录）
>
> 仅以上述两份文件为 **✅ 事实** 的锚点。凡超出它们的断言一律判为 **⚠️ 推测**。命题若与实测冲突则判为 **❌ 矛盾**。

## 0. 术语与简称

- **FACT-RM-n**：README.md 第 n 条实测
- **FACT-EX-n**：experiments.md 维度 n 的原始结果
- **PROP-Dx-n**：docs/0x-*.md 第 n 条待审命题

三类分类符号：

| 符号 | 含义 | 处理规则 |
|---|---|---|
| ✅ | 与事实来源一致 | 保留并按"深度化方向"扩写 |
| ⚠️ | 事实来源中未直接证实（推测/设计推理） | 原文保留，段首插 ⚠️ 标注 |
| ❌ | 与事实来源冲突 | 按事实来源改写，或整段删除 |

---

## 1. docs/01-workspace-uploads.md — 命题表

| # | 命题 | 分类 | 事实锚点 / 说明 |
|---|---|---|---|
| 1.1 | `downloads` 命名空间包含 5 个探针文件（含 `test_readme.md` 2279B 用户上传、`doubao_avatar.png` 26799B 用户上传） | ✅ | FACT-RM 的"实测语义"一节；这些文件也出现在 experiments.md 附录 Transcript 2 |
| 1.2 | `shell` 对 `/agent/downloads` 是 ✅ 读 / ❌ 写 (EROFS) | ✅ | FACT-EX-1 P07；FACT-RM 权限矩阵表 |
| 1.3 | `execute_js` + `twin:fs` 经 `/agent/current/downloads/` 可读可写 | ✅ | FACT-RM 权限矩阵；FACT-EX-2 写入尝试 |
| 1.4 | `file` 工具通过 `filename` 参数操作，返回中不含绝对路径 | ✅ | FACT-RM "file 工具的消费方式"；FACT-EX-1 P04 |
| 1.5 | "**shell 不能往 `downloads` 写**。如需从 shell 把产物放到 `downloads`，必须走 `execute_js` 拷贝（或反过来：让 shell 写到 `/agent/generated`，然后应用层自己约定从 `generated` 读）。" | ✅ | FACT-RM 同款表述；也与 FACT-EX-2 一致 |
| 1.6 | "写 `downloads`"的合法工具集合： `execute_js`(`twin:fs`)、`file.download_via_curl`；非法：`shell`、其它工具 | ✅ | FACT-RM 权限矩阵；FACT-EX-2 |
| 1.7 | 推荐读取样例 `pdftotext /agent/downloads/foo.pdf /agent/generated/foo.txt` | ✅ | FACT-EX 附录显示 poppler-utils 可用 + `/agent/generated` 可写 |

**结论**：01 文档几乎全部 ✅，无 ❌，无独立 ⚠️ 命题。深度化方向：POSIX 读写语义、VFS 命名空间分离、btrfs ro subvol 与 EROFS 原理、上传链路。

---

## 2. docs/02-file-tool.md — 命题表

| # | 命题 | 分类 | 事实锚点 / 说明 |
|---|---|---|---|
| 2.1 | `download_via_curl` 落盘到 `downloads` 命名空间，返回 `bytes_written`、`content_hash`（sha256），不返回绝对路径 | ✅ | FACT-EX-1 P04；FACT-RM |
| 2.2 | 下一次 `execute_js` 可读 `/agent/current/downloads/<name>`，下一次 `shell` 可读 `/agent/downloads/<name>` | ✅ | FACT-EX-1 P01、P02 |
| 2.3 | "`/agent/current/generated/<name>` — 不存在"、"`/agent/generated/<name>` — 不存在" | ✅ | FACT-EX-2 的兄弟目录（非镜像）探测 |
| 2.4 | "`file` 工具 = 一个跨沙箱的 HTTP 代理 + `downloads` 命名空间写入器" | ⚠️ | 第一句"跨沙箱 HTTP 代理"合事实（shell 无网络、execute_js 无 curl）。**第二句"`downloads` 命名空间写入器"与 FACT-EX-1 P06 相矛盾**——P06 证实 `file.upload_via_curl` 也能读取 `/agent/generated` 中的文件。因此"file 工具只操作 downloads"的含义不成立。标 ⚠️ 且需修正 |
| 2.5 | 支持 action：`download_via_curl`、`upload_via_curl`（含 multipart 等） | ✅ | 工具描述；FACT-EX-1 P04 P06 |
| 2.6 | "**`upload_via_curl` 只能从 `downloads` 命名空间读。从 `generated` 产生的文件若要上传，要先拷到 `downloads`。**" | ❌ | 与 FACT-EX-1 P06 直接冲突：实测 `file.upload_via_curl(filename='probe_P1SG-...', url='httpbin.org/post')` 成功从 `/agent/generated` 读取并上传。**必须删除该段，替换为"upload\_via\_curl 命名空间无关"的正确描述** |
| 2.7 | "`file` 工具不直接解压压缩包 —— 解压要用 `shell` (`unzip`/`tar`) 或 `execute_js`" | ⚠️ | 工具描述里未声明解压能力；但也未见直接实验。作为设计推理可接受，保留并标 ⚠️ |

**结论**：02 文档需要一次重要修正（2.6 ❌），一次措辞细化（2.4 ⚠️），新增"命名空间无关"的说明。

---

## 3. docs/03-execute-js-fs.md — 命题表

| # | 命题 | 分类 | 事实锚点 / 说明 |
|---|---|---|---|
| 3.1 | `fs.readdirSync('/')` 返回 `['agent','workspace']` | ✅ | FACT-EX-2 原始记录 |
| 3.2 | `fs.readdirSync('/agent')` 返回 `['current']` —— 没有 `/agent/downloads`、`/agent/generated`、`/agent/context` | ✅ | FACT-EX-2 原始记录 |
| 3.3 | `fs.readdirSync('/agent/current')` 返回 `['downloads','generated']` | ✅ | FACT-EX-2 |
| 3.4 | "`fs.readdirSync('/workspace')` 在 `execute_js` 里不可读取（未探测到内容）" | ⚠️ | experiments 明确写道 execute_js 对 `/workspace/chat` 写 EACCES、`/tmp` 和 `/workspace` 不可见。"不可读取"是对可见性的一种推断描述，严谨地说是"该路径在 execute_js 视图内不可见/写被拒绝"。保留原文但标 ⚠️ |
| 3.5 | `twin:fs` 可写路径只有 `/agent/current/downloads/*` 与 `/agent/current/generated/*`，其它 EACCES | ✅ | FACT-EX-2 原始记录 |
| 3.6 | 多次 `writeFileSync → storeVar → readFileSync` 链路持久性 100% 成功 | ✅ | FACT-EX-2；FACT-RM |
| 3.7 | 跨工具可见性 4 行矩阵（ejs→shell、shell→ejs、file→ejs） | ✅ | FACT-EX-1 P01–P06 |
| 3.8 | "**不可用**：`crypto` 模块（实测抛 `Cannot find module 'crypto'`）" | ✅ | FACT-EX 方法论一节："execute_js 里没有 crypto 模块，因此用 32 位 XOR 折叠做轻量相等性校验" |
| 3.9 | "`Buffer.isBuffer` 的行为与标准 Node 不完全一致" | ⚠️ | 实验文档未做此项探测；作为经验式告诫保留并标 ⚠️ |
| 3.10 | "**单次 `execute_js` 调用内的所有写操作是原子的** —— 脚本抛异常时所有写入都被丢弃" | ⚠️ | 来自 `execute_js` 工具的官方描述而非本仓库实验。README 作者也备注"实测未做针对性验证，按官方声明执行"。保留并标 ⚠️ |

**结论**：03 文档主要 ✅。3.4 / 3.9 / 3.10 标 ⚠️。

---

## 4. docs/04-shell-tool.md — 命题表

| # | 命题 | 分类 | 事实锚点 / 说明 |
|---|---|---|---|
| 4.1 | "`shell` 用的是**长驻工作目录**模式：先把 session 物化到本地磁盘，跑完命令再同步回来" | ⚠️ | experiments 横切观察中写道"shell 沙箱是一个工作副本……与'进入时按 manifest 物化、退出时把已提交的普通文件回提'的模型完全吻合"——但这被明确标注为**模型推断**，而非直接证据。保留并标 ⚠️ |
| 4.2 | `/proc/mounts` 内容（tmpfs `/`、四个 btrfs 同子卷、tmpfs `/tmp`） | ✅ | FACT-RM；FACT-EX 环境 |
| 4.3 | "根 (`/`) 是 tmpfs，每次调用重新创建" | ⚠️ | /proc/mounts 显示 `/` 是 tmpfs，但"每次调用重新创建"是由符号链接消失+mtime 重置+工作副本模型推出的。作为设计推理合理，标 ⚠️ |
| 4.4 | 4 个 `/agent/*` 与 `/workspace/chat` 同指同一 btrfs 子卷 | ✅ | FACT-EX 横切观察；FACT-RM |
| 4.5 | `/tmp` 独立 tmpfs，每次调用都是空 | ✅ | FACT-EX-2 `/tmp` 生命周期实验 |
| 4.6 | 用户/组：uid=995 gid=993 | ✅ | FACT-EX 环境 |
| 4.7 | shell ↔ execute_js 路径映射表 | ✅ | FACT-RM 映射表 |
| 4.8 | "**强烈建议保留默认** … 探测中见过 `working_directory=/agent/current/generated` 导致 `No such file or directory` 后**整次调用的写入全丢**" | ✅ | FACT-RM 写道同样的观察："`working_directory` 指向不存在的路径会让整个调用 exit_code != 0 并丢弃所有写入" |
| 4.9 | "观察到一次 exit_code=0 但某个写入没进入 `files_written` 也没持久化（`_sh_run_marker.txt`）；原因未稳定复现" | ⚠️ | FACT-RM 有此观察。但这是**一次**孤例，未纳入 experiments 探针矩阵。允许保留作警示并标 ⚠️ |
| 4.10 | 可用二进制列表（bash/coreutils/jq/qsv/pdftotext/pandoc/typst/tesseract/docx2txt/imagemagick 等）+ 不可用（网络、包管理器） | ✅ | FACT-EX 附录；工具描述 |
| 4.11 | 典型最佳实践（pdftotext、file→shell、写后回读、/tmp scratch） | ✅ | 直接来自工具描述与 experiments 建议 |
| 4.12 | "100 MB shell 输出回提上限" 与"超过即静默丢弃" **未在本文明确出现，但属于 shell 的致命事实** | ❌ | FACT-EX-5 P16 明确给出：stderr 横幅"total output size (104889902 bytes) exceeds limit (104857600 bytes)"——本文档当前版本没有提到该上限是一处**重要遗漏**，需补进"工作目录与持久性"节 |

**结论**：04 需补充 4.12（致命事实缺失），标注 4.1/4.3/4.9 为 ⚠️。

---

## 5. docs/05-conflicts-and-revisions.md — 命题表

| # | 命题 | 分类 | 事实锚点 / 说明 |
|---|---|---|---|
| 5.1 | 持久性矩阵表（ejs 100%、file 100%、shell 多数持久+1 例异常、shell EROFS、/tmp 短暂、ejs 其它路径 EACCES） | ✅ | FACT-RM；FACT-EX-2/P16 |
| 5.2 | "同一相对路径两个工具依次写 → 后写赢" 且举例 A 被 B 覆盖 | ✅ | FACT-EX-3 P09 正向冲突；P10 反向冲突 |
| 5.3 | "**并发写同一路径** —— 理论上不会发生：同一时刻只有一个工具在执行（单线程 agent）" | ⚠️ | experiments 未直接证伪"两个工具真正同时在跑"——作者基于 agent 调度模型推断。保留并标 ⚠️ |
| 5.4 | "`parallel_for` 发起 6 个 `github_put_repo_file` 的现象……不是文件系统冲突，是远端 GitHub Contents API 的 tree-SHA 竞态" | ⚠️ | 这是**本仓库另一次 build 的经验**，与 vfs 无关。experiments 未涉及该实验。作为背景材料保留并标 ⚠️ |
| 5.5 | 自检清单（写入后回读；命名空间化路径；关键写入走 execute_js；确认 shell working_directory 存在；/tmp 不跨调用） | ✅ | 全部可由 FACT-RM+FACT-EX 的观察推出 |
| 5.6 | "零版本元数据；可达文件系统的任何位置都没有 `.vfs`、`.manifest`、`agent_files*` 或 `.revision` 文件" **本文当前版本未直接引用该实验结论** | ❌ | FACT-EX-3 P12 是对"revision"这一主题的最强证据，当前 05 文档**未引用**。应补入以增强"无 revision 机制"的结论 |
| 5.7 | "每次 shell 提交都会抹除非普通文件元数据（符号链接、+x、硬链接、mtime）" | ❌ | FACT-EX-4 P13-P14 的核心结论**未出现在 05 文档**。这是持久性/冲突相关的关键事实，属遗漏（按用户规则，"矛盾或遗漏事实"归 ❌，按事实补写） |
| 5.8 | "shell 100 MB 输出回提上限（Dim 5 P16）" **未出现在 05 文档** | ❌ | 同 4.12。5 节需点名并链接到 04 |

**结论**：05 有一处重要命题（5.2）需深化；5.3/5.4 标 ⚠️；5.6/5.7/5.8 是事实来源里的关键结论被遗漏，按 ❌ 规则补入（不是反事实，而是漏写 —— 按规则"直接删除原文替换为正确描述"不适用，改为"按事实来源补入并在相应命题处以 ❌→✅ 标注"）。

---

## 6. 深度化方向（逐文档建议）

1. **docs/01**（用户上传）：VFS inode/dentry 分离、btrfs subvolume ro/rw 语义、POSIX EROFS 错误路径、上传链路的原子 rename 语义。对照实验：shell 写 vs ejs 写、用户上传 vs file 下载的路径差异。
2. **docs/02**（file 工具）：HTTP 代理的设计动因（shell 无网络、ejs 无 curl）、content-addressable 内容哈希、upload 命名空间无关的含义、multipart vs binary upload。对照实验：ejs→file upload vs shell→file upload（P04/P06）。
3. **docs/03**（execute_js + twin:fs）：Node.js fs 契约、同步 API vs 异步 VFS 的 lifting 关系、原子事务（exception-on-abort）的内核类比（transactional overlay / copy-on-write）、Stats 裁剪的原因。对照实验：100MB 单文件 vs 10k 小文件吞吐。
4. **docs/04**（shell）：working-copy 挂载模型、tmpfs 根的含义、同 btrfs 子卷不同挂载点的 mount namespace 表达、uid=995 的非 root 能力（无 CAP_SYS_ADMIN、无 mount）、100MB 回提上限的工程动因。对照实验：working_directory 有效 vs 无效。
5. **docs/05**（持久性/冲突）：last-writer-wins 与 CRDT/OT 的对比、原子性粒度（单次调用 vs 跨调用）、POSIX 文件锁（fcntl/flock）在该沙箱里的可用性、为什么没有 revision/manifest（单租户、短生命周期工作副本）。对照实验：快速 10× 重写序列、跨工具冲突序列。

---

## 7. 深度化兜底规则

- 任何**超出 README + experiments 断言范围**的机制讨论，必须明确写成：
  - "**设计推理**"段（内核/VFS 理论对照）
  - 或 "**扩展实验**"节（给出可复现步骤但声明"未在当前仓库实测过"）
- 不得把理论命题伪装成实测结论；不得新增探针记录。

---

## 8. 翻案声明

本 agent 没有对任何事实-来源的命题表达不同意见。所有 ❌/⚠️ 分类均严格依 README + experiments 的现有证据做出。若上游审计 agent 产出与本分类不同，**以其产出为准**——本文档仅为"上游 AUDIT 不可达"时的兜底。
