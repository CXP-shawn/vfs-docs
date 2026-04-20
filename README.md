# vfs-docs

> VFS 设计文档 — **已基于 shell + execute_js + file 工具真实验证**

本仓库文档经 2026-04-20 实地运行后重写，替换了早期纯理论设计稿。现实与最初设计有重大差异，请以本版本为准。

## 核心结论（一句话）

当前系统**没有一个统一的 VFS**。存在两套相互独立的**沙箱视图**，它们共享同一个底层 btrfs 子卷作为存储后端，但通过**不同的挂载路径** 映射给 `execute_js`、`shell`、`file` 三个工具。写入在一次工具调用结束后被提交/同步回后端，下一次调用时通过映射可见。

## 目录结构（真实挂载）

来源：`/proc/mounts` 于 shell 沙箱内的读取结果。

```
/dev/md125 /agent/downloads  btrfs ro,...,subvolid=259,subvol=/tmp
/dev/md125 /agent/context    btrfs ro,...,subvolid=259,subvol=/tmp
/dev/md125 /agent/generated  btrfs rw,...,subvolid=259,subvol=/tmp
/dev/md125 /workspace/chat   btrfs ro,...,subvolid=259,subvol=/tmp
tmpfs      /tmp              tmpfs rw,...
tmpfs      /                 tmpfs rw,...
```

所有 `/agent/*` 和 `/workspace/chat` 都来自 **同一个 btrfs 子卷** (`subvolid=259, subvol=/tmp`)。差别只有读写标志，以及子目录的语义约定（下载区/上下文/生成区/聊天附件）。

## 两套命名空间

| 工具 | 见到的根 | `downloads` 路径 | `generated` 路径 | 可见的其它 |
|---|---|---|---|---|
| `shell` | `/`, `/agent`, `/workspace`, `/tmp`, `/proc`, `/dev`, `/nix/store` | `/agent/downloads` (ro) | `/agent/generated` (rw) | `/agent/context` (ro), `/workspace/chat` (ro), `/tmp` (tmpfs, **每次调用重置**) |
| `execute_js` (`require('twin:fs')`) | `/agent`, `/workspace` | `/agent/current/downloads` (rw) | `/agent/current/generated` (rw) | 无 `/tmp`、无 `/agent/context`、无 `/agent/generated` |
| `file` 工具 | 不暴露文件系统 | 写入后可由其它工具读到 | —— | 操作路径即 `/agent/(current/)downloads/<filename>` |

**关键：同一份内容有两个合法路径。**

- `shell`  ── `/agent/downloads/xxx` ↔ `execute_js` ── `/agent/current/downloads/xxx`
- `shell`  ── `/agent/generated/xxx` ↔ `execute_js` ── `/agent/current/generated/xxx`

跨工具访问必须做 **路径转换**。

## file 工具的消费方式

实测：`file.download_via_curl` 将文件写入 `downloads` 命名空间。

- 下一次 `execute_js` 调用在 `/agent/current/downloads/<filename>` 可读到。
- 下一次 `shell` 调用在 `/agent/downloads/<filename>` 可读到（只读）。

`file` 工具本身不暴露真实路径给调用方；它返回 `filename`、`bytes_written`、`content_hash`（sha256），但不返回绝对路径。调用方需要记住它写到的是 `downloads` 命名空间。

对应的上传 (`upload_via_curl`) 以 `filename` 为输入，也从 `downloads` 命名空间读。

## 权限矩阵（实测）

| 路径 | shell 读 | shell 写 | execute_js 读 | execute_js 写 |
|---|---|---|---|---|
| `/agent/downloads` | ✅ | ❌ EROFS | N/A | N/A |
| `/agent/generated` | ✅ | ✅ | N/A | N/A |
| `/agent/context` | ✅ (空) | ❌ EROFS | N/A | N/A |
| `/agent/current/downloads` | N/A | N/A | ✅ | ✅ |
| `/agent/current/generated` | N/A | N/A | ✅ | ✅ |
| `/tmp` | ✅ | ✅ (但仅本次调用存活) | ❌ EACCES | ❌ EACCES |
| `/workspace/chat` | ✅ (空) | ❌ EROFS | N/A | N/A |

"N/A" 表示该路径对该工具不可见。

## 持久性与跨调用一致性

- `shell` 的每次调用是 **独立进程沙箱**：`/tmp` 清空，`/` 根目录其它非持久路径重置。
- `/agent/generated` 下的写入会被提交回共享后端。实测大多数写入成功持久化；但观察到一次 `_sh_run_marker.txt` 写入后未被保留（exit_code=0，但下一次调用不可见）。尚未找到稳定复现条件，建议：**重要结果始终 `cat` 回一次做自检**，或直接用 `execute_js` 的 `twin:fs` 写（100% 成功持久）。
- `execute_js` 对 `/agent/current/{downloads,generated}` 的写在该调用结束后立即对后端可见；后续的 `shell` 或 `execute_js` 均能看到。

## 哪些目录是"工作输入"、哪些是"工作输出"？（实测语义）

- `downloads` — **工作输入**：用户上传 + `file` 工具下载的产物。例：本 agent 的 `test_readme.md`（用户上传，2279 字节）、`doubao_avatar.png`（用户上传，26799 字节 PNG）、`_file_probe_test.txt`（`file` 下载得到）。`shell` 只能读、不能写；`execute_js` 可读可写。
- `generated` — **工作输出**：当前 agent 产生的文件。两种工具都可以写（通过各自路径前缀），最终合并到同一后端。
- `context` — 实验中为空。文档推测它是历史上下文只读镜像；本次运行未见任何内容，暂无法断言用途。
- `workspace/chat` — 实验中为空。推测是用户聊天附件；本次运行无附件。

## 陷阱清单

1. **路径前缀**：永远不要把 shell 的 `/agent/generated/foo` 字符串传给 `execute_js`（会 404），也不要反过来。
2. **read-only**：`shell` 往 `/agent/downloads` 写会立刻失败；`execute_js` 往 `/tmp`、`/agent/generated` 写也会直接 EACCES。
3. **/tmp 在 execute_js 里不存在**；`/tmp` 在 shell 里每次重置。
4. **shell 里 `cd` 出 `/agent/generated` 不丢失持久性**，但 `working_directory` 指向不存在的路径会让整个调用 exit_code != 0 并丢弃所有写入（观察到 `working_directory=/agent/current/generated` 时 shell 报 "No such file or directory"，那次的所有写都没提交）。
5. **file 工具返回值不含绝对路径**，你得自己记住它写到了 `downloads` 命名空间。

## 文件一览

- [01 - 用户上传与工作区](./docs/01-workspace-uploads.md)
- [02 - file 工具](./docs/02-file-tool.md)
- [03 - execute_js + twin:fs](./docs/03-execute-js-fs.md)
- [04 - shell 工具](./docs/04-shell-tool.md)
- [05 - 持久性与跨工具一致性](./docs/05-conflicts-and-revisions.md)

---

_本版本基于 2026-04-20 在生产沙箱内运行的探测结果编写。所有"实测"断言均由 `shell` / `execute_js` / `file` 的真实调用支持；下游所有结论都可据此复现。_
