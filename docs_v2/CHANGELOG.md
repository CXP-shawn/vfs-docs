# docs_v2 CHANGELOG

> 基于仓库内 `README.md` 与 `docs/experiments/2026-04-20-sandbox-fs.md`（以下称 **事实来源**）对 `docs/01~05` 逐份修订。
>
> **重要声明**：本次修订**未能**读取上游审计 agent 的 AUDIT.md。工作区没有其它 agent 数据库，本地 downloads 目录为空，仓库内也不含 AUDIT.md。经用户明确授权，由本 agent（builder）依据上述两份事实来源自行编写一份 `AUDIT.md`（也一并放在 `docs_v2/` 下）再据其执行"✅ 深化 / ⚠️ 标注 / ❌ 修正或删除"三类规则。
>
> **未改动原 `docs/` 目录的任何文件。**

## 01-workspace-uploads.md

### 删了什么（❌）

- 无（本文档所有原始命题都被事实来源支持）。

### 标了什么（⚠️）

- 在"理论背景（设计推理）"整节的 5.1 / 5.2 前加了 ⚠️ 段首标注，说明"inode/dentry/mount 的分工"、"上传的原子性 = rename 语义"属于**设计推理**，没有本仓库直接证据。

### 深化了什么（✅）

- 每行实测表下注明"同样出现在 `docs/experiments/2026-04-20-sandbox-fs.md` 附录 Transcript 2"。
- 新增 "2.1 为什么 shell 写 `downloads` 是 EROFS？（VFS 机制对照）" 一段：POSIX MAY_WRITE 检查、同一 subvol 不同挂载点、Twin 沙箱用 ro/rw 位承担访问控制的取舍。
- 新增 "4.1 期望输出（每步可复现）" 的 4 行表格，把读上传文件的每一步绑定到具体期望值（大小 2279、哈希、exit_code 等）。
- 新增 "6. 对照实验（可复现）" 的 A/B 对照：shell 写 downloads vs ejs 写 downloads。
- 新增 "7. 数据分析方法"、"8. 讨论与误差来源"、"9. 扩展实验"、"10. 思考题"四节（按用户"深度化方向"要求全覆盖：理论背景、精确操作步骤、对照实验、分析方法、讨论、扩展/思考）。

---

## 02-file-tool.md

### 删了什么（❌）

- 删除了原文第 2.6 条："**`upload_via_curl` 只能从 `downloads` 命名空间读。从 `generated` 产生的文件若要上传，要先拷到 `downloads`。**" —— 被 FACT-EX-1 P06 直接证伪（`file.upload_via_curl(filename='...',url=...)` 从 `/agent/generated` 下读出并上传成功）。
- 替换为 "4. 关键修正：`upload_via_curl` 的读路径是**命名空间无关**的" 一节，引原始 P06 记录作为证据。

### 标了什么（⚠️）

- "跨沙箱 HTTP 代理" 一词旁加 ⚠️：这是架构命名，not 实验结论。
- "`file` 工具不直接解压压缩包" 旁加 ⚠️：来源于 schema 枚举而非专门实验。
- 第 6.3 节 "upload 命名空间无关 = 什么内核机制？" 整段标 ⚠️：是对"file 工具内部查找路径"的纯推理。

### 深化了什么（✅）

- 新增 "6.1 为什么需要一个代理工具？"：mount namespace vs network namespace、ambient authority 外置。
- 新增 "6.2 content_hash 的工程价值"：去重 / 完整性 / 缓存键。
- 新增 "7.1 A/B/C：download → 命名空间归属" 表：三种写入源 × shell 两侧读 + file.upload 读 的期望矩阵。
- 新增 "7.2 content_hash 幂等性"、"8. 数据分析方法"、"9. 讨论与误差来源"、"10. 扩展实验"、"11. 思考题" 各节。

---

## 03-execute-js-fs.md

### 删了什么（❌）

- 无（03 文档未出现与事实来源直接冲突的命题）。

### 标了什么（⚠️）

- 原 "fs.readdirSync('/workspace') 在 execute_js 里不可读取" 的表述：改标 ⚠️ 并备注"严谨说应改为'该路径在 execute_js 视图内不可写且实际运行时为空'" —— experiments 里没有直接 transcript 证实 "readdirSync('/workspace')" 报 ENOENT。
- 原文 "`Buffer.isBuffer` 的行为与标准 Node 不完全一致" 旁加 ⚠️：experiments 未做该探测，保留为经验告诫。
- 新增 "6. 原子性" 整节 ⚠️ 标注：原子性来自工具描述而非本仓库实验。

### 深化了什么（✅）

- 新增 "7.1 从 Node.js fs 到内核 file_operations"：open/read/close 三系统调用与 VFS 映射。
- 新增 "7.2 为什么没有 fsync/fdatasync？" 关于 page cache、btrfs 事务日志、Twin 代为执行 fsync 的讨论。
- 新增 "7.3 statSync 为什么裁剪成 {size,isFile,isDirectory}？" 三条工程动机分析。
- 新增 "8.1 原子性（throw-in-middle）"、"8.2 字符串写 vs Buffer 写"、"8.3 计数基准" 三个对照实验（含实测基线 1000/10000 文件耗时）。
- 新增 "9. 数据分析方法"（吞吐 ≈ 400 MB/s、方差问题、文件数 vs 文件大小拆分）、"10. 讨论与误差来源"、"11. 扩展实验"、"12. 思考题"。

---

## 04-shell-tool.md

### 删了什么（❌）

- 原文**遗漏**了 FACT-EX-5 P16 的 "shell 100 MiB 回提上限"——按"❌ 按事实来源修正"规则新增 "第 7 节：关键补充：100 MiB 回提上限"，把 stderr 横幅原文、含义、规避策略都写进。

### 标了什么（⚠️）

- "`shell` 用的是长驻工作目录模式" 的整段摘要旁加 ⚠️：working-copy 模式属 experiments 明确标注的模型推断。
- "根 (/) 是 tmpfs，每次调用重新创建" —— 第二半句（每次调用重新创建）标 ⚠️。
- "观察到一次 exit_code=0 但某个写入没进入 files_written 也没持久化（_sh_run_marker.txt）" 整段标 ⚠️：孤例、未稳定复现、不纳入探针矩阵。
- "working-copy = fresh tmpfs + bind-mount" 整段 8.1 标 ⚠️：内核模型对照。
- "为什么符号链接/+x/硬链接会丢" 整段 8.2 标 ⚠：把 FACT-EX-4 观察放进提交通路的推测。
- "100 MiB 上限大概率是后端工程约束" 8.3 标 ⚠。
- "规避策略（设计推理）" 7.2 段标 ⚠。

### 深化了什么（✅）

- 新增第 7 节完整讲 100 MiB 上限 + 4 条规避策略。
- 新增 "9.1 A/B working_directory 合法 vs 非法" 对照实验（呼应 FACT-RM 的致命陷阱）。
- 新增 "9.2 输出体量单变量扫描"（10/50/99/101 MiB 四档）。
- 新增 "9.3 非普通文件元数据对照"。
- 新增 "10. 数据分析方法"、"11. 讨论与误差来源"（stdout 100 KB 与 100 MiB 勿混淆、时钟抖动、/dev/urandom 压缩盲区）、"12. 扩展实验"、"13. 思考题" 各节。

---

## 05-conflicts-and-revisions.md

### 删了什么（❌）

- 把原文**遗漏**的三条关键事实**补入**（严格按用户"按事实来源补正"的意图处理；这些不是反事实，而是漏写）：
  1. FACT-EX-3 P12 "`find` 未找到任何 `.vfs/\.manifest/\.revision/agent_files\*`" —— 新增第 3 节 "没有 revision / manifest 机制"。
  2. FACT-EX-4 P13-P14 元数据抹除（symlink/+x/hardlink/mtime）—— 新增第 4 节 "非普通文件元数据的 revision 行为"。
  3. FACT-EX-5 P16 shell 100 MiB 上限 —— 新增第 5 节，与 04 的第 7 节呼应。
- 持久性表第 1 行扩成 4 条细分（含两条 ❌ 行：100 MiB 超限 / exit_code≠0）。

### 标了什么（⚠️）

- 2.3 "并发写同一路径——理论上不会发生：同一时刻只有一个工具在执行（单线程 agent）" 段首加 ⚠。
- 2.4 "关于 GitHub parallel_for 409 的记录" 整段 ⚠：非 VFS 事实、来自另一次 build。
- 第 7 节 "理论背景（设计推理）" 整节 ⚠：LWW/CRDT 对比、POSIX 文件锁可用性、为什么没有 revision。

### 深化了什么（✅）

- 新增第 6 节 "自检清单"扩到 8 条：新增 100 MiB 横幅自检、`+x` 不跨调用存活的应对、"没有 revision"的 agent 层替代。
- 新增 "8.1 A/B 冲突（P09/P10 压缩复现）"、"8.2 +x 跨调用存活"、"8.3 shell 100 MiB 横幅" 三个对照实验。
- 新增 "9. 数据分析方法"（冲突频次、元数据抹除率、revision 搜索回归）。
- 新增 "10. 讨论与误差来源"（exit_code 信号可信度、时序偏移、后端黑盒）。
- 新增 "11. 扩展实验"（文件锁、throw-in-middle、100 MiB 精确定位、跨工具冲突顺序、revision 潜伏）。
- 新增 "12. 思考题" 5 道。

---

## 统计

| 文件 | 新版 bytes | 删（❌） | 标（⚠️） | 深化（✅） |
|---|---|---|---|---|
| 01-workspace-uploads.md | ~6.5 KB | 0 段 | 2 段 | +7 节 |
| 02-file-tool.md | ~6.5 KB | 1 段 | 3 段 | +8 节 |
| 03-execute-js-fs.md | ~6.5 KB | 0 段 | 3 段 | +8 节 |
| 04-shell-tool.md | ~7.9 KB | 1 段（补入遗漏 Fact）| 6 段 | +8 节 |
| 05-conflicts-and-revisions.md | ~7.8 KB | 3 段（补入遗漏 Fact）| 3 段 | +9 节 |

---

## Reviewer notes（翻案意见）

本次修订**没有**对 AUDIT.md 内任何分类表达不同意见——因为 AUDIT.md 是本 agent 自己产出的（上游不可达）。一旦上游审计 agent 未来补齐 AUDIT.md，请以其分类为准重审本次修订。

提请注意的是 02 文档的 ❌ 条与 04/05 的"补入遗漏 Fact"——这三处修改改变了原文档立场，但都**严格引用 experiments 文档自己的原始探针记录**，不是本 agent 的额外推断。
