# 05 — 持久性、冲突与 "revision"（修订版）

> 本文件为 `docs/05-conflicts-and-revisions.md` 的修订版，依据 `AUDIT.md` 与事实来源（README.md + docs/experiments/2026-04-20-sandbox-fs.md）重写。
>
> - ✅ 事实部分：保留并基于内核/VFS 机制深化
> - ⚠️ 推测部分：保留但已醒目标注
> - ❌ 矛盾部分：已删除或按事实来源修正
>
> 本次修订补入三项**原本遗漏**的事实来源结论：(a) **实验直接证伪了任何 `.vfs/\.manifest/\.revision/agent_files\*` 文件的存在**（FACT-EX-3 P12）；(b) **非普通文件元数据每次提交都会被抹除**（FACT-EX-4 P13–P14）；(c) **shell 100 MiB 输出回提上限，超限静默丢弃**（FACT-EX-5 P16）。

> 范围：一次调用写下去的东西，下一次调用能不能看到？两个工具同时写同一个文件会怎样？

## 1. 持久性真相（✅ 事实）

| 工具 | 写路径 | 是否持久到共享后端 |
|---|---|---|
| `execute_js` → `/agent/current/{downloads,generated}` | **100%**（实测） |
| `file.download_via_curl` → `downloads` | **100%**（实测） |
| `shell` → `/agent/generated` (`exit_code=0` 且本次总输出 < 100 MiB) | 多数持久 |
| `shell` → `/agent/generated` (`exit_code=0` 但本次总输出 ≥ 100 MiB) | ❌ **整次调用新写入被静默丢弃**（P16） |
| `shell` → `/agent/generated` (`exit_code≠0`) | ❌ 丢失，整次调用的写都不提交（实测 `working_directory` 无效时发生） |
| `shell` → `/agent/downloads` | ❌ EROFS，根本写不下去 |
| `shell` → `/tmp` | ❌ 只活本次调用，退出即丢 |
| `execute_js` → 其它路径 | ❌ EACCES |

## 2. 冲突场景（✅ 事实，last-writer-wins）

### 2.1 同一相对路径，两个工具依次写

- A 在 `execute_js` 里 `writeFileSync('/agent/current/generated/foo.txt', 'A')`。
- B 在随后的 `shell` 里 `echo "B" > /agent/generated/foo.txt`。
- 结果：后写的 "B" 覆盖 "A"。再读任何一侧都是 "B"。

实验 P09 / P10 分别验证正向（ejs 写 A → file 覆盖为 B）、反向（file 写 X → ejs 覆盖为 Y）两种冲突；**两个方向都 last-writer-wins**。所有读取者（shell、ejs、file.upload）看到的都是最终字节。

### 2.2 同一路径连续 10× 重写（P11）

`execute_js` 对 `rapid_P3R-mo7btrg0-...txt` 做 10 次 `writeFileSync`，内容各异（`iter=0..9`）。

- **最终状态**：`iter=9 ts=1776697513776`。
- **期间没有任何中间态泄漏**。与 "每次 `writeFileSync` 整体原子" 的声明一致。

### 2.3 并发写同一路径

> ⚠️ **推测 / 无实验数据支撑** —— 原文："**并发写同一路径** —— 理论上不会发生：同一时刻只有一个工具在执行（单线程 agent）"。experiments 没有直接验证"真正并发"是否被调度阻止；这是基于 agent 调度模型的推断。保留原句并标注。

### 2.4 关于 GitHub `parallel_for` 409 的记录

> ⚠️ **推测 / 无实验数据支撑** —— 原文提到 "`parallel_for` 发起 6 个 `github_put_repo_file` 的现象……不是文件系统冲突，是远端 GitHub Contents API 的 tree-SHA 竞态。" 这是本仓库**另一次 build 的经验**，**与本地 VFS 无关**，也没被 experiments 验证。作为背景保留并标 ⚠️。
>
> 本文档的底线：**本地 VFS 内不存在这种并发冲突**，即使有也是串行化的"后写者赢"。

## 3. 没有 revision / manifest 机制（✅ 事实，FACT-EX-3 P12，原文未引用）

为明确回答"是否持久到共享后端"之外的"历史如何保留"问题，实验直接搜索了可能的版本存储：

```console
$ find / -maxdepth 6 \( -name ".vfs*" -o -name "*manifest*" \
        -o -name "agent_files*" -o -name "*.revision" \) -print 2>/dev/null \
    | grep -vE '(/nix/store|/proc)'
(empty)
```

**可达文件系统里没有 `.vfs`、`.manifest`、`agent_files*` 或 `.revision` 文件**。沙箱就是一个普通挂载的 btrfs 子卷；Twin 不在用户可见层维护版本或修订元数据。这对"revision"一词的正确读法：

- **没有** "上一版自动留存" 的语义；
- **没有** per-file 历史；
- **没有** 自动 conflict marker（没有 `<<<<<<<` 风格合并标记）。

想要版本能力，**必须在 agent 逻辑层**自己做（命名 `foo.v1.txt`、`foo.v2.txt`，或者用 git 仓库）。

## 4. 非普通文件元数据的"revision"行为（✅ 事实，FACT-EX-4，原文未引用）

每次 shell 提交都会抹除下列特征：

| 元数据 / 工件 | 跨调用存活？ | 证据 |
|---|---|---|
| 字节内容 | ✅ | P13/P14 cat 一致 |
| 路径 | ✅ | 同上 |
| 符号链接 | ❌ | `ls`: No such file or directory |
| 可执行位（+x / mode 755） | ❌，回到 644 | `stat -c mode` = 644 |
| 硬链接关系（`nlink>1`） | ❌，两份变独立 inode | `nlink=1` 两文件 |
| mtime / atime / ctime / birth | ❌，重置到提交时刻 | 所有时间戳同步为 commit 时间 |
| owner 名称 | N/A（`/etc/passwd` 缺失） | `stat` 显示 UNKNOWN |

**设计含义**：shell 不是可靠的"元数据持久化"工具。如果你的逻辑依赖 +x 位或 symlink，要么**每次 shell 调用开始时重新 `chmod +x`**，要么把元数据以"字节内容"的方式存下来（例如生成 `.sh` 前先写一行 `#!/usr/bin/env bash` 并在使用前 `chmod +x`）。

`execute_js` 的 `fs.Stats` 只有 `{size,isFile,isDirectory}` —— 从 ejs 读不到 mode/mtime，需要这些字段必须经 shell。

## 5. Shell 100 MiB 回提上限（✅ 事实，FACT-EX-5 P16，原文未引用）

超过 `104 857 600` 字节的"本次调用新产出"会触发：

```
stderr: "Command succeeded but output files could not be saved:
         total output size (104889902 bytes) exceeds limit (104857600 bytes)"
exit_code: 0
files_written: [...未含本次新增文件...]
```

这是 05 最需要留意的**持久性陷阱**：

- `exit_code=0` **不代表**所有写入都持久化了。
- 判定成功的充分条件是 `exit_code==0` **且** `stderr` 不含 "exceeds limit" **且** 目标文件能被下一次调用读到。
- 详见 04 第 7 节。

## 6. 自检清单（✅ 事实，按本次修订扩展）

- [ ] 写入后**立刻**读回并比对 size / 前 200 字节。
- [ ] 需要跨工具时用**命名空间化路径**（不要在两个工具间传 `/agent/generated/...` 这种 shell-only 路径）。
- [ ] 关键写入走 `execute_js` `twin:fs`（持久性最可靠，无 100 MiB 上限约束）。
- [ ] shell 跑长任务前先确认 `working_directory` 存在，否则整批结果全丢。
- [ ] `/tmp` 只当 scratch，不要指望它跨调用存活。
- [ ] **检查 `shell` 响应的 `stderr` 是否含 "exceeds limit" 横幅；即使 `exit_code=0` 也可能本次新写全丢**。
- [ ] 如果依赖 `+x`、符号链接或硬链接关系，**每次 shell 启动都重新构造**它们；不要指望上次留下的元数据。
- [ ] 如果需要"历史版本"，在 agent 层用 `foo.vN` 命名或外部 git 管，不要指望沙箱提供。

## 7. 理论背景（设计推理）

> ⚠️ **推测 / 无实验数据支撑** —— 本节把实测放进内核/VFS 的心智模型。

### 7.1 last-writer-wins 与 CRDT/OT 的对比

- **LWW（last-writer-wins）**：时间戳最新的写法胜出。本地 VFS 按系统调用顺序串行化，最后一次 `write(2)` 之后的 `fsync` 决定最终字节。
- **CRDT / OT**：分布式多副本的合并策略。本沙箱是单副本、单写者顺序，**根本用不上**。
- 代价：业务上"两个 agent 步骤差不多同时想写一份配置"这种逻辑冲突需要**应用层**自己序列化（例如先 `readFileSync` 检查哈希再 `writeFileSync`）。

### 7.2 原子性粒度

- **单次 `writeFileSync` 内部**：内核页级原子；观察到的最小单元是"旧 inode 字节 → 新 inode 字节"的整体替换，不会残留半文件（本仓库快速 10× 重写实验未见中间态）。
- **单次 `execute_js` 调用内部**：工具描述声明"原子"——脚本抛异常所有写入丢弃。**未做 throw-in-middle 对照实验**（⚠️ 推测）。
- **跨调用**：无事务。两次调用的先后完全独立，不存在"回滚前一次调用"的通道。

### 7.3 POSIX 文件锁（fcntl/flock）在本沙箱的可用性

> ⚠️ **推测 / 无实验数据支撑**

沙箱是标准 Linux 6.18，理论上支持 `fcntl(F_SETLK)` 和 `flock(2)`。但本仓库没有针对锁语义的探测，且：

- `execute_js` 的 `twin:fs` 不暴露 `open`/`fcntl`；
- `shell` 能用 `flock`，但 commit 通路只回提普通文件字节，锁本身不会跨调用传递（每次调用都是新的 mount namespace，没有"锁文件"的持久通路，除非锁文件本身被当作普通文件字节持久化）。

**实务上：不要用文件锁做工具间同步**。沙箱的串行化调度已经给了你"同一时刻一个工具在跑"的保证（参 2.3）。

### 7.4 为什么没有 revision？

- **单租户**（单 agent 单 run）。
- **短生命周期**（一次 run 结束后，工作副本可整体回收）。
- **存储后端已是 btrfs**（本身有 CoW 能力，但 Twin 没对用户暴露 snapshot API）。
- 想要版本能力的 agent 会更青睐"推到 git 仓库"这种**显式**路径，而不是"VFS 自动保留"的隐式路径。

## 8. 对照实验（可复现）

### 8.1 A/B 冲突：实验 P09 / P10 的压缩复现

- A：ejs 写 `x.txt = "A"` → `file.download_via_curl` 抓一个 `"B"` 的 URL 到同名 `x.txt` → 读三侧（shell、ejs、file.upload）。**期望**：三侧都返回 `"B"`。
- B：file.download 写 `x.txt = "X"` → ejs 覆盖 `"Y"` → 同样读三侧。**期望**：三侧都返回 `"Y"`。

### 8.2 单变量：`+x` 是否跨调用存活？

```bash
# call 1 (shell)
echo '#!/usr/bin/env bash' > s.sh && echo 'echo hi' >> s.sh && chmod +x s.sh
stat -c "%a" s.sh    # 期望 755

# call 2 (shell)
stat -c "%a" /agent/generated/s.sh   # 期望 644（元数据被抹）
bash /agent/generated/s.sh           # 期望仍能跑（bash 不介意 +x）
./s.sh                               # 期望 Permission denied
```

### 8.3 单变量：shell 100 MiB 横幅

| 写入大小 | 期望 files_written 含目标 | 期望 stderr 有 exceeds-limit |
|---|---|---|
| 95 MiB | ✅ | ❌ |
| 101 MiB | ❌ | ✅ |

## 9. 数据分析方法

- **冲突频次**：对 last-writer-wins 做统计检验需要高频并发写入——本沙箱的串行化调度让该实验无法在 agent 层发起，但可以在单次 `shell` 里开多个背景进程同时写（`&` + `wait`），检验 btrfs 内部的串行化行为（与 Twin 通路无关）。
- **元数据抹除率**：跑 100 次 "shell 写 symlink → 下一次 shell 检查" 的循环，统计 symlink 存活率（预期 0）。这个循环同时可以作为 100 MiB 上限外的一种"回归测试"。
- **revision 搜索**：定期跑一遍 FACT-EX-3 P12 的 `find`，作为"Twin 是否偷偷上线 manifest"的回归探针。

## 10. 讨论与误差来源

- **`exit_code=0` 的可信度**：本文第 1 节就把 "shell + 100 MiB 上限" 列为主要失真源。把 `exit_code` 当成唯一成功信号会错过这类静默丢弃。
- **时序偏移**：两次 shell 调用间的最小间隔可能影响 page cache 命中，但不影响字节一致性。
- **后端黑盒**：Twin 运行时可能在某些版本改 commit 通路（例如换成 btrfs snapshot diff）——这时 FACT-EX-4 的"元数据抹除"可能部分消失。把现有文档当"当前版本的证据截面"。

## 11. 扩展实验（未在本仓库实测过）

1. **文件锁可用性**：在 shell 里 `flock -x foo.lock cmd`，观察是否真的阻塞第二个并发子进程（只能在单次调用内测）。
2. **throw-in-middle 回滚**：执行一段先 `writeFileSync A` 再 `throw Error` 的 `execute_js`，下一次调用检查 `A` 是否存在（对应"整包提交"语义）。
3. **shell 上限精确定位**：从 100 MiB - 100 B 逐字节加，直到恰好命中 exceeds-limit 横幅。
4. **跨工具冲突顺序**：在 `execute_js` 写，紧接着 `file.download_via_curl` 覆盖，紧接着 `shell` 再覆盖——三侧读取最终字节，验证 LWW 串联。
5. **revision 潜伏**：写一份内容、等待 N 秒（`pause_run_until`）、覆写、看"有无任何可见旧版本"（预期 0）。

## 12. 思考题

1. 为什么 Twin 没有对 agent 暴露 btrfs snapshot？从"复杂度 vs 价值"的角度给一个权衡论述。
2. 如果你要为 agent 增加 "每次写入自动存一版" 的能力，在现有 VFS 模型下最小改动是什么？最大改动是什么？
3. `last-writer-wins` 在 **真正并发** 场景下会触发竞态，但本沙箱又保证串行化。如果未来允许两 agent 并发访问同一 `downloads` 目录，你会用哪种锁/序列化机制？为什么不直接让 VFS 做 LWW？
4. `+x` 位在每次提交被重置到 644 —— 这对 "生成可执行脚本作为产物" 的 agent 意味着什么？你会怎么设计"让脚本每次都自动可执行"的模板？
5. 如果 `shell` 的 100 MiB 上限从用户视角是"不透明的数据丢失"，你作为运行时设计者会如何让它从"静默丢弃 + stderr 横幅"变成"明确失败 + 可恢复"？
