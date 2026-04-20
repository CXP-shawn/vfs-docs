# 03 — `execute_js` + `twin:fs`（实测）

> 范围：`execute_js` 沙箱的文件系统视图与能力边界。

## 可见根

`fs.readdirSync('/')` 返回：`['agent', 'workspace']`。

`fs.readdirSync('/agent')` 返回：`['current']` — **只有 `/agent/current`**；没有 `/agent/downloads`、`/agent/generated`、`/agent/context`。

`fs.readdirSync('/agent/current')` 返回：`['downloads', 'generated']`。

`fs.readdirSync('/workspace')` 在 `execute_js` 里不可读取（未探测到内容）。

**`/tmp` 和 `/` 的其他路径都不存在于 `execute_js` 的视图里。**

## 可写路径

`twin:fs` 允许 `writeFileSync` 的路径只有：
- `/agent/current/downloads/*`
- `/agent/current/generated/*`

其它任何路径都会 `EACCES: path is read-only`，实测包括：
- `/agent/downloads/...` — 不存在
- `/agent/generated/...` — 不存在
- `/tmp/...` — 不存在
- `/workspace/...` — read-only

## 持久性

实测多次 `writeFileSync → storeVar → ` 再下一次 `execute_js` readFileSync`` 链路，全部成功。`execute_js` 的写对后端是**立即提交**的，没有观察到丢失现象。

## 跨工具可见性（实测）

| 场景 | 结果 |
|---|---|
| `execute_js` 写 `/agent/current/generated/foo.txt` → 下一次 `shell` 读 `/agent/generated/foo.txt` | ✅ 可见，内容一致 |
| `execute_js` 写 `/agent/current/downloads/foo.txt` → 下一次 `shell` 读 `/agent/downloads/foo.txt` | ✅ 可见 |
| `shell` 写 `/agent/generated/foo.txt` → 下一次 `execute_js` 读 `/agent/current/generated/foo.txt` | ✅ 可见（多数情况下；见 04/05 对 shell 罕见丢失的说明） |
| `file` 下载 `foo.txt` → `execute_js` 读 `/agent/current/downloads/foo.txt` | ✅ 可见 |

## API 表面

`require('twin:fs')` 提供同步 Node.js 风格 API：
```js
fs.readFileSync(path, encoding?)
fs.writeFileSync(path, data, encoding?)
fs.readdirSync(path, options?)
fs.statSync(path)
fs.existsSync(path)
fs.mkdirSync(path, options?)
fs.unlinkSync(path) / fs.rmSync(path)
fs.renameSync(oldPath, newPath)
fs.copyFileSync(src, dest)
```

**不可用**：`crypto` 模块（实测抛 `Cannot find module 'crypto'`），`Buffer.isBuffer` 的行为与标准 Node 不完全一致。如需哈希，把数据落到 `shell` 再用 `sha256sum`。

## 原子性

来自工具描述：**单次 `execute_js` 调用内的所有写操作是原子的**——脚本抛异常时所有写入都被丢弃。实测未做针对性验证，按官方声明执行。
