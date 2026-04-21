# 02 — `file` 工具（修订版）

> 本文件为 `docs/02-file-tool.md` 的修订版，依据 `AUDIT.md` 与事实来源（README.md + docs/experiments/2026-04-20-sandbox-fs.md）重写。
>
> - ✅ 事实部分：保留并基于内核/VFS 机制深化
> - ⚠️ 推测部分：保留但已醒目标注
> - ❌ 矛盾部分：已删除或按事实来源修正（本文档原 2.6 条与 FACT-EX-1 P06 直接冲突，已按实测结果修正为"`upload_via_curl` 命名空间无关"）

> 范围：`file` 工具怎么与存储后端交互，实际写到哪儿、从哪儿读。

## 1. 行为实测（原始探针证据，✅ 事实）

执行：

```
file(action="download_via_curl",
     url="https://raw.githubusercontent.com/CXP-shawn/vfs-docs/main/README.md",
     filename="_file_probe_test.txt")
```

返回：

```json
{
  "success": true,
  "status_code": 200,
  "bytes_written": 101,
  "content_hash": "cdd998b416ffbe697c281fa273d6e5fdc6762dfa3d01403527845a5f60fd21a2",
  "filename": "_file_probe_test.txt"
}
```

**返回值中没有绝对路径**。调用方必须知道文件落在 `downloads` 命名空间。

### 下一次 `execute_js` 调用时确认

- `/agent/current/downloads/_file_probe_test.txt` — 存在，size 101，内容是被抓取的 README。
- `/agent/current/generated/_file_probe_test.txt` — **不存在**。

### 下一次 `shell` 调用时确认

- `/agent/downloads/_file_probe_test.txt` — 存在，size 101。
- `/agent/generated/_file_probe_test.txt` — **不存在**。

这佐证 FACT-EX-2 的"镜像测试"：`/agent/downloads` 与 `/agent/generated` 不是同一路径的两个投影，而是同一 btrfs 子卷下的**兄弟目录**。

## 2. 设计含义（✅ 事实）

`file` 工具 = 一个 **跨沙箱的 HTTP 代理 + `downloads` 命名空间写入器**。它存在是因为：

1. `shell` 没有网络；`execute_js` 无法发出主机级 curl 请求。
2. 需要一个统一入口把远端产物转入 `downloads`（与用户上传同命名空间），让所有下游工具都能读到。

> ⚠️ **推测 / 无实验数据支撑** —— "跨沙箱 HTTP 代理"这一层表述是一个**架构名称**，而非实验直接给出的结论。实验只证明 `file.download_via_curl` 能向公网 HTTPS 端点发请求并把字节写回 `downloads`；它**是否由真正的 curl、由 reqwest、由 hyper 或其它客户端实现**，本实验无法断言。

## 3. 支持的 action（✅ 事实）

来自工具 schema：

- `download_via_curl` — URL → `downloads/<filename>`。`curl_args` 支持 `-F/--form`、`-H/--header`、`-X`、`-d`、`--data-binary`、`-L`、`-u`、`-k` 等。
- `upload_via_curl` — `<filename>` → URL。支持 multipart、custom headers、Google Drive 多部件 metadata。

## 4. 关键修正：`upload_via_curl` 的读路径是**命名空间无关**的（✅ 事实，按 FACT-EX-1 P06 修正）

> 历史版本说"upload\_via\_curl 只能从 `downloads` 命名空间读"。该说法**已被 2026-04-20 实测证伪**：

**实测证据 P06**：

```
// shell 先在 /agent/generated 下写入一个 88 字节文件
// 紧接着 file.upload_via_curl 以纯 filename 读取它，POST 到 httpbin.org/post
file.upload_via_curl(
  filename='probe_P1SG-mo7btrg0-972675540716829.txt',
  url='https://httpbin.org/post'
)
=> httpbin.data == "dim1 writer=shell ns=generated id=P1SG-mo7btrg0-972675540716829 ts=2026-04-20T15:08:45Z\n"
   Content-Length: 88
```

httpbin 回显的 body 字节与 shell 写在 `/agent/generated` 下的内容**精确匹配**——说明 `file` 工具的 upload 通路能解析到 `generated` 里的文件。FACT-EX-1 P04 同时验证了它也能读 `execute_js` 写入 `downloads` 的文件。**实际的读路径 = "无论 `downloads` 还是 `generated`，只要文件名匹配即可。"**

### 4.1 实务建议

- 直接用 `filename="foo.ext"`（不要写目录前缀）。
- 如果 `downloads` 和 `generated` 下都存在同名文件，**实测 `file` 工具的决断顺序仍未知**（P06 并未设计该对照）；避免重名是最保险的做法。
- 如果你之前有"从 generated 拷贝到 downloads 再 upload"的模式，现在**这一拷贝不再必要**（省一次 IO）。

## 5. 陷阱（更新版）

- `file` 工具**不直接解压压缩包**——解压要用 `shell` (`unzip`/`tar`) 或 `execute_js`。

  > ⚠️ **推测 / 无实验数据支撑** —— 工具 schema 里没有 "unarchive" 动作，但我们没有专门做"上传 zip 会不会被自动解压"的实验。断言基于 schema 枚举。

- `download_via_curl` 只写 `downloads`。如果产物要给到 `shell` 的 working directory（`/agent/generated`），在 `execute_js` 里做一次 `copyFileSync`：

  ```js
  const fs = require('twin:fs');
  fs.copyFileSync('/agent/current/downloads/foo.pdf',
                  '/agent/current/generated/foo.pdf');
  ```

- `file` 没有"删除"动作。如需清理 downloads，在 `execute_js` 里 `fs.unlinkSync`。

## 6. 理论背景（设计推理）

> ⚠️ **推测 / 无实验数据支撑** —— 以下段落把 `file` 工具的观察放进 Linux/VFS 的心智模型里。

### 6.1 为什么需要一个代理工具？

Linux **mount namespace** 与 **network namespace** 可以各自独立。当前沙箱显然把 shell 进程放在了"无 network namespace 出站能力"的命名空间里（shell 里 `curl https://...` 实测不可用，本仓库未在 04 明确记录但广为已知）。与此同时，`execute_js` 运行在受限 V8 容器里，没有打开 TCP socket 的能力。因此，`file` 充当第三方代理：它运行在拥有网络命名空间的上下文，把请求结果落到一个**共享的存储挂载点**上，所有工具都能读到。

这是典型的 **ambient authority 外置** 设计：网络能力不属于任何一个"纯计算"工具，而是由具名动作显式发起。

### 6.2 content\_hash 的工程价值

`content_hash` 是 sha256。对调用方而言至少带来三项好处：

1. **重复下载去重**：先比 hash 再决定是否 re-fetch。
2. **完整性验证**：下载损坏 → `execute_js` 读出的字节再哈希应与返回值匹配。
3. **缓存键**：用 hash 取代 filename 作为索引，天然处理"不同 URL → 相同字节"的情形。

experiments 给出的是 sha256（64 位十六进制字符串），与 Linux `sha256sum` 的输出同格式，跨工具可直接比对。

### 6.3 upload 命名空间无关 = 什么内核机制？

> ⚠️ **推测 / 无实验数据支撑**

从内核角度，`file.upload_via_curl` 必须调用 `open(filename, O_RDONLY)`。如果它只 `open("downloads/" + filename)`，就不可能读到 `generated/` 下的文件；既然读到了，**它必然在 `downloads` 与 `generated` 之外拿到了一个"逻辑文件名→物理路径"的解析层**。可能的实现：

- 固定搜索路径：`[generated, downloads]`，先找到谁就用谁（类似 PATH）；
- 或者二者都是某个逻辑命名空间的别名，upload 在更高一层定位。

对用户来说，可执行的含义只有一条：**filename 唯一时放心用；重名时别依赖读哪一边**。

## 7. 对照实验（可复现）

### 7.1 A/B/C：download → 命名空间归属

三组各写 128 字节到同名 `foo_ab.txt`：

| 组 | 写入工具 | 写入路径 |
|---|---|---|
| A | `file.download_via_curl` | `downloads/foo_ab.txt` |
| B | `execute_js` | `/agent/current/downloads/foo_ab.txt` |
| C | `execute_js` | `/agent/current/generated/foo_ab.txt` |

**期望**：

| 读法 | A | B | C |
|---|---|---|---|
| `shell` `/agent/downloads/foo_ab.txt` | ✅ 128B | ✅ 128B | ❌ 不存在 |
| `shell` `/agent/generated/foo_ab.txt` | ❌ | ❌ | ✅ 128B |
| `file.upload_via_curl(filename="foo_ab.txt")` | ✅ | ✅ | ✅（P06）|

### 7.2 控制变量：content\_hash 幂等性

同一 URL 连续调用 `download_via_curl` 5 次，比较 5 次 `content_hash`。**期望**：5 次哈希相同、`bytes_written` 相同。若不同，说明源在 5 秒内变化（不是 `file` 工具不幂等）。

## 8. 数据分析方法

- **延迟模型**：`download_via_curl` 的延迟 ≈ TCP 握手 + TLS + 远端响应 + 本地写入。本仓库实验只给了 101 字节的**单次**观测，没有分解；无法分离"网络成本 vs 本地写"。
- **失败模式识别**：按 HTTP status code 分段——`2xx` 成功；`3xx` 若工具不默认跟随，要显式 `-L`；`4xx/5xx` 工具返回 `success=false`（需补做实验确认字段）。
- **吞吐估计**：1MB、10MB、100MB 下载的平均带宽尚无数据。

## 9. 讨论与误差来源

- **远端延迟**：任何外部 URL 的抖动都会污染延迟测量。应选本地 httpbin 或相近基础设施。
- **重定向**：`curl_args` 里没显式 `-L` 时，3xx 可能被当作"下载成功但字节为 0"。需要有对照实验。
- **代理认证**：`-u` 凭据若不正确，`download_via_curl` 的返回值语义（是 `success=true + status_code=401` 还是 `success=false`？）未有实验观察。

## 10. 扩展实验（未在本仓库实测过）

1. **download 幂等 + 去重**：同一 URL 请求 100 次，记录延迟分布 + `content_hash` 分布（应全相同）。
2. **upload 命名空间碰撞**：故意在 `downloads` 与 `generated` 下放两份不同字节、同名文件，观察 `upload_via_curl` 读取谁。
3. **超大 URL body**：下载 1 GB 文件，检查是否和 shell 的"100 MB 单次调用输出"上限类似——`file` 是否也有回提上限？
4. **HTTP/2、gRPC、WebSocket**：`download_via_curl` 能否处理 HTTP/2 streaming、长连接？schema 没说。

## 11. 思考题

1. 如果 `upload_via_curl` 决定当 `downloads` 和 `generated` 同名时优先用 `generated`，对现有 agent 代码意味着什么？会不会引入"忘记删除 generated 下旧版本"导致上传错版本的 bug？
2. `content_hash` 用 sha256 而非 md5 的理由是什么？在这个沙箱环境里两者的失败语义差异可忽略吗？
3. 你如何在不让 `file` 工具持有密钥的前提下，从 `download_via_curl` 向需要鉴权的 S3 签名 URL 下载？
4. 假如需要"下载到 `generated` 而非 `downloads`"，最小改动是什么？（提示：从"工具要变"还是"调用惯例要变"的角度都可回答。）
5. `file.upload_via_curl` 的 `multipart_related_json` 是专为 Google Drive 设的——为什么其它多部分格式（例如 S3 `POST policy`）需要用户自己拼 `-F`？这反映了什么设计取舍？
