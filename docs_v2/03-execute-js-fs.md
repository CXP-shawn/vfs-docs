# 03 — `execute_js` + `twin:fs`（修订版）

> 本文件为 `docs/03-execute-js-fs.md` 的修订版，依据 `AUDIT.md` 与事实来源（README.md + docs/experiments/2026-04-20-sandbox-fs.md）重写。
>
> - ✅ 事实部分：保留并基于内核/VFS 机制深化
> - ⚠️ 推测部分：保留但已醒目标注
> - ❌ 矛盾部分：已删除或按事实来源修正

> 范围：`execute_js` 沙箱的文件系统视图与能力边界。

## 1. 可见根（✅ 事实）

- `fs.readdirSync('/')` 返回：`['agent','workspace']`。
- `fs.readdirSync('/agent')` 返回：`['current']` —— **只有 `/agent/current`**；没有 `/agent/downloads`、`/agent/generated`、`/agent/context`。
- `fs.readdirSync('/agent/current')` 返回：`['downloads','generated']`。

> ⚠️ **推测 / 无实验数据支撑** —— 原文："`fs.readdirSync('/workspace')` 在 `execute_js` 里不可读取（未探测到内容）"。experiments 直接探测的是 "`execute_js` 对 `/workspace/chat` 写入 → EACCES"、"`/workspace/chat` 对 shell ro 且空"。并没有一次 `readdirSync('/workspace')` 的原始 transcript，因此"不可读取"是对可见性的一种**推断表述**，严谨说应改为"该路径在 `execute_js` 视图内不可写且实际运行时为空"。

**`/tmp` 和 `/` 的其他路径都不存在于 `execute_js` 的视图里**（ejs 调用 `fs.readdirSync('/tmp')` 会直接 `ENOENT`）。

## 2. 可写路径（✅ 事实）

`twin:fs` 允许 `writeFileSync` 的路径**只有**：

- `/agent/current/downloads/*`
- `/agent/current/generated/*`

其它任何路径都会 `EACCES: path is read-only`，实测包括：

- `/agent/downloads/...` — 不存在
- `/agent/generated/...` — 不存在
- `/tmp/...` — 不存在
- `/workspace/...` — read-only

## 3. 持久性（✅ 事实）

实测多次 `writeFileSync → storeVar → (下一次调用) readFileSync` 链路，全部成功。`execute_js` 的写对后端是**立即提交**的，没有观察到丢失现象。

## 4. 跨工具可见性（实测，✅ 事实）

| 场景 | 结果 |
|---|---|
| `execute_js` 写 `/agent/current/generated/foo.txt` → 下一次 `shell` 读 `/agent/generated/foo.txt` | ✅ 可见，内容一致 |
| `execute_js` 写 `/agent/current/downloads/foo.txt` → 下一次 `shell` 读 `/agent/downloads/foo.txt` | ✅ 可见 |
| `shell` 写 `/agent/generated/foo.txt` → 下一次 `execute_js` 读 `/agent/current/generated/foo.txt` | ✅ 可见（多数情况下；见 04 / 05 对 shell 罕见丢失的说明） |
| `file` 下载 `foo.txt` → `execute_js` 读 `/agent/current/downloads/foo.txt` | ✅ 可见 |

## 5. API 表面（✅ 事实）

`require('twin:fs')` 提供同步 Node.js 风格 API：

```js
const fs = require('twin:fs');
fs.readFileSync(path, encoding?)       // utf8 → string；省略 → Buffer
fs.writeFileSync(path, data, encoding?) // data: string | Buffer | Uint8Array
fs.readdirSync(path, options?)          // {withFileTypes:true} → Dirent[]
fs.statSync(path)                       // { size, isFile(), isDirectory() }
fs.existsSync(path)
fs.mkdirSync(path, options?)            // {recursive:true}
fs.unlinkSync(path) / fs.rmSync(path)
fs.renameSync(oldPath, newPath)
fs.copyFileSync(src, dest)
```

**不可用**（✅ 直接实测）：

- `crypto` 模块 —— 实测抛 `Cannot find module 'crypto'`。FACT-EX 方法论里明说"execute_js 里没有 crypto 模块，因此用 32 位 XOR 折叠做轻量相等性校验"。**需要哈希就落到 `shell` 里用 `sha256sum`。**

> ⚠️ **推测 / 无实验数据支撑** —— "`Buffer.isBuffer` 的行为与标准 Node 不完全一致" 属于经验性告诫，experiments 没有做针对 `Buffer.isBuffer` 的专门探测。如果你的代码依赖它，请先 `console.log(Buffer.isBuffer(data))` 自检。

## 6. 原子性（⚠️ 推测）

> ⚠️ **推测 / 无实验数据支撑** —— 工具描述写道"单次 `execute_js` 调用内的所有写操作是原子的 —— 脚本抛异常时所有写入都被丢弃"。README 作者同样备注"实测未做针对性验证，按官方声明执行"。本仓库实验矩阵（维度 1~5）里没有专门的 "throw-in-middle" 回滚探针。按官方声明执行，但不要把它当作一条已验证的事实。

## 7. 理论背景（设计推理）

> ⚠️ **推测 / 无实验数据支撑** —— 本节用通用 VFS 机制给 `execute_js` 的行为做心智模型，不是实测断言。

### 7.1 从 Node.js `fs` 到内核 `file_operations`

标准 Node.js `fs.readFileSync` 最终走 `fd = open()`、`read(fd, buf, n)`、`close(fd)` 这三条系统调用。内核通过 `struct file_operations`（btrfs 注册的那组）把请求路由到具体文件系统；VFS 层在 open 时解析 dentry → inode，read 时走 page cache。

`twin:fs` 是一个**受限映射**：

- 只暴露了 Node `fs` 的一个子集；
- 只允许 `/agent/current/{downloads,generated}` 两个写子树；
- 去掉了所有异步接口（全部 `*Sync`），避免 V8 事件循环与后端提交事务交叠。

### 7.2 为什么没有 `fsync` / `fdatasync`？

Node `fs.fsyncSync(fd)` 在标准 API 里存在；`twin:fs` 没有暴露。对应的内核调用 `fsync(2)` 负责把 page cache 脏页落到 `btrfs` 事务日志。**在 Twin 沙箱里该操作由运行时代为执行**：

- 一次 `execute_js` 调用结束时（整包提交）；
- 或者调用内部的某个 checkpoint（`writeFileSync` 返回成功之时——这一步**没有** 实验直接证实，只是"下一次 shell/ejs 读得到"的观察推出的）。

### 7.3 `statSync` 为什么裁剪成 `{size, isFile, isDirectory}`？

实测（FACT-EX-4）`execute_js` 的 `Stats` 对象**不暴露 mode、mtime、uid、gid、ino、nlink**。可能的工程动机：

1. 让 agent 不依赖易变元数据（shell 每次提交都会重置 mtime/nlink/+x）。
2. 避免泄漏 uid/gid（整个沙箱 uid=995/gid=993 都在 `/etc/passwd` 之外）。
3. 简化 twin:fs 的后端协议——只需要"存在性 + 类型 + 大小"三个字段。

若下游逻辑必须读 mtime、mode，就**只能**切到 `shell` 里用 `stat -c`。

## 8. 对照实验（可复现）

### 8.1 原子性（throw-in-middle）

```js
const fs = require('twin:fs');
fs.writeFileSync('/agent/current/generated/atom_A.txt', 'A');
throw new Error('abort mid-transaction');
// 期望：下一次调用读 atom_A.txt 要么不存在，要么回到上一次成功写入的内容
```

| 指标 | 期望 |
|---|---|
| 调用结果 | `error: String("abort mid-transaction")` |
| `fs.existsSync('/agent/current/generated/atom_A.txt')`（下一次调用） | `false`（如果整包提交语义成立）|

### 8.2 A/B：字符串写 vs Buffer 写

同一段字节 `Hello ✅`（utf-8 共 10 字节）：

- A：`fs.writeFileSync(p, 'Hello ✅', 'utf8')`
- B：`fs.writeFileSync(p, Buffer.from('Hello ✅','utf8'))`

**期望**：两份文件 size=10，`sha256sum` 一致。

### 8.3 计数基准

```js
const t0=Date.now();
for(let i=0;i<N;i++) fs.writeFileSync(`/agent/current/generated/n_${i}.txt`, String(i));
console.log(Date.now()-t0);
```

实测基线（FACT-EX-5）：`N=10` → 0 ms、`N=100` → 0 ms、`N=1000` → 3 ms、`N=10000` → 35 ms。**平均 ≈ 3.5 µs/文件**。

## 9. 数据分析方法

- **吞吐**：`execute_js` 单次调用 100 MB 文件 251 ms（≈ 400 MB/s）。对比同次 shell 侧 100 MB dd：本仓库只得到"local write OK"这类定性观察，没有同量级的分布数据。
- **方差**：实验只做 **1 次/规模**，没法估 σ。建议每档至少 10 次，剔除首次"冷启动"值，再算均值与 95% CI。
- **文件数 vs 文件大小**：二者的成本模型不同——大文件受 btrfs extent 分配 + 页对齐影响；多文件受 dentry/inode 分配 + 目录项线性插入成本影响。在分析结果时要分别拟合。

## 10. 讨论与误差来源

- **V8 字符串/Buffer 相互转换**：`writeFileSync` 的 utf8 路径多了一次 JS → UTF-8 编码；对等长字节差异不大，但对大字符串显著。
- **page cache 热度**：连续写读的 hot path 命中 page cache；跨调用测 read 时，cache 是否仍温热取决于 Twin 运行时的实现（未观察）。
- **时钟源**：`Date.now()` 精度 1 ms，不足以测 sub-ms 级别的写延迟——看到 `ms=0` 不代表"0 时间"，只代表"在 1 ms 精度下不可分"。
- **多工具穿插**：本文矩阵的"下一次 shell 读"场景，若中途插入 `file.download_via_curl`，路径语义不变，但 VFS 提交次序可能改变——未测。

## 11. 扩展实验（未在本仓库实测过）

1. **exception 回滚的确证**：按 8.1 方案跑 100 次，统计 abort 后磁盘状态（应全是 "不存在" 或 "前一次值"）。
2. **hash 性能对照**：在 ejs 里用纯 JS SHA-256（例如 JS 实现）对 10 MB 数据哈希，与 shell `sha256sum` 对比耗时，量化"没 crypto 模块"的成本。
3. **大目录列表**：在 `generated` 下 `readdirSync` 一个包含 10 万条目的目录，看延迟与内存是否线性。
4. **`renameSync` 原子性**：连续 `writeFileSync('x.part')` + `renameSync('x.part','x')` 是否对观察方总是原子可见。
5. **Buffer 行为探针**：`Buffer.isBuffer`、`Buffer.alloc`、`Buffer.from` 与 `Uint8Array` 的互操作性在这个 V8 版本里的实际表现。

## 12. 思考题

1. 为什么 `twin:fs` 全部用同步 API？如果未来引入异步版本，会打破"单次调用原子化"契约吗？
2. 沙箱拒绝 `/tmp` 的写入，只允许 `/agent/current/{downloads,generated}`。从 capability-based security 的视角看，这种白名单设计比标准 Unix ACL 有什么优势？
3. execute_js 里没有 `crypto`——如果你设计 Twin 运行时，会用什么替代方案同时满足"可审计的哈希"与"不给 agent 任意系统库"这两个目标？
4. 若同一次调用里写了 100 个 1 MB 的文件，整次事务的"回滚窗口"有多大？有哪些工程手段可以缩小这个窗口？
5. `statSync` 只暴露 `{size, isFile, isDirectory}`——在实际业务里这对"增量同步"、"缓存失效"、"文件差异比较"分别意味着什么，它们的代偿工具是什么？
