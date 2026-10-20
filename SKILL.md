---
name: mac-tools-gateway
description: 接入 zj-linjie 的 Mac 工具网关（带密钥的公网 MCP 服务），用 MCP 工具控制 Mac、路由 Mac 与小爱音箱的网易云音乐、检索打开 Obsidian 知识库、语音记事、控制 Home Assistant 智能家和 AI 生图。适用于配置网关并按目标设备选择音乐工具。
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

## 工具清单（35 个）

| 工具 | 作用 | 调用要点 |
|---|---|---|
| `open_app` | 打开/聚焦 Mac 应用 | 支持中文名与模糊匹配；没装时会返回候选列表，**不要猜着重试** |
| `open_website` | 默认浏览器打开网站首页 | 收录：斗鱼（douyu.com）、虎牙（huya.com）、B站（bilibili.com），「斗鱼直播」「哔哩哔哩」这类口语别名都收；也可直接传网址（`www.douyu.com` 或 `https://…`，裸域名自动补 https）；没收录的站返回候选列表，**不要猜着重试** |
| `search_obsidian_notes` | 关键词检索 Obsidian 知识库 | 返回笔记标题+路径+摘要 |
| `open_note_in_obsidian` | 打开最匹配的笔记并聚焦 | 配合上面的搜索结果使用 |
| `open_obsidian_search` | 打开 Obsidian 搜索面板 | 让用户自己挑结果时用 |
| `save_voice_note` | 把一段话记进 Obsidian 日记：追加 `- HH:MM 时间戳行` 到 Jggknowledge 仓库 `日记/当天日期.md`，可选附录音嵌入 | text 必填（口述正文）；audio_file 可选，传已存进 收件箱/Ios录音 的 m4a 文件名（如 20260913_175210.m4a），日记里会附 `![[文件名]]` 录音嵌入。机主说"记个日记/记一下/语音记事"时调用；Mac 快捷指令「记录日记 esp」走的就是这个工具 |
| `open_my_github_repositories` | 浏览器打开机主的 GitHub 仓库列表 | — |
| `play_song` | 在 Mac 网易云点歌播放 | `song_name` 必填，提到歌手就传 `artist`（选曲更准）；找不到指定版本如实说明 |
| `play_liked_music` | 在 Mac 顺序播放"我喜欢的音乐"歌单 | 无参数 |
| `music_play_pause` | 暂停/继续 Mac 网易云 | Mac 目标已确定后传 action：`"pause"`/`"play"`；目标不明先澄清，`"auto"` 仅用于明确要切换 Mac 播放状态时 |
| `music_next` / `music_previous` | 在 Mac 网易云切歌 | Mac 目标已确定后使用，返回当前曲目与播放状态 |
| `now_playing` | 查 Mac 网易云正在播放的歌 | 仅查询已确定的 Mac 目标；小爱曲目走 `ha_xiaoai_music(action="status")` |
| `generate_image` | AI 文生图（两步确认制，防误扣费）：只登记请求返回确认码，不扣费 | `prompt` 填具体画面描述；默认 3840x2160；**把结果给用户确认后**用 `confirm_image` 执行；成功返回保存路径/尺寸/耗时/相册地址/**相册访问密码**；用 `GET <网关>/img/<文件名>`（同一密钥）取回图片字节 |
| `edit_image` | AI 图生图（两步确认制）：以参考图为底按描述改图，登记同上 | `image_path` 填那台 Mac 上的图片路径——远端先用 `POST <网关>/upload` 上传手机图（Bearer 密钥，body 为原始图片字节），或用已生成图（如 `xiaozhi-mcp/generated/img_xxx.png`）串"生成→改图"链路 |
| `confirm_image` | 持确认码实际执行待确认的生图/改图 | 只在用户明确确认后调用（这一步才计费）；确认码 2 分钟有效、单次；每日实际生成上限 10 次（IMAGE_DAILY_LIMIT 可调） |
| `list_images` | 列相册现有图（生成+上传），按时间倒序 | "看看相册里有什么图/刚传的图叫什么"先调它；拿到 `generated/<文件名>` 直接当 `edit_image` 的 `image_path` |

**网易云目标路由**：明确设备优先，同一对话沿用最近明确的播放目标；无上下文的新播放默认 Mac。控制或曲目查询目标不明时先问“Mac 还是小爱音箱？”。小爱目标使用 `ha_xiaoai_music`，请求失败只报告失败，不回退到 Mac；小爱上下文的下一首/上一首尚不支持，不调用 Mac 切歌。Mac 的 `play_liked_music` 是顺序歌单，小爱的 `liked` 是最多 5 首的随机连播。

## Home Assistant 工具（18 个 `ha_*`）

`ha_list_devices` / `ha_light` / `ha_switch` / `ha_cover` / `ha_climate` / `ha_media_player` / `ha_xiaoai_music` / `ha_where_is` / `ha_parking` / `ha_treadmill` / `ha_briefing_reader` / `ha_speak_text` / `ha_briefing_reader_stop` / `ha_briefing_reader_status` / `ha_activate_scene` / `ha_run_script` / `ha_get_state` / `ha_call_service`——逐个工具的语义与话术见机主仓库 `xiaozhi-mcp/README.md` 的 homeassistant_tools 一节。控制的是真实家居设备，用户明确要求才调用。

**小爱网易云**：`ha_xiaoai_music(action, song_name="", artist="")` 的 action 限定为 `song` / `liked` / `pause` / `resume` / `status`。`liked` 从喜欢列表随机连播最多 5 首，不是 Mac 的顺序歌单。点歌/喜欢随机提交后等约 5 秒，按 HA 最近 30 秒内且歌名匹配的结果答复播放、被拒原因或失败；结果过期/读不到才答已提交。暂停/继续仍只确认请求已发送；`status` 为约 15 秒采样快照，并附最近 5 分钟内的点歌拒绝原因。

**ha_climate 2026-10-02 升级（格力红外空调）**：action 支持 on（开机）/ off（关机）/ mode（切模式）/ fan（调风速）。
- action=mode 配 `hvac_mode=cool/heat/dry/fan_only/auto`（中文「制冷/制热/除湿/送风/自动」也直接收，off 等于关机）
- action=fan 配 `fan_step=up/down`（这台只有相对升降档，没有低中高，不要传 low/medium/high）
- 只调温度就不填 action，传 `temperature`（摄氏度，超 16–30 自动钳到边界并在回复里说明）；「空调」「格力空调」都能命中实体
- 红外单向下发：读不到室温（ha_get_state 对它会回「读不到室温（红外控制）」），遥控器直改不会同步到 HA
- 示例：「空调制冷」→ action=mode, hvac_mode=cool；「空调调到制热」→ action=mode, hvac_mode=heat；「风速大一点」→ action=fan, fan_step=up；「调到26度」→ temperature=26

**货架开关（2026-10-03 新增，Matter）**：实体 `switch.quectel_matter_product_2`，别名「货架」「货架开关」已锚定——ha_switch 控开关、ha_get_state 查状态。家里另有一个同名 select 辅助实体（「启动时的开机行为」，是开机动作选项不是开关），用别名调用不会误命中，别对它下发控制。

**ha_parking 车位识别（2026-10-04 新增）**：查「家里有没有车位」——看起居室摄像头（`camera.qijushi`，画面朝楼下停车区）判断固定那格车位：空着 / 被占（尽量带车的颜色）/ 看不清（夜间、雨雾天会如实说看不清，不硬猜）。若回复含「看不清车的颜色」，主动向机主提议一句「要我调快照复核吗？」——机主同意后 `ha_parking with_image=true` 落盘快照，再 `GET /img/<回复里的文件名>` 取图目视确认。
- 无参数即可用；只有要自己看图复核时才传 `with_image=true`：会把当前快照落盘 `generated/cam_时间戳.jpg` 并在回复里给文件名，再用 `GET <网关地址去掉/mcp>/img/cam_xxx.jpg`（同一把密钥）取回图片。语音问答用不着传。
- 这是车位专用工具，不是通用摄像头工具：想看摄像头画面或家里其他摄像头，不要路由到它（camera 域不在可见白名单，这是唯一的窄例外）。
- 标定与参考图在 `calibration/parking.json`（机主重标 = 换参考图 + 改 JSON，即时生效，不用重启；结果有 60 秒缓存，改完最多等 1 分钟）。参考帧按时段槽位选取：`day_am`[6,11) / `day`[11,14) / `day_pm`[14,19) / `night`，槽位缺图自动回落 `day`。

**跑步机查询（2026-10-18 新增，Keep KR1 Pro）**：`ha_treadmill` 无参数，一句话报在不在跑 + 速度/已跑距离/时长；数据约 2 分钟没更新会注明。空闲时速度距离时长读不到属正常（无运动中会话），不是故障。**只读工具**：BLE 只有读数没有下发通道，启动/调速控制不了（那在 Keep App）——用户说「帮我开跑步机/调速度」时如实说控制不了，别硬路由。这是跑步机专用查询，查车位走 ha_parking，别混。

**AIHOT 简报与晓晓文字（2026-10-20 新增）**：音频由 Home Assistant 合成，经已配置的 L05C 播放。这不是小智设备自己出声。用户没明确要求不要调用。
- `ha_briefing_reader(action, date="")`：`action` 只能是 `preview` 或 `play`。源固定 `aihot_daily`。`date` 空则按 Asia/Shanghai 今天；传入必须是真实的 `YYYY-MM-DD`。`preview` 只校验稿件，不播放；`play` 把播放交给 HA。返回里的 accepted 只表示 HA 接受了调用，不是播完。
- `ha_speak_text(text)`：晓晓朗读。先去掉首尾空白，再要求 1–1000 个 Unicode 码点；内部换行和标点保留。返回只有 accepted 和 kind，不回显原文，也不表示已经念完。
- `ha_briefing_reader_status()`：查当前观察值，不要把它算成某一次调用已经完成。离开 reading/generating/playing 后，还要看 `last_run.result`：`completed` 才是播完；`mac_offline` 是 Mac 未开机，不能说简报或文字已播完。回复里没有简报全文，也没有用户原文。
- `ha_briefing_reader_stop(kind)`：`kind` 必须是 `briefing` 或 `text`，每次都要传。它只能避免停错类型，不能区分同类型的不同一次调用（没有公开 task_id）。停止 text 会保留已排队的定时简报；停止 briefing 会清掉简报等待队列。
- 状态已是 reading、generating 或 playing 时，新的 play / speak 会失败（busy）。不要改口说已经播完，也不要改用其它工具去取简报稿。

## 生图与图片回传

- **两步确认**：`generate_image`/`edit_image` 首次调用只登记返回确认码（不扣费）；把结果给用户确认后调 `confirm_image(确认码)` 才实际执行。机主说「确认生成」即视为确认。
- **手机图传入**：`POST <网关地址去掉/mcp>/upload`，Bearer 密钥鉴权，请求体为原始图片字节（PNG/JPG/WebP，≤30MB，iPhone HEIC 原图需先转 JPG），如 `curl --data-binary @照片.jpg -H "Authorization: Bearer <密钥>" <网关>/upload`，返回 `{"ok":true,"file":"generated/up_xxx.png"}`——这个 `file` 直接填进 `edit_image` 的 `image_path`。机主手机也可以不开 harness，直接开相册网页点「＋ 上传图片」。
- `generate_image` / `edit_image` 成功后，返回文本里的 `xiaozhi-mcp/generated/img_xxx.png` 是那台 Mac 上的仓库相对路径；远端取图用 `GET <网关地址去掉/mcp>/img/<文件名>`，带同一把密钥（Bearer 或 `?key=`）。返回文本还带 `相册地址`（机主本地公网域名，以工具返回文本为准）。
- **每次生成成功都会轮换「相册密码」**（返回文本里带 `相册访问密码: …` 一行）：那是机主相册网页（地址见返回文本里的 `相册地址`）的最新登录密码，旧密码随即作废；生图失败不轮换。转述给机主即可，不要当自己的凭据用。
- 生图通道偶发瞬时故障（错误文本含 `transient，生成通道抖动`）：这是中转站侧抖动，稍后原样重试通常可过；含 `deterministic` 的失败重试无用。

## 新增工具后调用报 not found

- 网关增删工具后，**已经开着的会话沿用会话开始时的工具清单**（Grok bot 实测；iOS harness 暂未复现）：新工具在工具搜索里可能搜得到，但调用报 `Tool user-mac-gateway-<name> was not found`。这是 harness 平台报的，请求没到网关；网关自己对未知工具报的是 `Unknown tool`。
- **处理**：换一个新开的会话/新任务来调用；在已开会话里重启或重新添加连接器无效；也不要因此改调别的工具替代（例如拿 Mac 音乐工具顶替小爱工具）。如实告诉机主需要新开会话。
- 原因在平台侧：调用前拿会话开始时的旧清单做校验。网关无从配合——无状态 HTTP 没有推送通道，`listChanged=false` 本就不发清单变更通知。

## 行为边界

- 这些工具**控制一台真实的 Mac**（开关应用、出声放歌）：用户明确要求才调用，不要"顺手试试"
- 播放类工具的返回即事实：返回"没在播放/已在播放"就如实转述，这台机器的状态检测已做过防谎报处理
- 密钥仅限一台 harness 使用；换机或怀疑泄露，让机主 `revoke-key` 后重发，**不要共享密钥**
- 网关密钥不止你这一把：keys 里若见 `shortcuts-phone`，那是机主本机快捷指令（「问灵犀 esp」）专用，与你的密钥隔离——不是你的密钥，别动它
