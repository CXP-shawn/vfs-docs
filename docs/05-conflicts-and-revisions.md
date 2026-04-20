# 05 — 持久性、冲突与"revision"（实测）

> 范围：一次调用写下去的东西，下一次调用能不能看到？两个工具同时写同一个文件会怎样？

## 持久性真相

| 工具 | 写路径 | 是否持久到共享后端 |
|---|---|---|
| `execute_js` → `/agent/current/{downloads,generated}` | 100%（实测） |
| `file.download_via_curl` → `downloads` | 100%（实测） |
| `shell` → `/agent/generated` (exit_code=0) | **多数持久**，观察到 1 例异常 |
| `shell` → `/agent/generated` (exit_code≠0) | 丢失，整次调用的写都不提交（实测 `working_directory` 无效时发生） |
| `shell` → `/agent/downloads` | ❌ EROFS，根本写不下去 |
| `shell` → `/tmp` | 只活本次调用，退出即丢 |
| `execute_js` → 其它路径 | ❌ EACCES |

## 冲突场景

**同一个相对路径，两个工具依次写**：
- A 在 `execute_js` 里 `writeFileSync('/agent/current/generated/foo.txt', 'A')`。
- B 在随后的 `shell` 里 `echo "B" > /agent/generated/foo.txt`。
- 结果：后写的 "B" 覆盖 "A"。再读任何一侧都是 "B"。

**并发写同一路径**——理论上不会发生：同一时刻只有一个工具在执行（单线程 agent）。但 `parallel_for` 发起 6 个 `github_put_repo_file` 的现象（我们今天刚吃过的亏）**不是文件系统冲突，是远端 GitHub Contents API 的 tree-SHA 竞态**。本地文件系统没有这种问题。

## 为什么今天 GitHub 推送会 409

推一个文件到 `/repos/{owner}/{repo}/contents/{path}` 时，GitHub 用"parent commit tree SHA"做乐观并发。6 个并行 PUT 都基于同一个 parent，第一个成功后，剩下 5 个的 parent 就过期了——返回 `is at <new> but expected <old>`。

解决办法：**要么串行推**，要么**每次 PUT 前 GET 一次拿当前 SHA**。

这跟本地 VFS **没有关系**，但把它写在这里，是因为"VFS 设计 doc"上一版把二者混为一谈。

## 修复建议（适用于未来想扩展成真正 VFS 的场景）

上一版 doc 写了 "manifest + blob store + revision 号" 的设计。**目前的系统里没有这些**。如果要加：

1. **revision 表**：在 `agent_files` 或类似表里给每个 `(namespace, path)` 加一列 `revision`。每次写 +1。
2. **shell 的提交阶段记录读基线**：进入 session 时把读过的文件的 revision 记下；提交阶段若 revision 变了，就报冲突（而不是盲目覆盖）。
3. **blob 去重**：`blobs/sha256/<hash>` + `agent_files` 只存引用，省空间并天然支持"同内容去重"。

以上是 **未实现** 的扩展方向，不是现状。现状就是直写覆盖、没有版本号。

## 自检清单（建议每个写流程都做一遍）

- [ ] 写入后 **立刻** 读回并比对 size / 前 200 字节。
- [ ] 需要跨工具时用**命名空间化路径**（不要在两个工具间传 `/agent/generated/...` 这种 shell-only 路径）。
- [ ] 关键写入走 `execute_js` `twin:fs`（持久性最可靠）。
- [ ] shell 跑长任务前先确认 `working_directory` 存在，否则整批结果全丢。
- [ ] `/tmp` 只当 scratch，不要指望它跨调用存活。
