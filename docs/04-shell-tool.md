# 04 — `shell` 工具（实测）

> `shell` 用的是**长驻工作目录**模式：先把 session 物化到本地磁盘，跑完命令再同步回来。

| 状态 | 负责人 | 最后更新 |
|---|---|---|
| 初稿(对齐当前代码实现) | 周朗多 | 2026-04-20 |

## Scope

`shell` 是最接近"真实 Linux"的工具：

- 一个能 `cd` 进去的真实目录（`working_directory` 参数决定）
- 命令之间保持 cwd/环境
- 像本地磁盘一样支持任意 POSIX 操作

它给你的答案是：**working copy 模式**。

## 真实沙箱形态（实测）

从 `/proc/mounts`：

```
tmpfs / tmpfs rw,...,uid=995,gid=993
/dev/md125 /agent/downloads btrfs ro,...,subvolid=259,subvol=/tmp
/dev/md125 /agent/context   btrfs ro,...,subvolid=259,subvol=/tmp
/dev/md125 /agent/generated btrfs rw,...,subvolid=259,subvol=/tmp
/dev/md125 /workspace/chat  btrfs ro,...,subvolid=259,subvol=/tmp
tmpfs /tmp tmpfs rw,...
```

- 根 (`/`) 是 tmpfs，每次调用重新创建。
- 四个 `/agent/*` 和 `/workspace/chat` 都指向同一个 btrfs 子卷，只是读写权限不同。
- `/tmp` 是独立 tmpfs，每次调用都是空的。
- 用户/组：uid=995 gid=993（不是 root）。

## 与其它工具的路径映射

| 我在 shell 看到 | execute_js 的对应路径 | 说明 |
|---|---|---|
| `/agent/downloads` | `/agent/current/downloads` | 只读，含用户上传 + file 下载 |
| `/agent/generated` | `/agent/current/generated` | 可读写 |
| `/agent/context` | 不可见 | 只读，实验中为空 |
| `/workspace/chat` | 不可见 | 只读，实验中为空 |
| `/tmp` | 不可见 | tmpfs，仅本次调用 |

## 工作目录与持久性

- `working_directory` 默认为 `/agent/generated`。**强烈建议保留默认**或显式设 `/agent/generated`——探测中见过 `working_directory=/agent/current/generated` 导致 `No such file or directory` 后**整次调用的写入全丢**。
- 命令退出码为 0 时，`/agent/generated` 下的修改被同步回后端；返回值里的 `files_written` 数组标出哪些文件最终进入持久层。
- 观察到一次 exit_code=0 但某个写入没进入 `files_written` 也没持久化（`_sh_run_marker.txt`）；原因未稳定复现。**可靠模式**：结果写完后立刻 `cat` 回读一次，或干脆用 `execute_js` `twin:fs` 做写入。
- `/tmp` 和 `/` 的其余路径永不持久——仅作为本次调用的临时缓存。

## 可用的二进制

工具描述列出并实测可用：bash, coreutils, findutils, sed, awk, grep, jq, qsv, pdftotext, pdfinfo, unzip, zip, tar, gzip, bzip2, xz, file, which, diff, patch, bc, make, pandoc, typst, tesseract, ocrmypdf, imagemagick, docx2txt。

**不可用**：网络（无 curl 到外网——验证 `file` 工具是跨沙箱 HTTP 代理的原因）、包管理器（无 apt/nix-env 可用）。

## 典型最佳实践

1. 读用户上传 → 写处理结果：
   ```bash
   pdftotext /agent/downloads/report.pdf /agent/generated/report.txt
   ```
2. 需要网络：用 `file` 工具把文件落到 `downloads`，再让 `shell` 处理。
3. 关键产物写完立刻自检：
   ```bash
   echo "$DATA" > /agent/generated/out.json && cat /agent/generated/out.json | head -c 200
   ```
4. `/tmp` 可以用作本次调用的 scratch，用完即弃。
