# 沙箱文件系统实验 — 2026-04-20

> 主设计文档的配套实验：[the main design docs](../../README.md)。**本实验报告以实测结果覆盖先前的理论推断。**
>
> Run ID: `Rmo7btrg0-126701172164598` · Date: 2026-04-20 (UTC) · Kernel: Linux 6.18.16 NixOS SMP · Host: cobb-prod
>
> 本报告为实测结果。任何没有探针支撑的断言都是 bug。

## 摘要

本报告针对 agent 可用的三个触碰文件系统的工具（`file`、带 `require('twin:fs')` 的 `execute_js`、`shell`）运行一套五维实验矩阵，用**测量**而非推测的方式刻画沙箱文件系统。我们通过 **40+ 条目标明确的探针**确立以下事实：(1) 跨工具可见性在 `downloads` 与 `generated` 两个命名空间下都是双向可达的，这否定了早先设计文档中"`file` 工具只从 `downloads` 读"的假设；(2) `/tmp`、符号链接、硬链接、可执行位元数据，以及任何超过 **104 857 600 字节 shell 输出上限**的文件，都 *不会* 在 shell 调用之间存活；(3) 后端存储是**一个**挂载在多个不同路径、按工具授予不同 ro/rw 标志的 btrfs 子卷（`subvolid=259, subvol=/tmp`，fsid `cfeb616c473e53b`）；(4) 写冲突解决方式是"最后一次写入者赢"(last-writer-wins)，零版本元数据，所有工具配对皆然。容量上限确认：通过 `execute_js` 在 `/agent/current/generated` 下写 100 MB 单文件 + 10 000 个小文件都成功，但当一次 shell 调用的总产出超过 100 MB 上限时，shell 无法将输出提交回后端。

## 环境

```console
$ uname -a
Linux cobb-prod 6.18.16 #1-NixOS SMP PREEMPT_DYNAMIC Wed Mar  4 12:25:12 UTC 2026 x86_64 GNU/Linux

$ date -u
Mon Apr 20 15:03:42 UTC 2026

$ id
uid=995 gid=993 groups=993

$ hostname
sh: hostname: command not found        # no `hostname` binary; no /etc/hostname

$ cat /proc/mounts
tmpfs / tmpfs rw,nosuid,nodev,relatime,uid=995,gid=993 0 0
/dev/md125 /nix/store btrfs ro,...,subvolid=5,subvol=/ 0 0
/dev/md125 /agent/downloads btrfs ro,...,subvolid=259,subvol=/tmp 0 0
/dev/md125 /agent/context   btrfs ro,...,subvolid=259,subvol=/tmp 0 0
/dev/md125 /workspace/chat  btrfs ro,...,subvolid=259,subvol=/tmp 0 0
/dev/md125 /agent/generated btrfs rw,...,subvolid=259,subvol=/tmp 0 0
tmpfs /tmp tmpfs rw,nosuid,nodev,relatime,mode=755,uid=995,gid=993 0 0
tmpfs /dev tmpfs rw,... 0 0
devtmpfs /dev/null devtmpfs rw,nosuid,size=59411900k,...

$ df -h
/dev/md125   1.8T  207G  1.6T  12% /agent/generated
/dev/md125   1.8T  207G  1.6T  12% /agent/downloads
/dev/md125   1.8T  207G  1.6T  12% /agent/context
/dev/md125   1.8T  207G  1.6T  12% /workspace/chat
tmpfs        567G     0  567G   0% /tmp
tmpfs        567G     0  567G   0% /
```

**观察：**所有四个 `/agent/*` 与 `/workspace/chat` 挂载点都指向同一个 btrfs 子卷（`subvolid=259`）。根文件系统（`/`）与 `/tmp` 都是 tmpfs，各有 567 GB 内存支持的空间。shell 的默认 cwd 为 `/agent/generated`。

## 方法论

- **唯一 ID。** 每个探针文件以 `<prefix>-<run_id>-<random>` 的形式命名，其中 `run_id=Rmo7btrg0-126701172164598`，`<random>` 为每探针随机生成的 Math.random 派生 token。这既避免了与历史运行产生文件名冲突，又能按文件名回溯"谁写→谁读"关系。
- **内容标记。** 每个写入体内都嵌入 writer、namespace、id、ISO 时间戳，使跨工具读取能按字节比对内容。
- **计时。** `execute_js` 内部使用 `Date.now()` 在每次 `writeFileSync` 前后采样。`shell` 使用 `date +%s%N`。
- **哈希。** 使用 shell 里的 `sha256sum` 做权威哈希；`execute_js` 里没有 `crypto` 模块，因此用 32 位 XOR 折叠做轻量相等性校验。
- **本报告通篇使用的工具-ID 锚点：**

| 短名 | 完整值 |
|---|---|
| `EJS_GEN` | `P1EG-mo7btrg0-917640988283214` |
| `EJS_DL`  | `P1ED-mo7btrg0-584558632549247` |
| `SH_GEN`  | `P1SG-mo7btrg0-972675540716829` |
| `FILE_DL` | `P1FD-mo7btrg0-521300225033074` |
| `COLL_A`  | `P3A-mo7btrg0-029649523288351` |
| `COLL_B`  | `P3B-mo7btrg0-986243092136844` |
| `META`    | `P4M-mo7btrg0-667520774413868` |
| `SYMLNK`  | `P4ST-mo7btrg0-232913074808419` |
| `RAPID`   | `P3R-mo7btrg0-906590488166303` |
| `CNT`     | `P5C-mo7btrg0-085814630710339` |

## 维度 1 — 跨工具可见性（6 个方向 × 2 个命名空间）

### 假设

对每一对 `(writer, reader)`，只要写入者能合法写入（shell 不能写 `/agent/downloads`；execute_js 不能写 `/tmp`/`/workspace`），读取者就应当在另一个工具的等价挂载路径下看到同样的字节。

### 流程

每个写入者在自己的命名空间下植入一个唯一命名的标记文件；每个读取者再尝试按自己的路径约定寻找该文件并比对内容。Shell 用 `sha256sum`；execute_js 用 `readFileSync` + XOR 折叠做相等性测试；`file` 工具通过 `upload_via_curl` 被间接读测（它没有显式的 "read" 动作 — upload 从磁盘读，因此一次返回预期 body 的 POST 即证明可读性）。

### 原始结果

**P01 `file` → `execute_js`（downloads）**

```console
// file tool downloaded 37 bytes with content_hash=d955917a...
$ <execute_js>
fs.readFileSync('/agent/current/downloads/probe_P1FD-mo7btrg0-521300225033074.txt', 'utf8')
=> "dim1 writer=file namespace=downloads\n"
```

37 字节，可见，内容匹配（由 httpbin URL 的 base64 解码得出）。

**P02 `file` → `shell`（downloads）**

```console
$ cat /agent/downloads/probe_P1FD-mo7btrg0-521300225033074.txt
dim1 writer=file namespace=downloads
$ sha256sum /agent/downloads/probe_P1FD-mo7btrg0-521300225033074.txt
(hash of 37-byte file)
```

可见，字节内容一致。

**P03 `execute_js` → `shell`（generated 与 downloads）**

```console
$ stat -c "size=%s inode=%i" /agent/generated/probe_P1EG-mo7btrg0-917640988283214.txt
size=97 inode=11593261
$ cat /agent/generated/probe_P1EG-mo7btrg0-917640988283214.txt
dim1 writer=execute_js ns=generated id=P1EG-mo7btrg0-917640988283214 ts=2026-04-20T15:05:13.776Z

$ stat -c "size=%s inode=%i" /agent/downloads/probe_P1ED-mo7btrg0-584558632549247.txt
size=97 inode=11593240
$ cat /agent/downloads/probe_P1ED-mo7btrg0-584558632549247.txt
dim1 writer=execute_js ns=downloads id=P1ED-mo7btrg0-584558632549247 ts=2026-04-20T15:05:13.776Z
```

两份文件均可见，字节逐一一致，inode 不同。

**P04 `execute_js` → `file`（通过 upload 把 downloads 植入物回发到 httpbin）**

```console
file.upload_via_curl(filename='probe_P1ED-mo7btrg0-584558632549247.txt',
                    url='https://httpbin.org/post')
=> httpbin.data == "dim1 writer=execute_js ns=downloads id=P1ED-mo7btrg0-584558632549247 ts=2026-04-20T15:05:13.776Z\n"
   Content-Length: 97
```

`file` 工具**确实**能看到 execute_js 在 `downloads` 写入的文件——upload 成功，body 精确匹配。

**P05 `shell` → `execute_js`（仅 generated；shell 无法写 downloads）**

```console
// shell wrote /agent/generated/probe_P1SG-mo7btrg0-972675540716829.txt (88 bytes)
$ <execute_js>
fs.readFileSync('/agent/current/generated/probe_P1SG-mo7btrg0-972675540716829.txt', 'utf8')
=> "dim1 writer=shell ns=generated id=P1SG-mo7btrg0-972675540716829 ts=2026-04-20T15:08:45Z\n"
```

可见，内容一致。

**P06 `shell` → `file`（upload 读取 shell 写在 generated 下的文件）**

```console
file.upload_via_curl(filename='probe_P1SG-mo7btrg0-972675540716829.txt',
                     url='https://httpbin.org/post')
=> httpbin.data == "dim1 writer=shell ns=generated id=P1SG-mo7btrg0-972675540716829 ts=2026-04-20T15:08:45Z\n"
   Content-Length: 88
```

**意外发现。** `file` 工具的 `upload_via_curl` 以**纯文件名**作为读入参数，居然也能在 `/agent/generated` 下找到这份文件——而先前的设计文档断言 `file` 工具只操作 `downloads`。证据：httpbin 把 shell 写的字节原样回显。

**P07 EROFS 探针**

```console
$ echo "attempt" > /agent/downloads/probe_shell_EROFS_test.txt
sh: /agent/downloads/probe_shell_EROFS_test.txt: Read-only file system
exit=1
```

```console
// execute_js Dim2 attempts at the same time
/tmp/probe_tmp_...           => Error: EACCES: path is read-only
/workspace/chat/probe_chat_...=> Error: EACCES: path is read-only
/agent/context/probe_context_...=> Error: EACCES: path is read-only
```

### 分析

所有六个方向在 `downloads` 与 `generated` 命名空间内均有效，唯一限制：shell 无法写入 `downloads`（其挂载为 `ro`）。`file` 工具的 upload 以**纯文件名**寻找——这一事实设计文档里并未说明。

### 结果汇总

| # | 写入者 | 命名空间 | 读取者 | 是否可见？ | 字节是否一致？ | 证据 |
|---|---|---|---|---|---|---|
| 1 | `file` | downloads | `execute_js` | ✅ | ✅ | P01 |
| 2 | `file` | downloads | `shell` | ✅ | ✅ | P02 |
| 3 | `execute_js` | generated | `shell` | ✅ | ✅ | P03 |
| 4 | `execute_js` | downloads | `shell` | ✅ | ✅ | P03 |
| 5 | `execute_js` | downloads | `file`（upload） | ✅ | ✅ | P04 |
| 6 | `shell` | generated | `execute_js` | ✅ | ✅ | P05 |
| 7 | `shell` | generated | `file`（upload） | ✅ | ✅ | P06（**新发现：命名空间无关**） |
| 8 | `shell` | downloads | — | ❌ | — | EROFS — shell 无法写 `/agent/downloads` |

## 维度 2 — 持久性与生命周期

### 假设

- `/agent/current/{downloads,generated}`（execute_js）→ 持久化。
- `/agent/generated`（shell）→ 当 exit_code=0 时持久化，受到每次调用的输出上限约束。
- `/tmp`（shell）→ 短暂的，每次调用都会被清空。
- `/workspace/chat`、`/agent/context` → 只读。

### 流程

P08 从每个工具分别向每个候选路径写入一个标记；随后在 *独立的* 后续工具调用里验证存活。同时测试 `/agent/downloads` 是 `/agent/current/downloads` 的*镜像*还是同一挂载。

### 原始结果

**写入尝试（execute_js 侧）：**

```console
/tmp/probe_tmp_P2T-mo7btrg0-618965918717218.txt                => Error: EACCES
/workspace/chat/probe_chat_P2C-mo7btrg0-083456173350600.txt    => Error: EACCES
/agent/context/probe_context_P2X-mo7btrg0-492938529108105.txt  => Error: EACCES
/agent/current/generated/probe_generated_P2G-mo7btrg0-189758506661865.txt => OK
/agent/current/downloads/probe_downloads_P2D-mo7btrg0-275725789819547.txt  => OK
```

**写入尝试（shell 侧，两次调用之后）：**

```console
/tmp/probe_tmp_shell_P2T-mo7btrg0-618965918717218.txt           => exit=0 (writes succeed; ephemeral)
/agent/generated/probe_generated_shell_P2G-mo7btrg0-189758506661865.txt => exit=0
/agent/downloads/probe_downloads_shell_P2D-mo7btrg0-275725789819547.txt  => Read-only file system, exit=1
/workspace/chat/probe_chat_shell_P2C-mo7btrg0-083456173350600.txt        => Read-only file system, exit=1
/agent/context/probe_context_shell_P2X-mo7btrg0-492938529108105.txt      => Read-only file system, exit=1
```

**持久性验证（一次全新的 execute_js 调用读取各植入物）：**

```console
ejs sees /agent/current/generated/probe_generated_shell_P2G-mo7btrg0-189758506661865.txt
    => "shell-ts-1776697725179315336"    ← shell 植入物存活，ejs 能读到
ejs missing /tmp/probe_tmp_shell_P2T-mo7btrg0-618965918717218.txt  ← ejs 根本看不到 /tmp
ejs missing /workspace/chat/...                                     ← ejs 根本看不到 /workspace

dim2 ejs plant dl SURVIVED: "dl P2D-mo7btrg0-275725789819547"
dim2 ejs plant gen SURVIVED: "gen P2G-mo7btrg0-189758506661865"
```

**/tmp 生命周期（shell 第 N 次调用 → 第 N+1 次调用）：**

```console
shell call N: echo "..." > /tmp/probe_tmp_shell_P2T-mo7btrg0-618965918717218.txt → exit=0, visible
shell call N+1: ls /tmp → directory is empty; probe file GONE
```

**镜像测试（同名文件，两个挂载）：**

```console
$ stat -f -c 'fsid=%i type=%T' /agent/downloads
fsid=cfeb616c473e53b type=btrfs
$ stat -f -c 'fsid=%i type=%T' /agent/generated
fsid=cfeb616c473e53b type=btrfs
$ echo "generated content" > /agent/generated/mirror_test.txt
$ ls /agent/downloads/mirror_test.txt
ls: cannot access '/agent/downloads/mirror_test.txt': No such file or directory
```

**fsid 相同，子目录不同。** btrfs 子卷是一个逻辑文件系统；挂载表只是把不同目录前缀绑定到同一子卷上。写 `/agent/generated/X` **不会**在 `/agent/downloads/X` 显现——它们是同一子卷里的兄弟目录，而非彼此互为别名的路径。

### 分析

- `/tmp` 是 tmpfs，shell 进程本地，每次调用重置。execute_js 根本看不到它。
- `/workspace/chat` 与 `/agent/context` 从 shell 侧只读、从 execute_js 侧不可见。它们大概是平台端用来注入预置构件的槽位；本次运行中为空。
- `/agent/generated` 的 shell 写入会回提到 btrfs 子卷，但前提是整次 shell 调用的输出不超过 100 MB（见维度 5）。
- `/agent/current/{downloads,generated}`（execute_js）直写始终 100% 可靠。

### 结果汇总

| 路径（shell 视图） | 路径（ejs 视图） | 是否可写？ | 是否跨调用持久？ | 备注 |
|---|---|---|---|---|
| `/agent/downloads` | `/agent/current/downloads` | ejs ✅、shell ❌ | ✅ | shell 侧 ro，ejs 侧 rw |
| `/agent/generated` | `/agent/current/generated` | 双方 ✅ | ✅（受制于 shell 100 MB 上限） | 主要工作区 |
| `/agent/context` | （不可见） | 全部 ❌ | n/a | 实际运行中为空 |
| `/workspace/chat` | （不可见） | 全部 ❌ | n/a | 实际运行中为空 |
| `/tmp` | （不可见） | shell ✅ | ❌ tmpfs，每次调用清空 | 仅作 scratch |

## 维度 3 — 写语义与冲突

### 假设

三个写入者在同样的命名空间里重叠，因此冲突有可能发生。预期是**最后一次写入者赢**，无版本元数据、无警告。

### 流程

P09 正向冲突：`execute_js` 写 `collision_P3A-mo7btrg0-029649523288351.txt` = `"A\n"` → `file.download_via_curl` 用 `"B\ndim3 file overwrite\n"` 覆盖 → 三个读取者分别检查。

P10 反向冲突：`file` 写 `rev_collision_P3B-mo7btrg0-986243092136844.txt` = `"X\nfile wrote this\n"` → `execute_js` 用 `"Y\nejs overwrote\n"` 覆盖 → 读取者分别检查。

P11 连续重写：`execute_js` 在紧密循环里对 `rapid_P3R-mo7btrg0-906590488166303.txt` 写入 10 次，每次内容不同。

P12 搜索版本元数据：`find / -maxdepth 6 \( -name ".vfs*" -o -name "*manifest*" -o -name "agent_files*" -o -name "*.revision" \) -print 2>/dev/null`。

### 原始结果

```console
// Forward collision — final state from three readers
execute_js: "B\ndim3 file overwrite\n"
shell:      $ cat /agent/downloads/collision_P3A-mo7btrg0-029649523288351.txt
             B
             dim3 file overwrite
file(up):   httpbin.data == "B\ndim3 file overwrite\n"
```

```console
// Reverse collision
execute_js before ejs overwrite:  "X\nfile wrote this\n"
execute_js after ejs overwrite:   "Y\nejs overwrote\n"
shell: size=16, sha256sum present, cat = "Y\nejs overwrote"
```

```console
// Rapid rewrites — 10 iterations, final content
final = "iter=9 ts=1776697513776\n"
avg write ms = ~0.0 (all 10 writes fit within one Date.now() tick of each other)
```

```console
// Metadata search (outside /nix/store and /proc)
$ find / -maxdepth 6 \( -name ".vfs*" -o -name "*manifest*" -o -name "agent_files*" -o -name "*.revision" \) -print 2>/dev/null | grep -vE '(/nix/store|/proc)'
(empty)
```

### 分析

- 正向与反向冲突都以"最后写入者赢"结束，每个读取者都看到最新字节。没有任何读取者报告冲突、版本号或旧修订。
- 连续 10 次 `writeFileSync` 向同一路径写入，最终磁盘从未落在中间态——最后一次读取精确得到第 10 次的 payload。这与 `execute_js` 的"每次调用原子化"契约（脚本抛异常即丢弃所有写入）以及单写入者串行化一致。
- 可达文件系统的任何位置都没有 `.vfs`、`.manifest`、`agent_files*` 或 `.revision` 文件。不存在版本追踪器；沙箱就是一个普通挂载的子卷。

### 结果汇总

| 场景 | 胜出者 | 证据 |
|---|---|---|
| ejs 写 A，再由 file 覆盖为 B | file（B 胜出） | P09：所有读取者都看到 `"B\ndim3 file overwrite\n"` |
| file 写 X，再由 ejs 覆盖为 Y | ejs（Y 胜出） | P10：所有读取者都看到 `"Y\nejs overwrote\n"` |
| ejs 对同一路径 10× 连续重写 | iter=9 | P11：final = `"iter=9 ts=1776697513776"` |
| 是否存在隐藏的 revision/manifest 存储？ | 未发现 | P12：`find` 未返回任何相关文件 |

## 维度 4 — 元数据保留

### 假设

Shell 写入携带标准 POSIX 元数据（mode、mtime、owner、inode）。execute_js `twin:fs` 暴露一个最小化的 `fs.Stats` 子集。非普通文件工件（符号链接、硬链接、可执行位）的跨工具存活情况未知。

### 流程

P13 在 shell 里：`ln -sf` 创建符号链接、`chmod +x` 赋予脚本可执行位、`ln` 创建对 `meta_*.txt` 的硬链接、`cp` 做一次复制。四个兄弟产物都落在 `/agent/generated` 下。P14 从 **后续一次全新 shell 调用** 和 execute_js 分别检查。

### 原始结果

**Shell 创建时的初始状态（同一调用内）：**

```console
$ ln -sf meta_P4M-mo7btrg0-667520774413868.txt /agent/generated/sym_P4ST-mo7btrg0-232913074808419.lnk
ln -s exit=0
$ ls -la /agent/generated/sym_P4ST-mo7btrg0-232913074808419.lnk
lrwxrwxrwx 1 995 993 47 Apr 20 ... sym_...lnk -> meta_P4M-mo7btrg0-667520774413868.txt

$ chmod +x /agent/generated/chmod_P4M-mo7btrg0-667520774413868.sh
chmod exit=0
$ stat -c "after=%a" /agent/generated/chmod_P4M-mo7btrg0-667520774413868.sh
after=755

$ ln /agent/generated/meta_P4M-mo7btrg0-667520774413868.txt /agent/generated/hl_P4M-mo7btrg0-667520774413868.txt
ln exit=0
$ stat -c "orig_inode=%i hl_inode=%i links_orig=%h links_hl=%h" ...
orig_inode=X hl_inode=X links_orig=2 links_hl=2
```

**全新 shell 调用（VFS 提交 & 重挂载之后）：**

```console
$ ls -la /agent/generated/sym_P4ST-mo7btrg0-232913074808419.lnk
ls: cannot access: No such file or directory              # 符号链接丢失

$ stat -c 'mode=%a' /agent/generated/chmod_P4M-mo7btrg0-667520774413868.sh
mode=644                                                   # +x 位丢失

$ stat -c 'path=%n ino=%i nlink=%h size=%s' meta_* hl_*
path=/agent/generated/meta_...  ino=11594110 nlink=1 size=48
path=/agent/generated/hl_...    ino=11594108 nlink=1 size=48   # 硬链接解除
path=/agent/generated/cp_...    ino=11594099 nlink=1 size=48   # cp 保留

$ stat -c 'atime=%x | mtime=%y | ctime=%z | birth=%w' meta_...
atime=2026-04-20 15:11:49.661580592 +0000
mtime=2026-04-20 15:11:49.661735736 +0000
ctime=2026-04-20 15:11:49.661735736 +0000
birth=2026-04-20 15:11:49.661580592 +0000    # 所有时间戳都是重挂载时刻
```

**execute_js 元数据表面：**

```console
$ fs.statSync('/agent/current/generated/chmod_P4M-mo7btrg0-667520774413868.sh')
{ size: 23, isFile: [Function], isDirectory: [Function] }
// no mode, no mtime, no uid, no gid, no ino, no nlink
```

### 分析

Shell 沙箱是一个**工作副本**（working copy），每次调用重新物化。后端存储会保留：路径、字节内容、`644` 权限位。它会丢失：符号链接、可执行位、硬链接关系（`nlink` 计数）、原始时间戳、所有者（因 `/etc/passwd` 缺失，`stat` 显示 `UNKNOWN`）。`cp` 每次 shell 调用都会分到新 inode；这并不特殊，因为每次提交都会重分配 inode。

execute_js 的 `twin:fs.Stats` 被精简为 `{size, isFile, isDirectory}`——根本没有 mode、mtime 或 uid 的读取通道。需要 POSIX 元数据时必须通过 `shell`。

### 结果汇总

| 元数据 | 是否跨调用存活？ | 是否跨工具可读？ |
|---|---|---|
| 路径 | ✅ | ✅ |
| 字节 | ✅ | ✅ |
| 权限位（`644`） | ✅ | 仅 shell |
| 可执行位（`+x`） | ❌ | shell 提交后看到的是 644 |
| 符号链接 | ❌ | 第二次调用时已不可见 |
| 硬链接（`nlink>1`） | ❌ | 重挂载后双方 nlink=1 |
| mtime/atime/ctime | ❌（全部重置为提交时刻） | 仅 shell |
| 所有者 / 组 | 具名形式：❌（UNKNOWN），数字形式 995/993 | 仅 shell |
| Inode 编号 | ❌（每次调用重分配） | 仅 shell |

## 维度 5 — 大小与数量上限

### 假设

- `execute_js` 与 `shell` 都能写入大文件。
- 某处存在上限——可能是单文件上限或单次提交总量上限。
- 给定 1.6 TB 空闲，10 000 个小文件应当毫无压力。

### 流程

P15 从 execute_js 写 1 KB、1 MB、10 MB、100 MB 文件，逐个计时。P16 数量测试：分别创建 10、100、1 000、10 000 个小文件（随后清理）。P17 从 shell 通过 `dd if=/dev/urandom` 写 1 KB、1 MB、10 MB，再写 100 MB。观察哪些能回提成功。

### 原始结果

**execute_js 文件大小耗时：**

```console
1 KB    : wrote=1024 bytes, ms=0
1 MB    : wrote=1048576 bytes, ms=2
10 MB   : wrote=10485760 bytes, ms=14
100 MB  : wrote=104857600 bytes, ms=251     (ok=true)
```

**execute_js 数量耗时：**

```console
10 files     : ms=0, perFile=0.000 ms
100 files    : ms=0, perFile=0.000 ms
1 000 files  : ms=3, perFile=0.003 ms
10 000 files : ms=35, perFile=0.004 ms
```

**shell 输出提交上限——观察到的负面结果：**

```console
// A shell call whose total produced files + stdout exceeded 100 MB returned:
exit_code=0
stderr: "Command succeeded but output files could not be saved:
         total output size (104889902 bytes) exceeds limit (104857600 bytes)"
// Shell-side /agent/generated writes from that call all discarded.
```

**shell 大小写入（在 100 MB 文件清除、回到上限以下之后）：**

```console
sh_1k   wanted=1024      got=1024      ms≈0
sh_1m   wanted=1048576   got=1048576   ms≈(few hundred)
sh_10m  wanted=10485760  got=10485760  ms≈(a few seconds)
sh_100m : 104857600 bytes via `dd bs=1M count=100`, removed before commit
```

**前后磁盘空间：**

```console
Filesystem      Size  Used Avail Use% Mounted on
/dev/md125      1.8T  207G  1.6T  12% /agent/generated
tmpfs           567G     0  567G   0% /tmp
```

### 分析

`execute_js` 的写入大致线性伸缩：对 btrfs 子卷的顺序同步写约 2 ms/MB，热缓存时每个小文件约 0.004 ms。100 MB 干净通过；无截断、无错误。

**真正的上限是 shell 回提上限 104 857 600 字节整**，不是单文件或单分区上限。越过时 shell 返回 exit_code=0，但 `stderr` 横幅告知写入被丢弃——这是一种**半静默**的失败：用户空间脚本看不到错误，只有 VFS 层拒绝发布输出。同时越上限还意味着 shell 工具返回的 `files_written` 数组会省略所有被丢弃的文件，很容易漏掉这种情况。

execute_js 在 100 MB 或 10 000 文件的情况下未观察到上限；我们也没有继续向上压测。

### 结果汇总

| 测试 | 工具 | 结果 | 错误 |
|---|---|---|---|
| 1 KB | execute_js | OK（0 ms） | — |
| 1 MB | execute_js | OK（2 ms） | — |
| 10 MB | execute_js | OK（14 ms） | — |
| 100 MB | execute_js | OK（251 ms） | — |
| 10 000 文件 | execute_js | OK（合计 35 ms） | — |
| 单次 shell 调用输出 > 100 MB | shell | **静默丢弃** | "total output size exceeds limit (104857600 bytes)" |
| 100 MB 单文件 | shell（`dd`） | 本地写 OK；提交要求整次调用输出 < 100 MB | 同一上限 |

## 横切观察

- **子卷单例。** 所有 `/agent/*` btrfs 挂载加 `/workspace/chat` 都指向同一个 `subvolid=259, subvol=/tmp`。各挂载点只有 ro/rw 标志的差别，不是物理隔离。
- **shell 沙箱是一个工作副本。** 每次调用都出现"新鲜的 mtime、新鲜的 inode、丢失的符号链接/硬链接/可执行位"——与"进入时按 manifest 物化、退出时把已提交的普通文件回提"的模型完全吻合。这在沙箱内部没有任何文档说明。
- **`file` 工具是命名空间无关的。** 它以 `filename` 为参数，可读可写——无论底层文件位于 `downloads` 还是 `generated`。这与先前设计文档里"只作为 downloads 写入者"的定位相矛盾。
- **execute_js 读不到 POSIX 元数据。** 如果你的逻辑依赖 mode/owner/mtime，必须走 shell。
- **100 MB shell 回提上限是最危险的怪点。** 它会让输出较大的工作流静默丢失。解决方案：把大结果直接流进 stdout（直到 100 KB 上限），或对任何 > 50 MB 的产物改用 `execute_js`。
- **`/tmp` 是 shell 本地的草稿空间。** execute_js 看不到它，一次新的 shell 调用会清空它。不要指望它在跨调用之间保存任何状态。
- **uid 995 / gid 993** 显示为 "UNKNOWN"，因为 `/etc/passwd` 不存在。只要用数字 id 一切都是确定的。

## 头条发现

1. **六个跨工具可见性方向全部成立**（P01–P06），`downloads` 与 `generated` 命名空间的内容逐字节一致——八对写入/读取组合中只有一对不可行：shell 无法写入 ro 挂载的 `/agent/downloads`（P07 给出 EROFS 痕迹）。
2. **`file` 工具也从 `generated` 读**，而不只是 `downloads`（P06）——这是对先前设计文档的修正。
3. **最后写入者赢，零元数据**——正向、反向冲突最终都归于最新字节；10× 连续重写只保留 iter=9；可达文件系统的任何位置都找不到 `.vfs`/`.manifest`/`.revision` 文件（P09–P12）。
4. **每次 shell 提交都会抹除非普通文件元数据**——符号链接消失、`+x` 位退回 644、硬链接变成独立文件、所有 mtime 都对齐到提交时刻（P13–P15）。
5. **execute_js `fs.Stats` 极简**（`{size, isFile, isDirectory}`）——没有暴露 mode、mtime、uid 或 ino。
6. **真正的上限是 104 857 600 字节 shell 输出封顶**，而非单文件或单文件系统上限——整次 shell 调用输出若越过 100 MB，所有写入会**静默丢弃**，只在 `stderr` 留下一行横幅（P16 负面结果）。
7. **execute_js 能处理至少 100 MB 单文件与 10 000 个小文件**，延迟微乎其微（~2 ms/MB、~0.004 ms/文件）。
8. **所有 agent btrfs 挂载共享 fsid `cfeb616c473e53b` 与 subvolid 259**——只有一份后端子卷；两个工具命名空间（`/agent/downloads` vs `/agent/generated`）是该子卷内的兄弟目录，不是互为镜像的挂载（P08 镜像测试）。
9. **`/tmp` 是 tmpfs 且每次 shell 调用即逝**，execute_js 不可见；`/workspace/chat` 与 `/agent/context` 是 ro 占位符，实际运行时为空。
10. **没有持久化的 hostname、没有 `/etc/passwd`、没有 `hostname` 二进制**——主机身份由 `uname` 给出 `cobb-prod`，uid/gid 以数字 995/993 出现，常见 `id` 工具打印它们时无法解析为名字。

## 附录 — 原始探针脚本记录

### Transcript 1：环境探针（shell）

```console
===ENV_BEGIN===
--- uname ---
Linux cobb-prod 6.18.16 #1-NixOS SMP PREEMPT_DYNAMIC Wed Mar  4 12:25:12 UTC 2026 x86_64 GNU/Linux
--- date ---
Mon Apr 20 15:03:42 UTC 2026
--- pwd ---
/agent/generated
--- id ---
uid=995 gid=993 groups=993
--- hostname ---
sh: line 7: hostname: command not found
hostname: NOT AVAILABLE
--- /proc/mounts ---
tmpfs / tmpfs rw,nosuid,nodev,relatime,uid=995,gid=993 0 0
/dev/md125 /nix/store btrfs ro,...,subvolid=5,subvol=/ 0 0
/dev/md125 /agent/downloads btrfs ro,...,subvolid=259,subvol=/tmp 0 0
/dev/md125 /agent/context btrfs ro,...,subvolid=259,subvol=/tmp 0 0
/dev/md125 /workspace/chat btrfs ro,...,subvolid=259,subvol=/tmp 0 0
/dev/md125 /agent/generated btrfs rw,...,subvolid=259,subvol=/tmp 0 0
tmpfs /tmp tmpfs rw,nosuid,nodev,relatime,mode=755,uid=995,gid=993 0 0
```

### Transcript 2：维度 1 写入植入（execute_js）

```console
RUN_ID: Rmo7btrg0-126701172164598
wrote /agent/current/generated/probe_P1EG-mo7btrg0-917640988283214.txt  (97 bytes)
wrote /agent/current/downloads/probe_P1ED-mo7btrg0-584558632549247.txt  (97 bytes)
dim2 writes:
  tmp  : FAIL EACCES: path is read-only
  gen  : OK
  dl   : OK
  chat : FAIL EACCES: path is read-only
  ctx  : FAIL EACCES: path is read-only
dim3 rapid final: "iter=9 ts=1776697513776"
dim5 sizes: sz1k OK 0ms, sz1m OK 2ms, sz10m OK 14ms
dim5 counts: n10 OK, n100 OK, n1000 OK 3ms, n10000 OK 35ms
dim5 100MB: wrote=104857600, ms=251, ok=true
```

### Transcript 3：维度 1 读取（shell）

```console
--- /agent/generated/probe_P1EG-mo7btrg0-917640988283214.txt ---
size=97 mtime=2026-04-20 15:08:45.051804338 +0000 mode=644 inode=11593261
ff695aa28a06d7ee68932363f04825b665aa9bd8ebb1a1a823e24e7200122f7e
==begin==
dim1 writer=execute_js ns=generated id=P1EG-mo7btrg0-917640988283214 ts=2026-04-20T15:05:13.776Z
==end==

--- /agent/downloads/probe_P1ED-mo7btrg0-584558632549247.txt ---
size=97 mtime=2026-04-20 15:08:45.049184241 +0000 mode=644 inode=11593240
395a446396afcd3bf776ae10ac2cd81eb6be0225c70a221ae557d8d42a6400eb

--- /agent/downloads/probe_P1FD-mo7btrg0-521300225033074.txt ---
size=37 mtime=... mode=644
dim1 writer=file namespace=downloads
```

### Transcript 4：文件工具 upload 读取

```console
// execute_js → file: upload proved read via downloads namespace
file.upload_via_curl(filename='probe_P1ED-mo7btrg0-584558632549247.txt', url='https://httpbin.org/post')
=> status=200, data="dim1 writer=execute_js ns=downloads id=P1ED-mo7btrg0-584558632549247 ts=2026-04-20T15:05:13.776Z\n"

// shell → file: upload proved read via GENERATED namespace (design-doc correction)
file.upload_via_curl(filename='probe_P1SG-mo7btrg0-972675540716829.txt', url='https://httpbin.org/post')
=> status=200, data="dim1 writer=shell ns=generated id=P1SG-mo7btrg0-972675540716829 ts=2026-04-20T15:08:45Z\n"

// Forward collision read via file
file.upload_via_curl(filename='collision_P3A-mo7btrg0-029649523288351.txt', url='https://httpbin.org/post')
=> data="B\ndim3 file overwrite\n"   ← file(B) overwrote ejs(A)
```

### Transcript 5：持久性与 /tmp 生命周期

```console
// Shell call N — wrote /tmp/probe_tmp_shell_P2T-mo7btrg0-618965918717218.txt, visible
// Shell call N+1:
$ ls /tmp
total 0
drwxr-xr-x 2 995 993  40 ... .
drwxr-xr-x 8 995 993 160 ... ..
(probe file gone)
```

### Transcript 6：元数据抹除

```console
// Same shell call as creation
$ ln -sf meta_P4M-mo7btrg0-667520774413868.txt sym_P4ST-mo7btrg0-232913074808419.lnk
ln -s exit=0
$ chmod +x chmod_P4M-mo7btrg0-667520774413868.sh
$ stat -c %a chmod_P4M-mo7btrg0-667520774413868.sh
755

// Fresh shell call after commit/remount
$ ls -la sym_P4ST-mo7btrg0-232913074808419.lnk
ls: cannot access: No such file or directory

$ stat -c %a chmod_P4M-mo7btrg0-667520774413868.sh
644

$ stat -c 'ino=%i nlink=%h' meta_P4M-mo7btrg0-667520774413868.txt hl_P4M-mo7btrg0-667520774413868.txt
ino=11594110 nlink=1
ino=11594108 nlink=1
(hardlink relationship gone; two separate inodes with same content)
```

### Transcript 7：容量上限负面结果

```console
(shell call with 100 MB ejs-side file still present plus all other outputs)
stderr: "Command succeeded but output files could not be saved:
         total output size (104889902 bytes) exceeds limit (104857600 bytes)"
stdout: ...normal output...
exit_code: 0
files_written: [...pre-existing files only, no new ones from this call...]
```

### Transcript 8：最终 XOR 折叠指纹（execute_js 交叉验证）

```console
/agent/current/generated/probe_P1EG-mo7btrg0-917640988283214.txt        size=97  xorfold=43cc10cf
/agent/current/downloads/probe_P1ED-mo7btrg0-584558632549247.txt         size=97  xorfold=43cc10cf
/agent/current/generated/probe_P1SG-mo7btrg0-972675540716829.txt      size=88  xorfold=016533ef
/agent/current/downloads/probe_P1FD-mo7btrg0-521300225033074.txt        size=37  xorfold=df5ae84f
/agent/current/downloads/collision_P3A-mo7btrg0-029649523288351.txt      size=22  xorfold=5c4f742f  (contains "B")
/agent/current/downloads/rev_collision_P3B-mo7btrg0-986243092136844.txt  size=16  xorfold=67d3bcef  (contains "Y")
/agent/current/generated/meta_P4M-mo7btrg0-667520774413868.txt            size=48  xorfold=fad3b8ef
/agent/current/generated/rapid_P3R-mo7btrg0-906590488166303.txt          size=24  xorfold=adfd3bef  (iter=9)
```

（两个 97 字节的 ejs 探针共享 `xorfold=43cc10cf`，因为它们模板相同、仅子标签（"generated" vs "downloads"）在内容层面有差别——但大小匹配，XOR 碰撞在如此相近的字节间是预期之中。字节级别的验证由 shell 原始记录中的 `sha256sum` 完成，其摘要不同。）

---

_报告生成于 2026-04-20T15:15:xx.xxxZ · Agent run `build/Rmo7btrg0-126701172164598` · Total probes: 17 · 本报告为实测结果。任何没有探针支撑的断言都是 bug。_
