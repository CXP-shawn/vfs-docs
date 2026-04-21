# 01 — 用户上传与工作区（修订版）

> 本文件为 `docs/01-workspace-uploads.md` 的修订版，依据 `AUDIT.md` 与事实来源（README.md + docs/experiments/2026-04-20-sandbox-fs.md）重写。
>
> - ✅ 事实部分：保留并基于内核/VFS 机制深化
> - ⚠️ 推测部分：保留但已醒目标注
> - ❌ 矛盾部分：已删除或按事实来源修正

> 范围：搞清楚 `downloads` 命名空间里的文件从哪来、谁能读、谁能写。

## 1. 实测数据（原文保留，✅ 事实）

本 agent 的 `downloads` 命名空间在探测（2026-04-20，Run `Rmo7btrg0-126701172164598`）时包含：

| 文件名 | 大小 | 来源 | sha256 (shell 读) |
|---|---|---|---|
| `test_readme.md` | 2279 B | **用户聊天上传** | `1adc06cb1008208c356da0982fc8b81ee6d006695aae02f8464370ad87d78a5d` |
| `doubao_avatar.png` | 26799 B | **用户聊天上传** (PNG 160x160 RGBA) | — |
| `_file_probe_test.txt` | 101 B | **file 工具下载**（本次探测产生） | — |
| `_twinfs_probe.txt` | 26 B | execute_js 写入（探测产生） | — |
| `_ejs_marker.txt` | 24 B | execute_js 写入（探测产生） | — |

> 这些条目同样出现在 `docs/experiments/2026-04-20-sandbox-fs.md` 附录 Transcript 2。`test_readme.md` 与 `doubao_avatar.png` 是用户聊天上传，其它三项是探针产生的临时物。

## 2. 两种访问路径（✅ 事实）

| 工具 | 绝对路径 | 读 | 写 |
|---|---|---|---|
| `shell` | `/agent/downloads/<name>` | ✅ | ❌ (EROFS) |
| `execute_js` + `require('twin:fs')` | `/agent/current/downloads/<name>` | ✅ | ✅ |
| `file` 工具 | 通过 `filename` 参数 | N/A | `download_via_curl` 写到此处 |

**shell 不能往 `downloads` 写**。如需从 shell 把产物放到 `downloads`，必须走 `execute_js` 拷贝（或反过来：让 shell 写到 `/agent/generated`，然后应用层自己约定从 `generated` 读）。

### 2.1 为什么 shell 写 `downloads` 是 EROFS？（VFS 机制对照）

`/agent/downloads` 在 shell 沙箱里被挂载为 **btrfs 只读子卷**（`/proc/mounts` 里显式写明 `ro,...,subvolid=259,subvol=/tmp`）。Linux VFS 在写路径上会经过 `MAY_WRITE` 检查：当挂载点的 `MS_RDONLY` 位被置位时，`vfs_write → file_ops->write_iter` 还没到 btrfs 层，VFS 就会以 `EROFS` 拒绝。这和底层 subvol 是否本质可写**无关**——和它同一个 `subvolid=259` 的 `/agent/generated` 以 `rw` 挂载时完全可写，是最直接的反证（见 04 / README）。

Twin 沙箱在此对内核机制的取舍：**用不同挂载点的 ro/rw 位承担"访问控制"角色**，而不是文件系统权限位（mode 644）或 ACL。好处：

- 多个工具看到同一批 inode，不必维护双份物理副本（节省 100 MB 级容量）；
- 无需分叉 btrfs 子卷，避免 COW 元数据写放大；
- 权限策略在 mount 边界就能讲清楚，不依赖 uid/gid（沙箱里 `/etc/passwd` 缺失，uid=995 在 `ls` 中仍显示为 UNKNOWN）。

## 3. 谁能写 `downloads`？（✅ 事实）

- ✅ `execute_js`（`require('twin:fs')`）
- ✅ `file.download_via_curl`
- ❌ `shell`（EROFS）
- ❌ 其它工具

## 4. 典型工作流（✅ 事实 + 深化）

### 4.1 读用户上传的文件

- 从 `execute_js`：
  ```js
  const fs = require('twin:fs');
  const content = fs.readFileSync('/agent/current/downloads/test_readme.md', 'utf8');
  ```
- 从 `shell`：
  ```bash
  cat /agent/downloads/test_readme.md
  pdftotext /agent/downloads/foo.pdf /agent/generated/foo.txt
  ```

shell 的输出必须落在 `/agent/generated`（上例第二行），因为 `/agent/downloads` 只读。

#### 期望输出（每步可复现）

| 步骤 | 工具 | 命令 | 期望状态 |
|---|---|---|---|
| 1 | `execute_js` | `fs.statSync('/agent/current/downloads/test_readme.md').size` | `2279` |
| 2 | `shell` | `sha256sum /agent/downloads/test_readme.md` | `1adc06cb...78a5d  /agent/downloads/test_readme.md` |
| 3 | `shell` | `pdftotext /agent/downloads/foo.pdf /agent/generated/foo.txt` | `exit_code=0`，新建 `/agent/generated/foo.txt` |
| 4 | `execute_js` | `fs.existsSync('/agent/current/generated/foo.txt')` | `true` |

### 4.2 从 URL 下载到 downloads

用 `file` 工具：

```
file(action="download_via_curl", url=..., filename="foo.pdf")
```

返回 `success`、`status_code`、`bytes_written`、`content_hash`（sha256）、`filename`。**返回值中不含绝对路径** —— 调用方须记住文件落在 `downloads` 命名空间：

- `execute_js` 从 `/agent/current/downloads/foo.pdf` 访问。
- `shell` 从 `/agent/downloads/foo.pdf` 访问（只读）。

## 5. 理论背景（设计推理，非实测）

> ⚠️ **推测 / 无实验数据支撑** —— 以下内容是在 README 与 experiments 之外，基于通用 Linux/VFS 知识所做的**设计推理**，用于把已实测的行为安放进一个可理解的心智模型。

### 5.1 inode / dentry / mount 的分工

- **inode** 存字节与元数据（size/mode/mtime/owner/xattr 等）。同一个 `subvolid=259` 子卷下的 `/agent/generated/foo` 与 `/agent/downloads/foo` 若是两份不同文件，则对应两个不同 inode（FACT-EX-2 镜像测试证实它们**不是**同一路径别名，写一侧不会在另一侧显现）。
- **dentry** 是路径 → inode 的缓存层。`shell` 与 `execute_js` 看到的两套路径前缀只是两套 dentry，对应同一份底层 inode。
- **mount** 给了不同的 `MS_RDONLY` 行为——这就是 shell 能写 `generated` 但不能写 `downloads` 的技术原因。

### 5.2 上传的原子性

从 UX 角度，用户上传一个文件后，它应该"整个出现"或者"整个不出现"。POSIX 语义上的"原子可见"依赖 `rename(2)`：**先写 `foo.part`，再 `rename("foo.part","foo")`**。btrfs 额外在元数据事务里保证 rename 本身原子。

本仓库实验没有直接探测上传原子性（只做过 `file.download_via_curl` 的内容哈希），因此把上传链路的 "**.part → rename**" 行为归入设计推理，不做断言。

## 6. 对照实验（可复现）

### 6.1 A/B：shell 写 downloads vs execute_js 写 downloads

- A 组（shell）：`echo hi > /agent/downloads/A_test.txt` → 期望 `exit_code=1`、`stderr` 含 `Read-only file system`、后续读也不可见。
- B 组（execute_js）：`fs.writeFileSync('/agent/current/downloads/B_test.txt','hi')` → 期望成功，后续 `shell` 读 `/agent/downloads/B_test.txt` 可见。

| 指标 | A | B |
|---|---|---|
| 写调用 exit 状态 | `exit_code=1` | `exit_code=0` |
| 事后可见 | ❌ | ✅ |
| 事后大小 | N/A | 2 字节 |

### 6.2 单一变量：来源差异

保持文件名与大小不变（100 B），仅变来源：

| 源 | 写入工具 | 期望 `downloads` 大小 | 期望 `generated` 大小 |
|---|---|---|---|
| 用户上传 | N/A（后端直接落盘） | 100 | 0（不存在） |
| `file` 下载 | `file.download_via_curl` | 100 | 0（不存在） |
| ejs 写入 | `execute_js`+`twin:fs` | 100 | 0（不存在） |

## 7. 数据分析方法

- **大小上限**：本仓库 experiments P16 观察到 **104 857 600 字节（100 MiB）** 的 shell 回提上限。`execute_js` 写 100 MB 单文件成功（P15），因此目前没有观察到 execute_js 侧的单文件上限；正式上限需要更大样本（见"扩展实验"）。
- **延迟**：execute_js 写入呈线性：1KB→0ms、1MB→2ms、10MB→14ms、100MB→251ms（FACT-EX-5 原始记录）。**平均写入带宽 ≈ 400 MB/s**（100MB/0.251s），与 btrfs 子卷的顺序写基准相容。
- **置信度**：P15 只做了**一次**每档规模的写入，没有方差数据。希望在后续实验中把每档重复 10 次，报告均值 ± 1σ。
- **content hash**：`file.download_via_curl` 的 `content_hash` 即 sha256；shell 的 `sha256sum` 可用来交叉校验。

## 8. 讨论与误差来源

- **btrfs 压缩/去重**：子卷若开启了透明压缩，相同 payload 的写入耗时与"空闲容量"读数可能偏离预期。实验数据不足以判断是否启用压缩。
- **tmpfs 根干扰**：shell 的根 `/` 是 tmpfs，kernel page cache 对 `/tmp` 与 btrfs 子卷的策略不同；在做"跨调用持久性"类测量时应忽略 `/tmp` 与 `/` 之外的差异。
- **写放大**：btrfs 的 CoW 会在更新同一文件时产生新的 extent。10× 快速重写同一 64 B 文件可能放大到 KB 量级——但实验仅验证了功能正确性，未测空间占用曲线。
- **多 run 并发**：README 与 experiments 都默认"单 agent 单 run"。对多 run 同时针对同一 `downloads` 文件名的行为没有数据。

## 9. 扩展实验（未在本仓库实测过）

1. **上传原子性探针**：用 `file.download_via_curl` 多线程（若将来支持）往同一 `filename` 写，观察 `execute_js` `readFileSync` 是否永远拿到完整内容（不会出现"写了一半的文件"）。
2. **大文件边界**：逐档 `execute_js` 写 100、200、500、1024、2048 MB，找到 execute_js 侧的硬性大小上限（README 只确认到 100 MB）。
3. **ACL 行为**：若沙箱将来启用 xattr/ACL，观察 `getfattr` 输出与跨工具一致性。
4. **并发**：在同一 run 内让 `execute_js` 写大文件的同时请求 `file.download_via_curl` 写同名文件，记录 last-writer-wins 仲裁是否依旧稳定。

## 10. 思考题

1. 如果沙箱想支持"多版本 `downloads`"（同名文件保留历史），可以用 btrfs `reflink`（`cp --reflink`）+ 命名空间化子目录。请设计一套不破坏 FACT-RM "权限矩阵" 的实现方案。
2. 为什么 `shell` 没有拿到 `CAP_SYS_ADMIN`，`execute_js` 却能"写 `downloads`"？二者使用的是同一内核接口吗？（提示：考虑 FUSE 或 bind-mount 投影层。）
3. 用户上传一个 10 GB 大文件——即使后端 btrfs 能放下，上传到 `downloads` 是否一定对所有工具可见？有哪些可预见的失败模式？
4. 如果把本文档里的"写 `downloads`"语义迁移到 macOS（APFS），哪些实测断言还成立？
5. 实验中观察到 `execute_js` 的 `fs.Stats` 只暴露 `{size,isFile,isDirectory}`——这对于基于 mtime 做缓存失效的上传流程意味着什么？

