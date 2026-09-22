---
name: mac-tools-gateway
description: 接入 zj-linjie 的 Mac 工具网关（带密钥的公网 MCP 服务），用 MCP 工具控制那台 Mac：开应用、点网易云音乐、检索打开 Obsidian 知识库。适用场景是拿到网关 URL 和密钥后完成 MCP 客户端配置并验证连通。
---

# Mac 工具网关接入

你（harness/Agent）拿到了两个输入：

- **网关 URL**：以机主给的为准，形如 `https://<域名>:<端口>/mcp` 或 `http://<IP>:<端口>/mcp`（未配证书的降级模式是 http，机主知情选用）
- **接入密钥**：形如 `xm_xxxxx...` 的字符串

目标：把它配成你的一个 MCP 服务，验证连通，然后按下面的工具语义使用。

## 接入步骤

1. **验活**（无鉴权）：`curl -s <网关地址去掉/mcp>/health` 应返回 `{"ok":true,...}`。不通就停下来向机主反馈，不要瞎猜地址。
2. **写 MCP 配置**：优先 Streamable HTTP；客户端不支持再用 SSE；两个都支持不了才用 `?key=` 查询参数兜底。具体条目写法见 [references/mcp-config-examples.md](references/mcp-config-examples.md)。要点：
   - 密钥放请求头 `Authorization: Bearer <密钥>`；配置不支持自定义头时用 `?key=<密钥>` 拼在 URL 上
   - 密钥是机密：不要写进会提交的文件、不要在对话里复述全文
3. **验证连通**：连接后调用一次 `now_playing`，能返回那台 Mac 的播放状态即接入成功。

## 工具清单（11 个）

| 工具 | 作用 | 调用要点 |
|---|---|---|
| `open_app` | 打开/聚焦 Mac 应用 | 支持中文名与模糊匹配；没装时会返回候选列表，**不要猜着重试** |
| `search_obsidian_notes` | 关键词检索 Obsidian 知识库 | 返回笔记标题+路径+摘要 |
| `open_note_in_obsidian` | 打开最匹配的笔记并聚焦 | 配合上面的搜索结果使用 |
| `open_obsidian_search` | 打开 Obsidian 搜索面板 | 让用户自己挑结果时用 |
| `open_my_github_repositories` | 浏览器打开机主的 GitHub 仓库列表 | — |
| `play_song` | 点歌播放 | `song_name` 必填，提到歌手就传 `artist`（选曲更准）；周杰伦等版权不在网易云，找不到原唱要如实说 |
| `play_liked_music` | 播放"我喜欢的音乐"歌单 | 无参数 |
| `music_play_pause` | 暂停/继续 | **必须传 action**：`"pause"`/`"play"`，不确定才用默认 `"auto"`；返回话术是真实状态，不要脑补 |
| `music_next` / `music_previous` | 切歌 | 返回里带当前曲目与播放状态 |
| `now_playing` | 查正在播放的歌 | 用户问"放的什么歌"时先调它，别猜 |

## 行为边界

- 这些工具**控制一台真实的 Mac**（开关应用、出声放歌）：用户明确要求才调用，不要"顺手试试"
- 播放类工具的返回即事实：返回"没在播放/已在播放"就如实转述，这台机器的状态检测已做过防谎报处理
- 密钥仅限一台 harness 使用；换机或怀疑泄露，让机主 `revoke-key` 后重发，**不要共享密钥**
