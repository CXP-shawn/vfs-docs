# 01 — 用户上传与工作区（实测）

> 范围：搞清楚 `downloads` 命名空间里的文件从哪来、谁能读、谁能写。

## 实测数据

本 agent 的 `downloads` 命名空间在探测时包含：

| 文件名 | 大小 | 来源 | sha256 (shell 读) |
|---|---|---|---|
| `test_readme.md` | 2279 B | **用户聊天上传** | `1adc06cb1008208c356da0982fc8b81ee6d006695aae02f8464370ad87d78a5d` |
| `doubao_avatar.png` | 26799 B | **用户聊天上传** (PNG 160x160 RGBA) | — |
| `_file_probe_test.txt` | 101 B | **file 工具下载**（本次探测产生） | — |
| `_twinfs_probe.txt` | 26 B | execute_js 写入（探测产生） | — |
| `_ejs_marker.txt` | 24 B | execute_js 写入（探测产生） | — |

## 两种访问路径

| 工具 | 绝对路径 | 读 | 写 |
|---|---|---|---|
| `shell` | `/agent/downloads/<name>` | ✅ | ❌ (EROFS) |
| `execute_js` + `require('twin:fs')` | `/agent/current/downloads/<name>` | ✅ | ✅ |
| `file` | 通过 `filename` 参数 | N/A | `download_via_curl` 写到此处 |

**shell 不能往 `downloads` 写**。如需从 shell 把产物放到 `downloads`，必须走 `execute_js` 拷贝（或反过来：让 shell 写到 `/agent/generated`，然后应用层自己约定从 `generated` 读）。

## 典型工作流

### 读用户上传的文件

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

注意 shell 的输出必须落在 `/agent/generated`（如上第二例），因为 `/agent/downloads` 只读。

### 从 URL 下载到 downloads

用 `file` 工具：
```
file(action="download_via_curl", url=..., filename="foo.pdf")
```
返回 `bytes_written`、`content_hash`（sha256）。之后：
- `execute_js` 通过 `/agent/current/downloads/foo.pdf` 访问
- `shell` 通过 `/agent/downloads/foo.pdf` 访问（只读）

## 谁能写 `downloads`？

- ✅ `execute_js`  (`twin:fs`)
- ✅ `file.download_via_curl`
- ❌ `shell`（只读）
- ❌ 其它工具
