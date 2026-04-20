# 02 — `file` 工具（实测）

> 范围：`file` 工具怎么与存储后端交互，实际写到哪儿。

## 行为实测

执行 `file(action="download_via_curl", url="https://raw.githubusercontent.com/CXP-shawn/vfs-docs/main/README.md", filename="_file_probe_test.txt")` 返回：

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

下一次 `execute_js` 调用时确认：
- `/agent/current/downloads/_file_probe_test.txt` — 存在，size 101，内容是被抓取的 README。
- `/agent/current/generated/_file_probe_test.txt` — **不存在**。

下一次 `shell` 调用时确认：
- `/agent/downloads/_file_probe_test.txt` — 存在，size 101。
- `/agent/generated/_file_probe_test.txt` — **不存在**。

## 设计含义

`file` 工具 = 一个"跨沙箱的 HTTP 代理 + `downloads` 命名空间写入器"。它存在是因为：

1. `shell` 没有网络；`execute_js` 无法发出主机级 curl 请求。
2. 需要一个统一入口把远端产物转入 `downloads`（与用户上传同命名空间），让所有下游工具都能读到。

## 支持的 action

来自工具 schema：
- `download_via_curl` — URL → `downloads/<filename>`，支持 `curl_args`（-H, -u, -L 等）。
- `upload_via_curl` — `downloads/<filename>` → URL，支持 multipart、custom headers、Google Drive 多部件 metadata。

## 陷阱

- `upload_via_curl` 只能从 `downloads` 命名空间读。**从 `generated` 产生的文件若要上传，要先拷到 `downloads`**（用 `execute_js` 做 `readFileSync` + `writeFileSync`）。
- `file` 工具不直接解压压缩包——解压要用 `shell` (`unzip`/`tar`) 或 `execute_js`。
