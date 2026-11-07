# 给 AI 的配置手册：用户说「请帮我验证 B站动态记忆桥 的功能是否可用，并配置好」时，照这份做

这份文件是给**接手配置的 AI**看的，不是给普通用户看的。目标只有一个：
让 QQ 里的机器人（AstrBot）达到「能收 B站 推荐私信、能解析用户分享的视频、能回答最近刷到什么」的状态。

整份流程约 4 步，改配置 → 重启 → 验证 → 收尾。**别跳步**，尤其别跳过「重启」。

---

## 0. 先定位这几处路径

| 东西 | 位置 |
| --- | --- |
| AstrBot 根目录 | 含 `data/plugins` 的那个目录（用运行中进程的 cwd 确认） |
| 本插件 | `<AstrBot>/data/plugins/astrbot_plugin_bili_bridge/` |
| 插件配置 | `<AstrBot>/data/config/astrbot_plugin_bili_bridge_config.json` |
| 同步状态 | `<AstrBot>/data/plugin_data/astrbot_plugin_bili_bridge/state.json` |
| 私信状态 | 同目录 `dm-state.json`（没有＝私信转发还没跑过） |
| 挑选结果 | `~/.dsh/bili-duty/digest/YYYY-MM.md`（DSH 侧 bili-scroll 写的） |

> AstrBot 一般是 Python 进程；改完配置要**重启它**（`python main.py` 重新拉），
> 因为配置只在 import/initialize 时读一次。

---

## 1. 三个前置依赖自检（缺哪个就让用户去装）

```powershell
# ① bili-cli：必须能返回 mid，说明已扫码登录
<python> -m pip show bili-agent
bili me show

# ② MemoryCompanion：目录在、日志有 loaded
Test-Path <AstrBot>/data/plugins/astrbot_plugin_memory_companion

# ③ bili-scroll：digest 目录里至少有一个当月文件
Get-ChildItem ~/.dsh/bili-duty/digest
```

缺 ①：`pip install bili-agent`，然后 `bili auth login` 让用户扫码。
缺 ②：装 `astrbot_plugin_memory_companion`（应用市场搜「我会牢牢记住你」）。
缺 ③：装 DSH 的 `dsh-bili-scroll`，跑一轮让它写出 digest。

---

## 1.5 问用户：要不要装「语音转写」（可选，别自作主张装）

这一步**必须先问一句**，而且要把「它到底是什么、为什么要装东西」说清楚。
照下面这段问（可按语气改写，但要点不能少）：

> 插件默认只读 B站 自己的字幕（CC 字幕 / AI 字幕）。
> 额外还有一个**本地语音转文字**的兜底：**当这条视频没有 B站 AI 字幕时**，
> 用你电脑上的语音识别把视频里说的话转成文字，再交给模型去讲。
> 需要在本地部署 **ffmpeg + 一个语音识别模型**（几百 MB），第一次转写会慢十几秒。
> 要装吗？

要点复述一遍，问的时候别漏：

- **触发条件**：只在「没有 B站 AI 字幕」时才用，有字幕仍然优先用字幕（更准、更快）
- **它做什么**：本地语音转文字，把说的话转成文本，不是识图 OCR、也不是什么云端服务
- **要装什么**：本机要部署 **ffmpeg**（抽音频）+ **ASR 模型**（sherpa-onnx + 中文模型）
- **代价**：几百 MB 空间；第一次转写十几秒到一两分钟（按时长）；识别结果可能有错字

**用户说不要**：跳过，保持 `transcribe_fallback: false`（默认就是关）。
插件取不到字幕时会退回 B站 官方 AI 总结，不会报错。

**用户说要**：

```powershell
bili asr setup local --device auto --model qwen3-asr --github-proxy https://gh-proxy.com/ --hf-mirror
bili asr status        # ready 必须为 true，ffmpeg / sherpa_onnx / vad_model 三项都为 true
```

> 显卡小（<4GB）或想省事：把 `qwen3-asr` 换成 `sensevoice --device cpu`——体积小、CPU 就能跑、
> 中文够用，代价是没标点、准确率略低。qwen3-asr 要装 torch（2GB+），国内网络容易失败。

装完把配置里的开关打开（并在 README 里说明这是可选依赖）：

```json
{
  "transcribe_fallback": true,
  "transcribe_max_minutes": 8,
  "transcribe_timeout": 180
}
```

验证：随便挑一条**没有字幕**的视频（先用 `bili video subtitle <BV>` 确认返回 `没有可用字幕`），
发到 QQ 里问「这条讲了什么」，日志应出现转写，回答里会写「这条没有 CC 字幕，以下是本机语音转写的前 N 分钟」。

> 装之前先确认 `bili asr status` 的 `ready`；没就绪时插件只会打一行日志然后退回官方总结，
> 不会报错，但也不会给你转写——所以别把「装了」当成「好了」，要看日志。

---

## 2. 改配置（下面这些键，一项都别漏）

先备份，再改（`utf-8-sig` 读写，别把中文写坏）：

```json
{
  "bili_executable": "<bili 的绝对路径，例如 C:\\path\\to\\Scripts\\bili.exe>",
  "owner_uids": "<用户 QQ 号>",
  "memory_user_ids": "<同上>",
  "sync_enabled": true,
  "sync_interval": 120,

  "dm_forward_enabled": true,
  "dm_owner_uid": "<收信方 B站 UID>",
  "dm_direction": "me",
  "dm_only_video": true,
  "dm_poll_interval": 120,

  "scroll_enabled": true,
  "scroll_owner_only": true
}
```

逐个为什么：

- **`bili_executable` 用绝对路径**：AstrBot 进程的 PATH 未必有 Scripts 目录，写 `bili` 会偶发「用不了」。
- **`owner_uids` 必填**：空着等于谁都能触发解析。
- **`dm_forward_enabled` 默认是 false**，不打开就永远收不到私信——这是最常见的「装完没反应」。
- **`dm_owner_uid` 填「收信方」**（被推荐的那个人），不是登录号自己。
- **`dm_direction` 填 `me`**：刷视频工具用「登录 bili-cli 的那个号」给主人发推荐，消息在会话里是 `from: me`。
  拿不准就让用户跑一次 `bili dm read <dm_owner_uid> --limit 5`，看那批要转的消息是 `me` 还是 `them`。
- **`dm_target_session` 可以留空**：插件会按平台真实 id 自动拼成 `<平台id>:FriendMessage:<第一个 owner_uid>`。
  填了但对不上平台 id 也会自动纠正，不用纠结。

改完 **重启 AstrBot**。

---

## 3. 验证（按这个顺序，每步都要过）

1. **启动日志**里应该有这几行（搜关键字）：
   - `[BiliBridge] 私信转发已开：...`
   - `[BiliBridge] 已启动（同步=True...）`
   - `[BiliBridge] 已同步 B站 近况（N 字，来源 digest:...）`
   - `[BiliBridge] 私信转发：已记录基线 seqno=...（这批 me=X 条 / them=Y 条，当前只转 me）`

2. **让用户在 QQ 私聊发 `/bili_doctor`**，逐项要 ✅：
   - bili-cli 可用（带登录昵称）
   - 挑选结果可读：N 行可用
   - MemoryCompanion 在线
   - 触发白名单是你的 QQ
   - 私信转发：开，方向 me
   - 上次同步：成功/跳过 + 原因

3. **端到端测一次私信转发**（不用等用户）：
   ```powershell
   bili dm send <dm_owner_uid> "【转发测试】链路验证 https://www.bilibili.com/video/BV1xxxxxxxxx"
   ```
   等一个轮询周期（`dm_poll_interval`），日志出现 `私信转发：1 条私信 → QQ（共 1 次）` 即通过；
   用户 QQ 上会收到以「📬 B站 私信」开头的一条。

4. **测「最近刷到什么」**：让用户在 QQ 问「今天刷到什么」，应返回带编号的清单（插件自己答，不经过模型）。

5. **测「第一个视频讲了什么」**：应解析到最近推送清单里的第 1 条（日志有 `第 1 个 → BV...（来源 push）`），
   而不是记忆块里另一份排列的第 1 条。

6. **测「刷视频」口令**（v0.5.0 起）：让用户在 QQ 私聊里直接说一句 **「刷视频」**（不带 `/`）。应该是：
   - 她**用自己的口气**回一句（插件不塞台词：后台说明是临时注入给模型的，所以可能是「又来？」也可能是「行吧去看看」，都正常）；**也不该出现「还要等 170 秒」这种数字**——说明里明令禁止报时间，出现了就是模型没听话
   - 日志出现 `[BiliBridge] 口令触发刷一轮（...，started）` 和 `已把「刷一轮」的说明挂到本轮请求（started）`；
   - 大约 1~2 分钟后，**只有一条**结果消息：B站 那条私信被转发过来的「📬 B站 私信」（插件默认不再自己发一份）。
   `/bili_doctor` 里还会多一行 **「「刷视频」口令可用：DSH 网关 127.0.0.1:xxxxx」** —— 那行 ❌ 就说明
   DSH 没开，或者刷B站插件没在 DSH 里装好（口令走的就是它自己的接口）。

   如果刚跑过一轮（`scroll_min_gap` 三分钟内），正确表现是：她回一句嫌你急的话，
   日志里 `status=gap`、**没有** `刷一轮已开跑` —— 那就是挡住了，不是 bug。
   而且**三分钟过了也不会自己补跑**：必须再喊一次才会跑（老大定的规矩）。

---

## 4. 已知坑（照着避）

| 现象 | 原因 / 处理 |
| --- | --- |
| 有 digest 文件但不同步、日志干净 | 0.4.0 及更早的正则只认行首日期；0.4.1 起容忍 `- ` / `* ` / `1. ` 前缀。老版本手动改 `DIGEST_LINE_RE` |
| 私信一条都不转 | `dm_forward_enabled` 没开 / `dm_owner_uid` 空 / `dm_direction` 填反（大多数是填了 `them`） |
| 第一次上线没补发历史 | 设计如此：首次只记基线。要补发就临时删 `dm-state.json` 的 `last_seqno`（会重放最近 20 条） |
| 私信收到了但没转 | `dm_only_video=true` 时只转带 BV/视频链接的消息，寒暄会被跳过 |
| 改完配置没反应 | 没重启 AstrBot |
| 「第几个视频」答错 | 0.4.1 起按「最近推送顺序」解析；确认 `dm-state.json` 里有 `pushBvs` |
| `bili dm read` 报错 | bili-cli 没登录，或 UID 填的是登录号自己 |
| 说「刷视频」回「刷不了：连不上 DSH…」 | DSH 没开（桌面版/`dsh web` 都算），或者它换了端口。用 `dsh_gateway` 手填 `127.0.0.1:<端口>` 兜底 |
| 说「刷视频」能跑，但回报「没挑到新的」 | 多半是 DSH 那一轮自己没跑成。看 `~/.dsh/bili-duty/duty.log` 尾部 |
| ⚠️ 刷B站那一轮「工具没挂上」（`unknown tool`、`Python was not found`） | **`~/.dsh/bili-duty/config.json` 被写成了带 BOM 的 UTF-8** ⇒ 插件里 `JSON.parse` 直接抛、整份配置退回默认值 ⇒ `pythonPath` 变回 `python` ⇒ B站 MCP 起不来。把文件头三个字节 `EF BB BF` 去掉（存成「UTF-8 无 BOM」）就好。2026-11-05 真机上就是这么坏的 |
| 说「刷视频」回「刚刷过，再等 N 秒」 | 是我们自己那道闸（`scroll_min_gap`，默认 **180 秒**，从上一轮**开跑**算）。等它就对了，别当成 bug；**这三分钟里再喊也不会跑** |
| 说「刷视频」回「这会儿在冷却，还得 N 分钟」 | 冷却没被绕过（`scroll_ignore_cooldown=false`，或者读不到刷B站的原冷却值时的保守处理）。这一轮就不跑；想立刻跑就打开那个开关，并确认 `GET /api/bili-duty/config` 能读到 `manualCooldownMin` |
| 说「刷视频」回「上一轮还在跑」 | 有一轮真在跑（或另一个 DSH 宿主占着运行锁）。这一轮不跑，等他收尾再喊 |
| 被拒之后过了一会儿**自己又跑了一轮** | 设计上不该发生：2026-11-05 起所有拒绝路径都不留定时器（以前 `busy`/`queued` 会 30 秒后自己补跑，已删）。真遇到就看 `duty.log` 那条「刷B站开跑」是谁触发的（`manual` = 人喊的；`window:xxx` = 刷B站自己的定时窗口） |
| 一轮刷完收到**两遍**同样的视频 | 以前会：插件那条「刷完了」+ B站 私信被转发的「📬 B站 私信」（晚几十秒）。v0.5.0 起按 BV 去重，半小时内同一批不再重复推 |
| ⚠️ 改完 `main.py`，面板上点「重启」却没生效（日志还是旧版本号、旧行号） | **面板那个重启是在同一个进程里重来**，`sys.modules` 里还是旧模块 ⇒ 改过的代码不会被重新读（2026-11-05 实测：16:41 重启后日志里仍是 0.4.2、`bili_scroll_run` 也没了）。要让改动能生效，必须**把整个 AstrBot 进程换掉**（杀进程再起，或跑一次 `QQBot-Launcher.ps1`），必要时顺手删掉插件目录下的 `__pycache__` |
| 日报里**这一轮刚写的内容被删了** | dsh-bili-scroll ≤2.3.4 的 `writeRoundNotesToReport` 会从 `## 投递` 处整段截断，把模型追加在它后面的内容一起删掉。本插件不受影响（日报没有新条目时会退回读 `digest` 归档），要修就动那边：只摘掉旧的那一段，别从 `## 投递` 往后全切 |

---

## 5. 收尾：告诉用户现在能干什么

1. B站 推的视频私信会自动转到 QQ（想关就把 `dm_forward_enabled` 关掉）
2. 直接把 B站 视频链接/卡片发给她，她会拉字幕讲内容
3. 问她「今天刷到什么」能拿到最近清单；问「第 N 个讲了什么」会讲对应那条
4. 说一句「刷视频」/「刷B站」/「看视频」，她就去刷一轮，跑完把挑到的几条发回来


## 清单的触发规则（0.6.0 起）

- 只问时间 / 问「刷没刷」：插件直接回一句「最近一轮在 X」，延迟 1~2 秒（`direct_reply_delay` / `direct_reply_delay_max`）
- 问「刷到了什么」且句子不超过 `recent_list_max_chars`（默认 12 字）：插件直接回清单，延迟 2~5 秒
- 句子太长（夹着别的问题）：不直接发清单，交给模型结合记忆块说
- `list_repeat_window`（默认 600 秒）内再问：不重复列，只注入一句「别重复列」的说明，由模型自己回
