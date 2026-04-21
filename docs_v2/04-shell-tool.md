# 04 — `shell` 工具（修订版）

> 本文件为 `docs/04-shell-tool.md` 的修订版，依据 `AUDIT.md` 与事实来源（README.md + docs/experiments/2026-04-20-sandbox-fs.md）重写。
>
> - ✅ 事实部分：保留并基于内核/VFS 机制深化
> - ⚠️ 推测部分：保留但已醒目标注
> - ❌ 矛盾部分：已删除或按事实来源修正
>
> 本次修订还补入一条原本缺失的**致命事实**：`shell` 单次调用的输出总量有 **104 857 600 字节（100 MiB）上限**，超过即静默丢弃。见第 7 节。

> ⚠️ **推测 / 无实验数据支撑** —— 历史摘要："`shell` 用的是**长驻工作目录**模式：先把 session 物化到本地磁盘，跑完命令再同步回来。" 该表述是 experiments 横切观察里明确标注的**模型推断**，不是直接证据。保留并在此处打警示。

| 状态 | 负责人 | 最后更新 |
|---|---|---|
| 修订对齐 README + experiments | builder agent | 2026-04-21 |

## 1. Scope（✅ 事实 + 设计推理）

`shell` 是最接近"真实 Linux"的工具：

- 一个能 `cd` 进去的真实目录（`working_directory` 参数决定）
- 命令之间保持 cwd/环境
- 像本地磁盘一样支持任意 POSIX 操作

> ⚠️ **推测 / 无实验数据支撑** —— "它给你的答案是：**working copy 模式**。" —— 同样属于架构命名；本次实验只证明了"每次提交会重置 inode/mtime/非普通文件元数据"，和 working-copy 模型吻合，但没有任何 experiments 探针独立证实该实现形态。

## 2. 真实沙箱形态（✅ 事实）

从 `/proc/mounts`：

```
tmpfs / tmpfs rw,...,uid=995,gid=993
/dev/md125 /agent/downloads btrfs ro,...,subvolid=259,subvol=/tmp
/dev/md125 /agent/context   btrfs ro,...,subvolid=259,subvol=/tmp
/dev/md125 /agent/generated btrfs rw,...,subvolid=259,subvol=/tmp
/dev/md125 /workspace/chat  btrfs ro,...,subvolid=259,subvol=/tmp
tmpfs /tmp tmpfs rw,...
```

- 根 (`/`) 是 tmpfs（实测挂载类型）。"每次调用重新创建"属推断（见下）。
- 四个 `/agent/*` 和 `/workspace/chat` 都指向同一个 btrfs 子卷，只是读写权限不同。
- `/tmp` 是独立 tmpfs，每次调用都是空的（FACT-EX-2 `/tmp` 生命周期实验直接证实）。
- 用户/组：uid=995 gid=993（不是 root）。`/etc/passwd` 不存在，所以 `id` 打印是 `UNKNOWN`。

> ⚠️ **推测 / 无实验数据支撑** —— "根 (`/`) 是 tmpfs，每次调用重新创建。" 根是 tmpfs 已被 `/proc/mounts` 证实；"每次调用重新创建"是由"符号链接/可执行位/硬链接在下一次 shell 调用前丢失"+`cp` 总拿到新 inode 推出的——兼容 working-copy 模型，但不是直接观察。

## 3. 与其它工具的路径映射（✅ 事实）

| 我在 shell 看到 | execute_js 的对应路径 | 说明 |
|---|---|---|
| `/agent/downloads` | `/agent/current/downloads` | 只读，含用户上传 + file 下载 |
| `/agent/generated` | `/agent/current/generated` | 可读写 |
| `/agent/context` | 不可见 | 只读，实验中为空 |
| `/workspace/chat` | 不可见 | 只读，实验中为空 |
| `/tmp` | 不可见 | tmpfs，仅本次调用 |

## 4. 工作目录与持久性（✅ 事实）

- `working_directory` 默认为 `/agent/generated`。**强烈建议保留默认**或显式设 `/agent/generated` —— 探测中见过 `working_directory=/agent/current/generated` 导致 `No such file or directory` 后**整次调用的写入全丢**。
- 命令退出码为 0 时，`/agent/generated` 下的修改被同步回后端；返回值里的 `files_written` 数组标出哪些文件最终进入持久层。
- `/tmp` 和 `/` 的其余路径永不持久——仅作为本次调用的临时缓存。

> ⚠️ **推测 / 无实验数据支撑** —— README 记录了一次"exit_code=0 但某个写入没进入 `files_written` 也没持久化（`_sh_run_marker.txt`）；原因未稳定复现"。这是**孤例**，未纳入 experiments 探针矩阵，不能作为稳定事实断言。
>
> **可靠写入模式**：结果写完后立刻 `cat` 回读一次，或干脆用 `execute_js` 的 `twin:fs` 做写入（FACT 证实 ejs 100% 持久）。

## 5. 可用的二进制（✅ 事实）

工具描述列出并实测可用：`bash`、`coreutils`、`findutils`、`sed`、`awk`、`grep`、`jq`、`qsv`、`pdftotext`、`pdfinfo`、`unzip`、`zip`、`tar`、`gzip`、`bzip2`、`xz`、`file`、`which`、`diff`、`patch`、`bc`、`make`、`pandoc`、`typst`、`tesseract`、`ocrmypdf`、`imagemagick`、`docx2txt`。

**不可用**：

- 网络（无 curl 到外网 —— 验证 `file` 工具是跨沙箱 HTTP 代理的原因之一）
- 包管理器（无 apt/nix-env 可用）

## 6. 典型最佳实践（✅ 事实 + 深化）

### 6.1 读用户上传 → 写处理结果

```bash
pdftotext /agent/downloads/report.pdf /agent/generated/report.txt
```

| 步骤 | 期望 |
|---|---|
| exit_code | 0 |
| 生成文件 | `/agent/generated/report.txt` 存在，size > 0 |
| 之后 ejs 读 | `fs.readFileSync('/agent/current/generated/report.txt','utf8')` 拿到 UTF-8 文本 |

### 6.2 需要网络：用 `file` 工具把文件落到 `downloads`，再让 `shell` 处理

### 6.3 关键产物写完立刻自检

```bash
echo "$DATA" > /agent/generated/out.json && cat /agent/generated/out.json | head -c 200
```

自检可以直接捕获那种"`exit_code=0` 但某文件未落到 `files_written`"的罕见异常。

### 6.4 `/tmp` 可以用作本次调用的 scratch，用完即弃

## 7. 关键补充：**100 MiB 回提上限**（✅ 事实，原文遗漏）

FACT-EX-5 P16 原始记录：

```
(shell call with 100 MB ejs-side file still present plus all other outputs)
stderr: "Command succeeded but output files could not be saved:
         total output size (104889902 bytes) exceeds limit (104857600 bytes)"
stdout: ...normal output...
exit_code: 0
files_written: [...pre-existing files only, no new ones from this call...]
```

### 7.1 含义

- **单次 `shell` 调用**在 `/agent/generated` 下**新产出的总字节**如果超过 `104 857 600`（= 100 MiB），**整次调用**的新写入都会被**静默丢弃**。
- `exit_code` 仍可能是 `0` —— 用 `exit_code` 判断成功**不够**，必须检查 `stderr` 是否出现 "total output size ... exceeds limit"，或检查目标文件 `size` 是否符合预期。
- **预先存在**（上次调用已经持久化）的文件不会因此丢失；丢的只是"本次调用新写的"。

### 7.2 规避策略（设计推理）

> ⚠️ **推测 / 无实验数据支撑** —— 规避策略是基于已知上限的推理，没有在 experiments 里做对照。

1. **大产物改走 `execute_js` `twin:fs`**。FACT-EX-5 P15 已证实 `execute_js` 能落 100 MB 单文件（251 ms）与 10k 小文件（35 ms）。
2. **分批输出**：把一次生成 200 MB 拆成两次 `shell` 调用，每次 < 100 MiB。
3. **流式到 stdout**：stdout 有单独的 100 KB 上限（工具描述），对超大文本结果没帮助。
4. **边写边删**：若 shell 只需要"处理大文件的某一部分"，处理完立刻 `rm` 中间产物，可控制总产出。

## 8. 理论背景（设计推理）

> ⚠️ **推测 / 无实验数据支撑** —— 本节把 shell 行为放进 Linux mount namespace / btrfs / tmpfs 的心智模型。

### 8.1 working copy = fresh tmpfs root + btrfs subvol bind-mount

每次 `shell` 调用相当于：

1. 创建全新的 mount namespace。
2. 根文件系统是一张 **tmpfs**（全内存），所以 `/` 下的一切（除绑定挂载）每次都是空的。
3. 把 btrfs 子卷按不同挂载选项绑定到 `/agent/{downloads,context,generated}` 与 `/workspace/chat`。
4. 在根 tmpfs 上创建一张新的 `/tmp`。
5. `cd` 到 `working_directory`（默认 `/agent/generated`）。
6. 执行命令。
7. `exit_code=0` 且 `/agent/generated` 下本次的新增/修改字节 < 100 MiB 时，把变更 commit 回后端；否则 stderr 报一次"exceeds limit"。

### 8.2 为什么符号链接、+x、硬链接会丢？

如果提交通路是"读目标文件字节 + 写回目录"（而不是 btrfs 层 snapshot diff），则：

- `symlink` 会被当作普通 "文件内容 = 目标路径字符串" 的一次性回发——但若对方路径解析/识别时被误当作普通文件，就丢了"链接"属性。
- `+x` (mode 755) 这种 POSIX mode 位如果提交通路按 "所有文件一律 644" 写回，就会被强制回到 644（FACT-EX-4 正是这么观察到的）。
- 硬链接的 `nlink > 1` 需要在提交层显式记录并恢复；按"内容字节 + 路径"两元素提交时，同内容不同路径会变成两个独立 inode。

内核角度上这与 `rsync --no-links`、`cp -r --no-preserve=links,mode` 行为等价。

### 8.3 100 MiB 上限大概率是后端工程约束

- 100 MiB 恰好是 `104 857 600` 字节（`100 * 1024 * 1024`）。
- 如果 commit 通路把所有变更编码后经过一个带有消息上限的通道（gRPC / Kafka / 云函数输入大小限制等），100 MiB 就是一个常见的工程红线。
- `execute_js` 的写不受此上限约束，说明 ejs 是走另外一条直连 btrfs 的通道，不走 shell 的 commit 队列。

## 9. 对照实验（可复现）

### 9.1 A/B：`working_directory` 合法 vs 非法

- A 组：`working_directory=/agent/generated`（默认），`echo 1 > A.txt` → 期望 `exit_code=0`，`files_written` 含 `A.txt`。
- B 组：`working_directory=/agent/current/generated`，`echo 1 > B.txt` → 期望 `exit_code≠0`，`stderr` 有 "No such file or directory"，**此次调用的所有写入都丢失**。

### 9.2 单变量：输出体量

固定命令 `dd if=/dev/urandom of=/agent/generated/big.bin bs=1M count=N`：

| `N` | 期望 `exit_code` | 期望 `files_written` 含 `big.bin` | 期望 `stderr` 有 exceeds-limit 横幅 |
|---|---|---|---|
| 10 | 0 | ✅ | ❌ |
| 50 | 0 | ✅ | ❌ |
| 99 | 0 | ✅ | ❌ |
| 101 | 0 | ❌（丢弃） | ✅ |

### 9.3 单变量：非普通文件元数据

- 写 `meta.txt`、`ln -s meta.txt sym.lnk`、`chmod +x meta.txt`、`ln meta.txt hl.txt` 于同一次 shell 调用。
- 下一次 shell 调用：`stat -c "mode=%a nlink=%h" meta.txt hl.txt sym.lnk` → 期望 `mode=644`、`nlink=1`、`sym.lnk` 不存在。

## 10. 数据分析方法

- **延迟分解**：`shell` 调用总延迟 = 容器冷启动 + 命令执行 + commit 回提 + 响应序列化。experiments 只报告了输出大小对提交成功/失败的影响，没有把四段拆开。
- **上限裸验证**：把测试目标文件大小从 95 MB 逐步增到 105 MB（每档 +1 MB），确定断崖点是否精确为 104 857 600 字节。
- **置信区间**：对任何 stochastic 结论（如 "1 次 exit=0 丢失"）至少跑 50 次取频次作为概率估计。

## 11. 讨论与误差来源

- **stdout 100 KB 限制**：与文件回提的 100 MiB 是两套限制，勿混淆。
- **`stderr` 50 KB 限制**：长堆栈可能被截断。
- **时钟抖动**：Twin 运行时的冷启动耗时占比若不稳定，单次计时无参考价值。
- **`/dev/urandom` 随机写**：btrfs 不压缩，大小准确；若换 `/dev/zero`，btrfs 可能启用透明压缩（若开启）使磁盘大小 << 逻辑大小。
- **`/tmp` 容量 567 GB**（experiments 报）—— 理论上 shell 可在 tmpfs 临时堆超大文件，但**提交时仍受 100 MiB**约束。

## 12. 扩展实验（未在本仓库实测过）

1. **精确上限**：95/99/100/101/104/105 MB 逐点测，画出断崖曲线。
2. **多文件分布**：分别用 1×100 MB、100×1 MB、10000×10 KB 三档，看 100 MiB 上限是按总 bytes 还是总 block 算。
3. **stderr 横幅的稳定性**：尝试 101 MB、150 MB、500 MB，验证 stderr 是否始终有 "exceeds limit" 文案。
4. **commit 回滚范围**：超限时是"本次全丢"还是"按文件粒度部分保留"？
5. **并发 shell 调用**：若运行时允许并发（目前看是单 agent 顺序执行），观察 commit 队列是否按 FIFO 处理。

## 13. 思考题

1. 为什么 `/tmp` 是独立 tmpfs 而 `/` 也是 tmpfs？二者作用分别是什么，能合并吗？
2. 如果 Twin 运行时把 `shell` 的回提通路改成 btrfs snapshot diff，有哪些现在的"怪异行为"会消失（例如 +x 回退到 644、硬链接断裂）？新的代价是什么？
3. 用户希望 `shell` 能从 `/agent/downloads` 写回——在不破坏现有 FACT 的前提下，你会怎么设计一个"write-through"子目录？
4. uid=995 gid=993 这种非 root 的 anonymous 身份，对接了宿主哪些安全边界？如果你想在容器内做 `mount -t tmpfs ... /foo` 能成功吗？为什么？
5. 100 MiB 这个阈值的工程含义：如果你负责把上限翻到 1 GiB，需要改动哪些层（运行时、存储、协议层）？
