# hermes-Digital-Human · 设计冻结文档

> 本文档基于 2026-05-24 一次结构化设计访谈产出，共 16 轮决策、约 60 个子分支。
> 后续任何与本文档冲突的实现都视为偏差，需要先更新本文档再写代码。
> 修订原则：MVP 阶段冻结、v2 阶段开放。每次修订附 Decision Log 条目。

---

## 0. 项目愿景一句话

**一个面向开发者群体（α 用户）的开源 Android 数字人客户端，让用户在自己的服务器上跑 Hermes Agent，通过 3D VRM 形象 + 语音 + 表情动作，把 Hermes 的智能体能力包装成"和数字人对话"的体验。**

不是托管服务（用户自带服务器），不是商业产品（MIT 开源），不是聊天 App 套壳（D 档 3D 形象 + 情绪动作驱动是核心差异点）。

---

## 1. 系统全景

```
┌──────────────────────────── Android App ─────────────────────────────┐
│  UnityPlayer (UniVRM 3D 形象, MToon shader, 60fps)                    │
│  Compose UI 半透明覆盖：聊天浮层 / 抽屉 / 设置 / 配对                  │
│  Android SpeechRecognizer (默认 STT 路径)                             │
│  Silero VAD + Opus 编码 (BYOK STT 路径才用)                           │
│  AudioTrack 播放 + getTimestamp 锁播放时钟 (jitter 固定 120ms,v2 自适应)│
│  BlendShape 时间轴驱动器 + Animator 状态机                            │
│  Room 数据库 (最近 500 条/会话只读缓存) + VRM 缓存 (LRU 200MB)        │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ wss://  (单连接、多路复用、二进制帧)
                           │ 协议: HDH/1.0 (SemVer 协商, feature 枚举)
                           │ 鉴权: Authorization: Bearer device_credential
                           ▼
        ┌──────── Caddy / Cloudflare Tunnel (TLS 终结) ───────┐
        └──────────────────────┬──────────────────────────────┘
                               ▼
┌────────────────── hermes-voice-gateway (新组件) ─────────────────────┐
│  WS Server (aiohttp, Python 3.11) — 锁定 aiohttp                     │
│  ├─ 配对端点 POST /v1/pair (BIP39 pairing_code → device_credential)   │
│  ├─ device_credential 鉴权 + 限流 + 熔断中间件                         │
│  ├─ 协议握手 (hello 帧 / feature 枚举 / min_app_version)              │
│  ├─ 流式 XML 解析器 (合法 XML + 容错降级)                              │
│  ├─ Turn 租约管理器 (单会话单活跃 turn)                                │
│  ├─ STT Router                                                       │
│  │    ├─ default: 不存在（默认走 App 端 SpeechRecognizer）            │
│  │    └─ BYOK proxy: Azure / MiniMax adapter (服务端代理)             │
│  ├─ LLM Bridge → Hermes /v1/chat/completions (SSE)                   │
│  ├─ TTS Router                                                       │
│  │    ├─ default: Edge-TTS (CPU, 免费, 流式)                          │
│  │    └─ BYOK proxy: Azure Neural / MiniMax speech-02-hd             │
│  ├─ Viseme Aligner (按 lang × provider 分支)                          │
│  │    ├─ Edge-TTS+zh: pypinyin                                       │
│  │    ├─ Edge-TTS+en: g2p_en                                         │
│  │    ├─ Edge-TTS+ja: pyopenjtalk                                    │
│  │    └─ Azure: 原生 viseme 事件                                      │
│  ├─ Frame Muxer (统一 envelope + 时间轴)                              │
│  └─ SQLite (/data, WAL): 会话/消息/device/byok/pairing 表             │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ HTTP http://hermes:8642 (compose 内部网络)
                           ▼
                  Hermes Agent (独立只读实例容器)
                  - 工具白名单 (yaml allow-list): chat, web_search, memory, session_search
                  - 工具禁用: terminal, file, cronjob, skills
                  - 持久化: hermes_data volume
                  - 上游 LLM provider 由用户配置
                           │
                           ▼
                远程/本地 LLM (OpenRouter / Claude / GPT / 本地模型)
```

---

## 2. 决策快照（Q1~Q16 全部锁定项）

| 维度 | 锁定决策 | 备注 |
|---|---|---|
| **数字人档位** | D 档：3D + 全身动作 + 表情 + 唇形 | 工作量最大、体验最完整 |
| **Android 渲染栈** | Unity 2022 LTS as Library + UniVRM | VRM 1.0 标准 |
| **目标机型** | 骁龙 8 系（旗舰） | 渲染预算 100k tri / 4096 纹理上限 |
| **服务端组件** | 新增 hermes-voice-gateway（Python） | 不修改 Hermes 上游 |
| **TTS 默认** | Edge-TTS (CPU, 免费) | viseme 由服务端 pypinyin 对齐补齐 |
| **TTS BYOK** | Azure Neural / MiniMax speech-02-hd | 经服务端代理，key 不在客户端 |
| **STT 默认** | Android SpeechRecognizer（系统内置，免费） | 上行只传文本，无音频 |
| **STT BYOK** | Azure / MiniMax | 上行 Opus，服务端代理 |
| **GPU 6GB** | 闲置预留 | v2 可能接本地 LLM 或动作生成模型 |
| **鉴权** | 公网 + BIP39 配对码 → ed25519 device_credential + wss:// 强制 + Hermes 工具白名单 | D002 修订：从共享 token 升级为设备级凭证 |
| **协议** | WebSocket + 二进制帧（依赖 WS RFC 6455 长度）+ JSON 文本帧 | 调试简单；删除 D001 冗余 4B length 头 |
| **部署模式** | D 策略：用户自装一服务器一用户 | 不做多租户托管 |
| **目标用户** | α（开发者） | β/γ 不在 v1 支持范围 |
| **情绪/动作生成** | LLM 内联合法 XML fragment（`<emo name="happy" intensity="0.6">...</emo><gesture name="nod"/>`） | Hermes 加 digital-human-tagger skill；GW 解析时合成 `<stream>` root |
| **情绪词汇** | 6 个：neutral/happy/angry/sad/surprised/relaxed | 对齐 VRM 1.0 标准 BlendShape |
| **动作词汇** | 9 个：idle/nod/shake_head/thinking/pointing/wave/shrug/lean_forward/lean_back | Mixamo 通用动画 |
| **动画库** | Mixamo + UniVRM humanoid retargeting | D002 修正：Mixamo Additional Terms 禁止 redistribute，原始动画**不入仓库**，用户用 `tools/fetch_mixamo.sh` 本地下载；v2 评估 CC0 替代库 |
| **交互模态** | Siri 风格点按麦克风 + 自动检测说完 | 主流数字人 App 模态 |
| **打断** | ②简单按钮打断（v1）；③真·语音 barge-in 留 v2 | AEC 是深坑 |
| **摄像头/视觉** | v1 不做；协议预留 video_in 帧类型 | v2 再做 |
| **模型分发** | 三层回退：本地导入 → 服务器 URL → APK 默认 | 灵活性最大 |
| **默认模型来源** | VRoid Studio 自生成 α 二次元少女 | 法律最干净、零成本 |
| **会话粒度** | 多会话用户管理（类 ChatGPT） | 抽屉式 UI |
| **历史存储** | 服务端真源 + App Room 读缓存 | 不双向同步 |
| **多设备** | 同 device_credential 自动同步同一份对话 | D002 修正：Turn 租约单活跃写入（原子 CAS + fencing token），不再"最后写入胜出"；conv_switch 不广播 |
| **消息编辑/删除** | 不支持 | 简化设计 |
| **主界面布局** | 全屏 Unity + 底部聊天浮层 | 形象为主、文字为辅 |
| **首启流程** | 极简两屏 + 二维码扫码配对 | BYOK 不在首启 |
| **应用语言** | 中 / 英 / 日 | 覆盖 VRM/VTuber 文化圈 |
| **协议** | WebSocket + 二进制帧（依赖 WS RFC 6455 长度）+ JSON 文本帧 + 方向性 envelope | D002 修订 |
| **TTS 流式策略** | 按句末标点切片 | 平衡首音延迟和韵律 |
| **延迟预算** | Server-side P50<600ms / P95<1200ms；Client-side P95 待 v0.3 真机基准 | D002 修正：D001 写的端到端 < 1.5s 不可达，拆分 server/client + 待真机验证 |
| **重连** | 会话级幂等；不重传半截 TTS | 简化设计 |
| **背压** | sendBuffer > 256KB 暂停，< 64KB 恢复 | 移动网络鲁棒 |
| **消息 ID** | 服务端生成 UUIDv7 | 时序友好 |
| **部署** | Docker Compose（Caddy 或 Cloudflare Tunnel 二选一） | ghcr.io 镜像 |
| **工具裁剪** | 独立只读 Hermes 实例 | 无需改上游 |
| **协议演进** | SemVer + hello 握手 + feature flag | 版本协商 |
| **版本号** | App / Gateway / Protocol 三者独立 SemVer | 解耦发版 |
| **镜像 tag** | latest / stable / 1 / 1.2 / 1.2.3 多档 | README 默认 :stable |
| **可观测性** | structlog JSON + /metrics Prometheus + 本地 crash log | 不接 Sentry/Firebase |
| **遥测** | 完全不做 | 隐私优先 |
| **License** | MIT（含 voice-gateway / Android / Unity 工程） | VRM 模型版权独立 |
| **仓库结构** | 单一 monorepo | 单人开发 |

---

## 3. WebSocket 协议规范（HDH/1.0）

### 3.1 连接建立

```
URL:     wss://{host}/v1/ws?lang={zh|en|ja}
Headers (强制): Authorization: Bearer {device_credential}
```

**鉴权策略**（[C1] 修订）：
- token 一律走 `Authorization` header，**禁止**放在 URL query（避免泄漏到反向代理 / Cloudflare / 浏览器历史 / 截图）
- 首次配对：服务端首启生成 5 分钟有效的一次性 `pairing_code` (BIP39 6 词)，App 用 pairing_code 调 `POST /v1/pair` 换取设备级长期 `device_credential` (绑定到 App 端生成的 device pubkey，服务端持久化映射)
- `device_credential` 可被 `gateway-admin device revoke <id>` 撤销
- 服务端永不接受跨设备复用同一 credential

token 鉴权失败：返回 HTTP 401，不升级 WS。  
token 通过：升级 WS，等待双方 hello 帧。

### 3.2 hello 握手（首帧，强制）

```
APP  → GW: 0x00 hello
{
  "app_version": "1.2.3",
  "protocol_supported": ["HDH/1.0", "HDH/1.1"],
  "user_lang": "zh",
  "device_id": "服务端首次配对时签发的持久 ID（非客户端自报）",
  "request_id": "uuid-v7 (本次握手幂等键)",
  "byok": {
    "tts": {"provider": "azure", "region": "eastasia"},     // 不含 key
    "stt": {"provider": "minimax"}
  }
}

GW   → APP: 0x00 hello_ack
{
  "gateway_version": "0.5.0",
  "protocol_selected": "HDH/1.0",
  "min_app_required": "1.0.0",
  "server_features": [
    {"name": "emotion_tag", "version": 1},
    {"name": "action_tag", "version": 1},
    {"name": "viseme_arkit", "version": 1}
  ],
  "default_avatar_url": "https://.../avatar.vrm",
  "default_avatar_sha256": "0123abcd...",
  "byok_upload_credential": "短效 JWT, exp 300s, aud=byok_key_set",
  "active_session_lease_seconds": 3600
}
```

不兼容时 GW 发 0x90 error + close。

**device_id 由服务端在配对时签发并绑定** — [C11] 修订。客户端自报的 UUID 不能当授权边界。`byok_upload_credential` 是 5 分钟有效的一次性凭证，仅用于 0x09 / 0x0A 帧；防止 device_credential 泄漏后被攻击者批量上传/覆盖 BYOK key。

**server_features 用枚举 schema + version** — [W9] 修订。每个能力是 `{name, version}` 结构，App 可基于 (name, version) 决定 UI 行为；弃用窗口为 1 个 minor 周期。

### 3.3 帧类型表（最终）

```
─── 上行（APP → GW）───
0x00 hello              JSON   首帧握手
0x01 text_in            JSON   {"conv_id", "turn_id", "request_id", "text"}
0x02 audio_in_start     JSON   {"conv_id", "turn_id", "request_id", "codec": "opus", "sr": 16000}
0x03 audio_in_chunk     BIN    [opus packet, ~20ms, <= 8KB]
0x04 audio_in_end       JSON   {"turn_id"}
0x05 interrupt          JSON   {"conv_id", "turn_id", "request_id"}
0x06 conv_switch        JSON   {"conv_id", "request_id"}
0x07 conv_create        JSON   {"title", "request_id"}
0x08 history_req        JSON   {"conv_id", "before": "msg_id", "limit": <=50, "request_id"}
0x09 byok_key_set       JSON   {"provider", "key", "region", "upload_credential", "request_id"}
0x0A byok_test          JSON   {"provider", "request_id"}  ← [W11] 测试 BYOK 连通性
0x10~0x12 (预留 video_in)
0xF0 pong               无 payload

─── 下行（GW → APP）───
0x00 hello_ack          JSON   握手响应
0x80 token              JSON   {"conv_id", "msg_id", "turn_id", "delta", "done"}
0x81 emotion            JSON   {"msg_id", "name", "intensity": 0~1, "ts_ms"}
0x82 action             JSON   {"msg_id", "name", "duration_ms", "ts_ms"}  ← [I6] 加 duration
0x83 audio_out_start    JSON   {"msg_id", "codec": "opus", "sr": 24000}
0x84 audio_out_chunk    BIN    [opus packet]
0x85 audio_out_end      JSON   {"msg_id"}
0x86 viseme             JSON   {"msg_id", "phoneme", "weight", "ts_ms"}
0x87 msg_complete       JSON   {"msg_id", "full_text"}
0x88 history_page       JSON   {"conv_id", "messages": [...], "has_more"}
0x89 conv_list          JSON   {"conversations": [...]}
0x8A conv_changed       JSON   {"event": "created|switched|deleted", "conv_id"}  ← 广播会话变更（多设备同步）
0x8B turn_state         JSON   {"conv_id", "turn_id", "state": "active|completed|interrupted|preempted"}
0x90 error              JSON   {"code", "message", "fatal", "request_id"}
0x91 byok_test_result   JSON   {"provider", "ok", "latency_ms", "error?", "request_id"}
0xF0 ping               无 payload
```

### 3.3.1 帧封装格式（[C9 / WR3 D002] 修订：依赖 WS RFC 6455 长度，删除冗余）

WebSocket RFC 6455 自身已提供完整帧封装（每条 message 含 payload_length + TEXT/BINARY 区分 + 自动分片）。**不再加自定义 length 字段**避免冗余和"WS 解码对但 HDH 解码错"的难调试 bug。

```
┌─────────┬─────────────────────────────────────────────┐
│ 1B type │ payload (长度由 WebSocket frame 自身提供)    │
└─────────┴─────────────────────────────────────────────┘
```

- `type`：1 字节，参考 §3.3 帧类型表
- `payload`：JSON 文本帧 → UTF-8 编码字节；二进制帧 → 原始字节
- payload 字节数 = `len(WS_message.data) - 1`

**最大尺寸约束**（由 WebSocket 层执行）：
```python
# aiohttp WSResponse
WebSocketResponse(max_msg_size=1_048_576)  # 1 MB 全局上限
# 超限由 WS 层 close 1009 (Message Too Big)，不需要应用层判断
```

| 帧类型 | 单帧最大 payload | 应用层校验位置 |
|---|---|---|
| `audio_in_chunk` / `audio_out_chunk` | 8 KB | 接收时校验，超限发 0x90 + close |
| `history_page` | 单页 ≤ 50 条 messages | 发送时强制，分页拉取 |
| `byok_key_set` | 16 KB | 接收时校验 |
| 其他 JSON 帧 | 64 KB | 接收时校验 |
| 整体连接默认值 | 1 MB | WS 层 close 1009 |

**未知帧类型处理**：
- 上行未知（0x00-0x7F）→ GW 立即 close + 0x90 fatal
- 下行未知（0x80-0xEF）→ App 忽略 + warn 日志（允许 minor 升级新加下行帧）

**方向性 envelope**（[CR4 D002] 修订：拆分必含字段）：

| 字段 | 哪类帧必含 | 含义 |
|---|---|---|
| `v` | 所有 JSON 帧 | 协议版本，如 `"HDH/1.0"` |
| `request_id` | 仅"请求型"上行帧（text_in / audio_in_start / interrupt / conv_switch / conv_create / history_req / byok_key_set / byok_test）+ 对应下行响应 | UUIDv7，幂等键 |
| `ts_server_ms` | 仅下行帧（GW 本地 monotonic 时间）；上行帧不含 | 仅日志/排障，**不参与播放调度** |
| (其他) | — | — |

**特殊帧豁免**：
- `0xF0 ping` / `pong` → 无 payload，envelope 不适用
- `0x00 hello` → 含 `request_id` 但无 `ts_server_ms`（GW 尚未确认协议）
- 二进制帧 `0x03 audio_in_chunk` / `0x84 audio_out_chunk` → 不含 JSON envelope；归属由前序 `*_start` 帧的 `turn_id` 隐式继承；GW 必须丢弃任何在 `audio_in_start` 之前到达的 `audio_in_chunk`
- 服务端主动广播 `0x8A conv_changed` / `0x8B turn_state` → 含 `v` + `ts_server_ms`，无 `request_id`

### 3.4 时间戳与播放时钟（[C5] 修订）

所有下行帧中的 `ts_ms` **相对于该 msg_id 的首个 audio sample 实际从扬声器播出的瞬间**为零点。

**为何不能用接收时刻**：Android `AudioTrack` 从 `write()` 到喇叭实际出声有 80-250ms 硬件延迟，蓝牙耳机 +150-400ms（SBC/AAC 编码）。用接收时刻当零点 = 口型永远早于声音，超过 80ms 人眼即可察觉。

**实现要求**（Android 端）：
```kotlin
// 用 AudioTrack 提供的硬件 timestamp 建立 frame → wall-clock 映射
val ts = AudioTimestamp()
audioTrack.getTimestamp(ts)
// ts.framePosition = 当前已经播出的 sample 序号
// ts.nanoTime      = 上述 sample 播出时的 CLOCK_MONOTONIC 纳秒
// 用此映射将 viseme/emotion/action 帧的 ts_ms 调度到正确的 nanoTime
```

- 音频路由切换（蓝牙↔扬声器↔听筒）时必须重算偏移
- viseme 调度以 audio frame 位置为时基，**绝不用 wall clock**
- App 收到 audio_out_start 时记录 `t_recv`，第一个 audio chunk 提交 AudioTrack 后用 getTimestamp 推算"首 sample 播出时刻"`t_play0`，所有后续帧的 `ts_ms` 用 `t_play0 + ts_ms` 调度

**getTimestamp 失效兜底**（[CR7 / XR1 D003] 修订：放弃 hidden API，用静态路由表）：

`AudioManager.getOutputLatency()` 是 `@hide + @UnsupportedAppUsage` API（自 API 1 起从未进 public SDK；Android 11+ greylist→blacklist，反射调用 throws `NoSuchMethodError`；Play Store 上架审核会拒）。**禁止使用**。

部分国产 ROM（华为 EMUI / OPPO ColorOS / 某些蓝牙路径）AAudio 实现 bug 会让 `getTimestamp()` 返回 false / framePosition=0 / 时间戳停滞。降级方案：

```kotlin
// 检测：连续 3 次 getTimestamp 返回 false 或 framePosition 不变 → 标记"硬件时基不可用"
// 降级：用静态路由表 + 已写入帧数估算
//   t_play0_estimated = t_first_write + STATIC_ROUTE_LATENCY_MS[currentRoute]
// 静态路由表（经验值，可在 v0.3 真机基准后调整）：
val STATIC_ROUTE_LATENCY_MS = mapOf(
    AudioRoute.BUILTIN_SPEAKER to 80,    // 内置扬声器
    AudioRoute.WIRED_HEADSET  to 40,     // 有线耳机
    AudioRoute.EARPIECE       to 20,     // 听筒
    AudioRoute.BLUETOOTH_A2DP to 200,    // 蓝牙 A2DP（粗略，SBC/AAC/aptX 差异大）
    AudioRoute.BLUETOOTH_SCO  to 150,    // 蓝牙通话路由
    AudioRoute.USB_DEVICE     to 50,     // USB 音频
)
// 实测精度：getTimestamp 路径 ±30ms；静态表降级路径 ±150ms（蓝牙更差）
// 蓝牙路径检测到降级 → 弹一次 Snackbar "蓝牙音频精度受限，建议改扬声器"（本次连接仅一次）
// 监控指标: hdh_audio_timestamp_fallback_total{device_model, route} Counter
```

**v0.3 真机矩阵退出条件**（§14 R6 强化）：必须覆盖至少一台 `getTimestamp` 返回 false 的机型（如华为 HarmonyOS 4.x + 蓝牙 SBC）并验证降级路径出声且 viseme 漂移 < 200ms。

不加这一段，§3.6 延迟预算里 80~600ms 的硬件延迟无法量化，viseme 漂移会让 D 档体验崩盘。

### 3.5 文本切片与 TTS 流式策略

voice-gateway 流式累积 Hermes 返回 token：

```python
buffer = ""
last_flush_ts = now()
async for token in hermes_stream:
    buffer += token
    剥离 <emo .../> <gesture .../> tag → 打 emotion/action 帧（带预估时间戳）
    if 满足任一切片条件:
        send_to_tts(buffer)
        buffer = ""
        last_flush_ts = now()
    # 防卡死：1.5s 静默强制 flush
    if now() - last_flush_ts > 1.5 and buffer:
        send_to_tts(buffer)
        buffer = ""
if buffer: send_to_tts(buffer)   # 流正常结束 flush

# Hermes 错误事件 → 立即 flush 残留 buffer + 发 0x90 error
```

**切片条件（按语言分支）** — [W13] 修订：

| 语言 | 标点集 | 长度阈值 | 额外条件 |
|---|---|---|---|
| 中文 (zh) | `。！？；` | 60 字 | — |
| 英文 (en) | `.!?;` | 80 字 | — |
| 日文 (ja) | `。！？、` | 40 字 | 检测助词 `は/を/が/に/で/と` 后空格 |

**Viseme 对齐策略表** — [W14] 修订（按 lang × provider）：

| 语言 \ Provider | Edge-TTS (默认) | Azure (BYOK) | MiniMax (BYOK) |
|---|---|---|---|
| zh | `pypinyin` → A/I/U/E/O + 时间均分 | 原生 viseme 事件（ARKit 52 weights） | 原生 phoneme 时间轴 → A/I/U/E/O |
| en | `g2p_en` (CMU) → viseme + 时间均分 | 原生 viseme 事件 | phoneme 时间轴 |
| ja | `pyopenjtalk` → 音素 → A/I/U/E/O | 原生 viseme 事件 | phoneme 时间轴 |

**精度说明**：Edge-TTS 路径的 viseme 精度低于 BYOK 云路径；非中文（en/ja）的 Edge-TTS viseme 仅依赖音素级时长均分，无法表达重音/拗音/促音的细微嘴型变化。

### 3.6 延迟预算（[C6 / W19 / WR4 D002] 修订：拆 server-side / client-side SLO，标记待验证）

**目标 SLO（待 v0.3 真机基准验证后冻结）**：

| 维度 | 范围 | 目标 | 验证方式 |
|---|---|---|---|
| **Server-side**（GW 内部，可由 `/metrics` 直接量化） | text_in 到 gateway → audio_out_start 下行 | P50 < 600ms / P95 < 1200ms | Prometheus histogram，每周回归 |
| **Client-side**（端到端，含 Android STT + AudioTrack 硬件延迟 + Unity 渲染） | 用户说完 → 数字人开口 | 待 v0.3 真机基准（Pixel + 小米 + 华为 + OPPO）测出真实分布后冻结 | 真机录制 + 帧分析 |
| **临时假设** | 默认路径 | P50 < 2.5s / P95 < 4.0s（**待验证**） | — |
| **临时假设** | BYOK 路径 | P50 < 2.8s / P95 < 4.5s（**待验证**） | — |

```
（参考分解，非 SLO 承诺）
用户说完话                                                        T=0
  ↓  Android STT 识别 (SpeechRecognizer 实测分布)                +600~1200ms
[text_in 上行]                                                    +50ms
  ↓  voice-gateway → Hermes /chat/completions                    +100ms
  ↓  Hermes 上游 LLM TTFT                                         +400ms
[首个 token 流回 gateway]                                         T≈1150~1750ms
  ↓  累积到首句末标点                                              +200ms
  ↓  Edge-TTS 调用 + 首音返回                                     +250ms
[首个 audio_out_chunk 下行]                                       T≈1600~2200ms
  ↓  App jitter buffer (固定 120ms — D002 砍简自适应)             +120ms
  ↓  AudioTrack 硬件输出延迟 (扬声器 80~200ms / 蓝牙 +150~400ms) +80~600ms
  ↓  Unity BlendShape 渲染管线 (≥ 2 帧 @ 60fps)                  +33ms
[扬声器实际出声 + 嘴型同步上屏]                                    T≈1833~2953ms
```

**工具调用分支**（[IR7 D002] 新增）：
- 触发 `web_search` → +800~3000ms（取决于上游搜索引擎）
- 触发 `session_search` → +200~500ms
- 工具触发时 P95 临时放宽到 4.5s；App 端播放 `thinking` 动作（§5.1）作视觉反馈

**SLO 监控**（暴露在 `/metrics`，仅 server-side 部分）：
- `hdh_gw_internal_latency_seconds{path=default|byok}` Histogram
- `hdh_llm_ttft_seconds`, `hdh_tts_first_audio_seconds{provider=...}`
- Client-side latency 由 App 通过 0x88 history_page 上报 + 设置页诊断导出（用户主动）
- 每周回归对比，P95 退化 > 20% 自动开 GitHub issue

**硬约束**：voice-gateway 必须**首句 TTS 启动后立刻发 audio_out_start + 第一个 chunk**，不能等整段 TTS 完成。Edge-TTS 流式输出原生支持。

### 3.7 重连与状态恢复（[C8] 修订）

| 触发 | App 行为 | Gateway 行为 |
|---|---|---|
| **用户按"打断"按钮**（橙色态）| 立即 `AudioTrack.flush()` + 清 jitter buffer + viseme 归零 + 触发 idle 动作；按钮恢复灰色（**< 100ms 完成**） | 收到 0x05 interrupt：cancel 当前 Hermes 流；不再下发该 msg_id 后续帧；写历史时标记"已打断"；ACK 0x90 `{"code": "interrupted"}` |
| **WS 断开** | 重连退避：1s, 5s, 30s, 60s, 300s（5 档，上限 300s — [D002 砍简]）；±20% jitter 防风暴；后台 5min 停重连，回前台立即重试 | 保留会话映射 30 分钟；接受同 device_credential 重连 |
| **重连握手成功** | 发 history_req 同步增量；恢复 UI 状态 | 推 history_page（含未送达的最终消息） |
| **重连后有未完成的回复** | 显示"...(已断开，重新提问)" + 该消息气泡下方加"重试"按钮 | 不重发；等待用户重问 |

**关键修订**：原方案"中断时本地缓存继续播放完"被废弃——用户按打断必须立即静音，否则按了没反应、连按风暴。这是 D 档体验的强制约束。

### 3.8 心跳与背压（[W16 / I8 / D002 砍简] 修订）

**心跳**：
- 前台：客户端每 30s 发 ping，服务端 30s 内必 pong；120s 无 pong → 客户端主动 close 重连
- 后台：90s 间隔（节能模式 + 防 doze 阻塞）；后台超过 5 分钟 → 主动 close WS 不再保持长连接；回前台立即重连
- Doze 模式下定时器由系统批量延迟到 maintenance window，**不主动 setExactAndAllowWhileIdle 唤醒**（违反 §8.6"App 不主动唤醒 radio"原则）

**背压**：
- 服务端检测 WS sendBuffer > 256KB → 暂停 audio_out_chunk 生产；恢复到 < 64KB → 继续
- 监控指标：`hdh_backpressure_events_total{direction=upstream|downstream}` Counter

**Jitter Buffer（[D002 砍简] 固定 120ms，自适应留 v2）**：
- 固定 6 帧（120ms @ 20ms 帧）— 在 4G/弱 Wi-Fi 下 95% 场景够用
- 运行中 buffer < 1 帧 → 插入静音保护
- 监控指标：`hdh_jitter_underrun_total` Counter
- v2 再做自适应升降级

---

## 4. 鉴权 / 安全模型

### 4.1 配对与设备凭证（[C1 / C11 / W18] 修订）

**两阶段鉴权**：

```
阶段 1 · 一次性配对（首次连接 / 撤销后重新连接）
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. 服务端启动 / gateway-admin pair                            │
  │ 2. 生成 pairing_code (BIP39 6 词)，存 SQLite，TTL 5 分钟      │
  │ 3. 输出 PNG 二维码 + 一次性 deep link + 终端 ASCII（fallback）│
  │ 4. App 扫码或粘贴 → 调 POST /v1/pair                          │
  │    Body: { pairing_code, device_pubkey, device_name }       │
  │ 5. GW 验证 pairing_code 未过期未使用                          │
  │ 6. 签发 device_credential (ed25519 签名, 含 device_id, exp)   │
  │ 7. 返回 { device_id, device_credential, server_features }    │
  │ 8. pairing_code 立即失效                                      │
  └─────────────────────────────────────────────────────────────┘

阶段 2 · 长期连接
  - App 调 wss:// 时强制带 Authorization: Bearer {device_credential}
  - 禁止 URL query 传凭证
  - GW 验签：拒绝过期、被撤销、跨设备重放
```

**凭证规格**（[WR2 / D002] 修正表述）：
- `pairing_code`：**从 BIP39 wordlist（2048 英文小写词）随机抽取 6 词**作为高熵 token（66 bit 熵）；**不是 BIP39 mnemonic**（标准 mnemonic 是 12/15/18/21/24 词且含校验位）；不要用标准 mnemonic 校验库
- TTL 5 分钟，单次使用，使用后立即失效
- **输入规范化**（[WR20 / D002] 新增）：服务端接受时统一 `lowercase` + 任意 whitespace/dash/underscore 分隔 → 内部存为单一形式 "apple river tiger cloud music bridge"（空格分隔小写）
- 二维码 deep link 用 dash 分隔：`apple-river-tiger-cloud-music-bridge`（URL-safe）
- 终端 / 日志输出统一 dash 分隔
- `device_credential`：ed25519 签名的 JWT，含 `device_id` / `iat` / `exp`（默认 180 天）/ `device_pubkey_hash`
- `gateway-admin device list/revoke/rotate <id>` 用于服务端管理

**为何不用 32 hex 共享 token**（[W18] 旧理由）：手工输入 32 字符错误率 > 30%；6 词 BIP39 抽取可读、可口述、可记忆，熵足够安全。

### 4.1.1 服务端密钥管理（[WR11 / D002] 新增）

服务端持有两套密钥：
1. `signing.ed25519` — ed25519 私钥，用于签发 `device_credential` JWT
2. `byok_master.key` — Fernet 32 字节对称密钥，用于加密存储用户 BYOK key

**生命周期**：

```
首次启动 (bootstrap.py):
  - 检测 /data/keys/signing.ed25519 不存在 → 自动生成 + chmod 0600
  - 生成 /data/keys/byok_master.key (Fernet 32 字节) + chmod 0600
  - 密钥永不通过 stdout / docker logs / metric 标签输出
  
轮换 (v1 仅手动):
  - gateway-admin key rotate signing
    → 生成新密钥；保留旧密钥 30 天用于验签存量 JWT；30 天后强制全设备重配
  - gateway-admin key rotate byok-master
    → 用新密钥重新加密所有 BYOK key 后落库

备份:
  - keys/ 目录纳入 gateway-admin backup
  - 备份文件本身用 backup-passphrase 加密 (gateway-admin backup --to /data/backups/ --pass ...)

灾难场景 ([WR17 / D002] DR runbook):
  - keys/ 丢失 → 所有 device_credential 失效 + 所有 BYOK key 失效
    → 全部设备必须重新配对，所有用户必须重输 BYOK key
    → README 显式提示此风险

多副本部署 (v1 不支持):
  - 单 Docker Compose 单 gateway 进程
  - v2 才考虑多副本，届时密钥需通过 K8s secret / vault 注入
```

### 4.2 TLS 强制

- 不接受 `ws://`，App 启动时校验 URL scheme 必须 `wss://`
- 两种 TLS 部署：
  - **Caddy + Let's Encrypt**（用户有域名 + 公网 IP）
  - **Cloudflare Tunnel**（无需公网 IP，自动 TLS）
- **Cloudflare Tunnel 边界**（[W8] 修订）：
  - 临时 `*.trycloudflare.com` 域名不保证持久性，重启可能变（推荐生产环境用命名 tunnel）
  - **音视频流经第三方（CF）中转**，BYOK 用户的音频帧 + LLM 文本会经过 CF 边缘节点
  - 必须在首次配对时显式告知用户："使用 Cloudflare Tunnel 意味着你的对话内容会经过 Cloudflare 网络（CF 不解密 TLS，但元数据可见）"，由用户同意

### 4.3 Hermes 工具白名单（[XR3 D003] 精确按 toolset 命名）

D002 已修掉假设性 `/v1/tools` 端点，但工具/toolset 命名仍混淆。**D003 重新校正**——Hermes 的实际抽象是 **toolset**（一组工具的命名集合），不是单个工具。

**Hermes toolset 命名**（按 `~/.hermes/hermes-agent/toolsets.py` 实际定义）：

| toolset 名 | 含义 | v1 允许？ | 理由 |
|---|---|---|---|
| `default` | LLM 自带能力（无外部工具调用） | ✅ 允许 | 纯聊天 |
| `web_search` | Web 搜索 | ✅ 允许 | 用户查询提升 |
| `memory` | Honcho 用户建模 + 长期记忆 | ✅ 允许 | Hermes 核心价值 |
| `session_search` | FTS5 历史会话搜索 | ✅ 允许 | 跨会话检索 |
| `terminal` | shell 命令执行 | ❌ 禁用 | 远程 RCE 边界 |
| `file` | 文件读写 | ❌ 禁用 | 敏感数据外泄 |
| `cronjob` | 定时任务 | ❌ 禁用 | 主动对话留 v2 |
| `skills` | 自学技能创建（agentskills.io） | ❌ 禁用 | 防 prompt 注入持久化 |

> **重要**：以上 toolset 命名以 v1.0 GA 之前实测确认的 Hermes 真实 `toolsets.py` 内容为准。**新增 R12 milestone 退出条件**：v0.0 准备期必须 `grep -r "^class.*Toolset\|TOOLSET_NAME" ~/.hermes/hermes-agent/` 核对，发现命名变更立即更新本表。

**配置落地方式**：
- Hermes 通过 `platform_toolsets.api_server.enabled_toolsets` 配置项控制 API server 启用哪些 toolset
- v1 gateway-config.yaml：
  ```yaml
  hermes_platform:
    api_server:
      enabled_toolsets:
        - default
        - web_search
        - memory
        - session_search
      # 显式不列入 terminal / file / cronjob / skills
  ```
- voice-gateway 启动时通过 `GET /v1/capabilities` 拉取实际生效的 toolset 列表
  - 若返回中含 deny 列表里的任意 toolset → **fatal exit**
  - 若 allow 列表缺失关键项（如 `memory` 不可用）→ warn 日志 + 标记 `server_features` 降级（hello_ack 不报 `long_term_memory`）
  - 校验结果写入 `/data/tool-audit-{timestamp}.json` 供审计

**与 §12 v1 围栏一致性**：
- "数字人主动发起对话"留 v2 ↔ 这里禁用 `cronjob` toolset，一致
- "App 内捏脸"永不做 ↔ 与 toolset 无关，不影响
- §12 行 1422 同步：`cronjob` toolset 重新评估在 v2 是否在受限沙盒下开放

### 4.4 BYOK key 安全（[C11] 修订）

**流程**：
```
1. App 设置页输入 Azure/MiniMax key
2. App 调 0x0A byok_test（不含 key）+ 0x09 byok_key_set
   - 0x09 帧必须含 hello_ack 返回的 byok_upload_credential（5 分钟有效，单 byok 一次性）
3. GW 验证 byok_upload_credential 后将 key 用服务端主密钥加密（Fernet/age）存入 SQLite
4. GW 内存中 key 永不通过 stdout / log / metric 标签泄漏
5. App 调 0x0A byok_test → GW 用 key 试调一次 provider → 返回 0x91 byok_test_result
6. 撤销：用户在设置页"删除 BYOK"按钮 → 0x09 帧 key=null
   - 网络断开 ≠ 撤销（修正原"断线清密钥"误设计）
   - 仅用户主动撤销 / device_credential 被 revoke 时才清密钥
```

**安全约束**：
- `byok_upload_credential` 单次使用，使用后立即失效
- key 上传 0x09 帧大小 ≤ 16 KB
- 同一 device_id 同一 provider 同一时间只能有一条 key 记录（新 key 覆盖旧 key）
- GW 进程崩溃后内存中 key 清零，磁盘加密存储需重新载入

### 4.5 速率限制与攻击面（[W5] 新增）

**限流策略**（per device_id + per IP 双轨）：
| 资源 | 限流 |
|---|---|
| 配对 `POST /v1/pair` | 10 次 / 小时 / IP |
| WS 新连接 | 30 次 / 小时 / device_id |
| `text_in` / `audio_in_start` | 60 次 / 分钟 / device |
| `history_req` | 10 次 / 分钟 / device |
| `byok_key_set` / `byok_test` | 5 次 / 分钟 / device |
| 单帧 payload | 见 §3.3.1 表 |
| 整体连接最大并发 turn | 1 per conv_id（[C3] 串行化） |

**熔断**（[WR5 D002] 修订：per-provider 不全员踢线）：
- 单 device_id 连续 5 次 401/403 → 冻结 15 分钟
- **某 provider 错误率 > 10% 持续 60s**（不是全局错误率）→ 该 provider 暂停 5 分钟，客户端**降级到 fallback provider**（如 BYOK Azure TTS 故障 → fallback 到 Edge-TTS 默认）；不踢现有 WS 连接
- **不做"全员 fatal 断开"** — 该机制会把局部故障放大成全站雪崩

### 4.6 日志脱敏

- 默认 LOG_LEVEL=info：不记录 prompt / 用户语音转写 / BYOK key / device_credential 完整值
- LOG_LEVEL=debug：仅记录前 50 字 prompt + 哈希；device_credential 仅记录前 8 + 后 4 字符 + sha256[:8]
- BYOK key / pairing_code / device_pubkey 永不记录，任何级别

---

## 5. 情绪 / 动作 Tag 系统

### 5.1 词汇表（v1 冻结）

**情绪（6 个，VRM 1.0 standard expressions 对应）**：
```
neutral | happy | angry | sad | surprised | relaxed
```

**动作（9 个，Mixamo 通用动作 — D002 修正：原始动画不入仓库）+ 时长** — [I6] 新增：

| 动作 | 默认时长 | 后续行为 |
|---|---|---|
| `idle` | 持续 | 循环 |
| `nod` | 1.2s | 回 idle |
| `shake_head` | 1.5s | 回 idle |
| `thinking` | 3.0s | 循环到下一个 action 触发 |
| `pointing` | 2.0s | 回 idle |
| `wave` | 1.8s | 回 idle |
| `shrug` | 1.5s | 回 idle |
| `lean_forward` | 2.5s | 回 idle（姿态保持 0.5s 后过渡） |
| `lean_back` | 2.0s | 回 idle |

**动作优先级**：后到的 action 覆盖前一个（即时切过渡 200ms）；情绪持续不被动作中断。

**强度**：0.0 ~ 1.0，仅情绪支持，动作不支持。

### 5.2 LLM 输出格式（[C2 / CR5 D002] 修订：合成根 + 流式拼接）

**为何改用合法 XML**：原 `<emo:happy:0.6>` 不是合法 XML。但 LLM 直接输出"多顶层元素 + 裸文本"也不是 well-formed XML document（XML 要求单一根元素）。

**约定**：LLM 输出**XML fragment**（不带根），voice-gateway 在解析时**合成 root** 包裹：

```python
# voice-gateway 流式解析时
parser = lxml.etree.XMLPullParser(events=("start", "end"))
parser.feed("<stream>")          # 合成 root 开始
async for token in hermes_stream:
    parser.feed(token)
    for event, elem in parser.read_events():
        ...
parser.feed("</stream>")         # 流结束时关闭 root
```

LLM 实际输出的 fragment（**注意没有 wrapper**，是给 TTS 流的）：

```xml
<emo name="happy" intensity="0.6">你今天找我聊这个</emo><gesture name="nod"/><emo name="relaxed">真的让我很开心，我们慢慢说</emo>
```

**语法规则**：
- `<emo name="X" intensity="Y">...</emo>` — 情绪范围，`name` ∈ 6 情绪，`intensity` ∈ [0.0, 1.0]，默认 1.0
- `<gesture name="X"/>` — 自闭合动作触发，`name` ∈ 9 动作
- **非栈式语义**：`<emo>` 设置当前情绪 → `</emo>` 回到 `neutral`（不支持嵌套栈）
- 文本节点是正文（送 TTS），元素是控制信号
- 未知 name → 警告日志，按 neutral / idle 处理
- 转义：正文里的 `<` `>` `&` 由 Hermes 输出时自动转义为 `&lt;` `&gt;` `&amp;`

### 5.2.1 流式解析与 TTS 投喂策略（[C2 / WR8 D002] 修订：统一超时常量）

```python
class StreamingTagParser:
    """
    增量解析 XML fragment, 用合成根 <stream> 包裹.
    pending_buffer 缓冲遇 < 后未闭合的内容.
    超时阈值与 §3.5 切片超时统一为 CONFIG.tts_flush_idle_ms (默认 1500ms).
    """
    CONFIG_tts_flush_idle_ms = 1500
    CONFIG_lt_buffer_max_chars = 100   # < 后超 100 字符仍未闭合则视为字面量

    def feed(self, token: str) -> Iterator[Event]:
        # state machine: TEXT | LT_SEEN | IN_TAG | IN_ATTR
        # 遇 < → state=LT_SEEN, 暂不投 TTS, 启动空闲计时
        # 标签闭合 → emit TagEvent, flush 残留 TEXT
        # 任一触发: 自上次 token 间隔 > 1500ms OR < 后累积 > 100 字符 → 视为误判, 当文本 flush
```

**容错降级**：
- 触发条件**取较晚者**：① 自上次 token > 1500ms（与 §3.5 切片 idle 阈值统一）② 或 `<` 后累积 > 100 字符
- 慢 LLM 场景（如 ollama + Qwen-7B 本地，token 间隔 150-300ms，单标签 1.5-3s 完成）→ 不会误杀
- 误判时把 `<...` 残段当文本 flush（保留 `<` 进 TTS，听感是"小于号"），记 `hdh_xml_tag_truncated_total` Counter
- 主流 LLM 违反 schema 概率 5-15%，阈值 `hdh_xml_tag_malformed_total` 跨过 5% → README 提示 prompt 调优
- Hermes Skill 末尾加 few-shot 样例（v1 仅做这一层防御，不做"自动重提示"——避免复杂度）

### 5.3 Hermes system message 注入（[XR2 D003] 修订：放弃假设的 skill 注册 API）

**为何改方式**：Hermes 实际暴露的端点是 `/v1/chat/completions`、`/v1/responses`、`/v1/runs`、`/v1/capabilities`、`/v1/models`、`/health`，**没有 skill 注册端点**。且 §4.3 工具白名单又禁用 `skills` toolset，"通过 Hermes API 注册 digital-human-tagger skill" 是自相矛盾的假设性接口。

**新方案**：voice-gateway 每次调用 `/v1/chat/completions` 时**注入 system message**（OpenAI 兼容 API 原生支持）：

```python
# voice-gateway 调 Hermes 时拼接 messages:
messages = [
    {"role": "system", "content": DIGITAL_HUMAN_TAGGER_SYSTEM_PROMPT},
    *conversation_history,   # 已剥 tag 的历史（content_clean，见 §5.4）
    {"role": "user", "content": user_input},
]
```

**`DIGITAL_HUMAN_TAGGER_SYSTEM_PROMPT` 内容**（含 [W3 / XR2 D003] 强化的"禁止包裹元素"指令）：

```
你正在以 3D 数字人形象与用户对话。每段回复必须用以下合法 XML 元素标注情绪和动作：

- <emo name="NAME" intensity="N">...</emo>
  NAME ∈ {neutral, happy, angry, sad, surprised, relaxed}
  intensity ∈ [0.0, 1.0]，缺省 1.0
  </emo> 之后回到默认 neutral

- <gesture name="NAME"/>
  自闭合（不要写 </gesture>），name ∈ {idle, nod, shake_head,
  thinking, pointing, wave, shrug, lean_forward, lean_back}

示例：
<emo name="happy" intensity="0.7">太好了！<gesture name="nod"/>我完全同意你的想法。</emo>

【硬性要求】
1. 输出必须是合法 XML fragment（属性值双引号引用，< > & 字符用 &lt; &gt; &amp; 转义）
2. 标签嵌入在文本中,不要解释也不要末尾汇总
3. 不要嵌套 <emo>,</emo> 直接回 neutral
4. **禁止输出任何包裹元素**:不要写 <reply>...</reply>、<answer>...</answer>、<root>...</root>、<response>...</response>,也不要写 markdown 代码块围栏 ```xml。从回复的第一个字符开始就是正文或情绪/动作标签,到最后一个字符结束。违反这条会让 TTS 卡流。
5. 一句话 1~3 个标签足够,不要过密
```

**Few-shot 容错**：v0.2 起，若 GW 检测到 `hdh_xml_tag_malformed_total` 跨过 5%，开 GitHub issue 让维护者调整 prompt（不做自动重提示，避免 v1 复杂度）。

**与 Hermes Skill 系统的关系**：本项目**不使用** Hermes 内置 skill 系统（agentskills.io 格式）。所有数字人特定 prompt 注入由 voice-gateway 控制。Hermes 在本项目中扮演纯 LLM-with-memory 角色。

**Parser 防御补充**（[W3 D003]）：§5.2.1 状态机检测到 `<stream>` 的首个 child 是除 `<emo>` `<gesture>` 之外的元素 → warn log + 跳过该外层包裹直接解析其 children，作为 LLM 偶发违规的兜底。

### 5.4 历史压缩与 Hermes Memory 回写（[I5] 新增）

避免 tag 污染 Hermes 长期记忆 + 节省 token：
- **写回 Hermes memory 的文本必须是 `content_clean`**（已剥离 tag 的纯文本）
- App 端 Room 数据库存两份：`content`（含 tag，用于回放 UI 展示）和 `content_clean`（用于历史搜索）
- 服务端 SQLite 同样存两份字段
- Hermes session API 的 `X-Hermes-Session-Id` 续接时，GW 传递的 history 必须是 `content_clean`

---

## 6. 形象（VRM）资产体系

### 6.1 三层回退加载顺序

```
启动时:
  ① 用户本地导入的 .vrm 文件（最优先）
  ② 服务器配置的 vrm_url（hello_ack 含 default_avatar_url + default_avatar_sha256）
  ③ APK assets/default.vrm（保底）
```

**加载失败检测与降级**（[W6] 修订）：

| 阶段 | 超时 | 失败处理 |
|---|---|---|
| 下载（仅 ② 层） | 30s + 单文件 ≤ 50MB | 重试 1 次后降级 |
| sha256 校验（仅 ② 层） | — | 不匹配立即降级 + 日志告警 |
| 解析（VRM 头 + glTF） | 5s | 失败立即降级 |
| BlendShape 烘焙 | 10s | 失败立即降级 |
| Spring Bone 初始化 | 5s | 失败可继续（spring bone 仅装饰） |
| 首次显示渲染 | 3s | 超时记入指标但不降级 |
| **总预算 P50** | **< 3s** | — |
| **总预算 P95** | **< 6s** | — |
| **超 8s** | — | 视为失败，触发降级 |

**性能持久化缓存** — [C10] 新增：
- 服务器 URL 模型首次下载后，按 sha256 命名缓存到 `filesDir/avatar-cache/{sha256}.vrm` + `{sha256}.parsed.bin`（解析后的 Mesh/Texture）
- 二次启动直接 mmap 解析缓存，目标 < 1s
- LRU 淘汰，缓存上限 200MB

每层加载失败自动降级到下一层。最终都失败 → 显示纯色背景 + 错误提示，但仍可文字对话。

### 6.2 VRM 技术规格（v1 强制约束）

| 项 | 推荐 | 上限 | 失败处理 |
|---|---|---|---|
| 多边形 | 30k~60k tris | 100k | > 100k 拒绝加载，弹错 |
| 纹理 | 2048×2048 × 4 | 4096×4096 × 4 | 超限自动 mipmap 降采样 |
| 骨骼 | VRM Humanoid 55 根 | 严格符合 | 缺关键骨骼 → 拒绝加载 |
| 情绪 BlendShape | 6 个标准 | 必须全 | 缺失项用 neutral 替代 |
| 嘴型 BlendShape | A/I/U/E/O 5 个 | 必须全 | 缺失 → 嘴不动 |
| 嘴型 BlendShape (扩展) | ARKit 52 | 可选 | 缺失则 BYOK Azure 路径自动降级到 5 个 viseme，并在设置页 TTS 提供商旁显示"⚠️ 当前形象不支持精细口型"提示 |
| Spring Bone | < 60 根 | < 100 根 | 超限警告但加载 |
| Shader | MToon (URP) | URP 兼容 | 非 URP shader **拒绝加载**（不再"强转 MToon"——不可靠且会出现纹理错乱） |
| 文件大小 | < 15 MB | < 30 MB | > 30 MB 拒绝（防 OOM） |
| **加载时间 P50** | **< 3s** | < 8s | 见 §6.1 降级表 |

### 6.2.1 VRM ↔ ARKit BlendShape 映射（[I9] 新增）

下行 `0x86 viseme` 帧的 `phoneme` 字段取值约定：

| 类别 | 取值集合 | 用途 |
|---|---|---|
| 简化 viseme | `A` / `I` / `U` / `E` / `O` / `sil` | Edge-TTS / VRM 默认 5 元音 |
| ARKit BlendShape | `jawOpen` / `mouthFunnel` / `mouthPucker` / `mouthShrugLower` / `mouthLeft` / `mouthRight` / `cheekPuff` / ...（共 52 项） | Azure Neural TTS 原生输出 |

App 端按映射表查 BlendShapeKey：
```kotlin
// VRM standard 5 viseme → VRM blendshape preset key
"A" -> BlendShapePreset.Aa
"I" -> BlendShapePreset.Ih
"U" -> BlendShapePreset.Ou
"E" -> BlendShapePreset.Ee
"O" -> BlendShapePreset.Oh
"sil" -> all zero

// ARKit 52 → VRM 1.0 expression slot
"jawOpen" -> "jawOpen" (VRM 1.0 含此项)
"mouthSmileLeft" -> "mouthSmileLeft"
// ... 完整映射表见 docs/vrm-blendshape-mapping.md
```

### 6.3 默认模型制作流程

1. 安装 VRoid Studio (免费)
2. 选女性脸模板 → 调五官 → 选发型 → 选服装
3. 导出 VRM 1.0
4. License metadata 设置：
   - Allow commercial use: YES
   - Allow redistribution: YES
   - Allow modification: YES
   - Author: {项目维护者}
   - Title: hermes-digital-human default avatar
5. 用 UniVRM Inspector 校验所有 BlendShape 齐全
6. 放入 `android/app/src/main/assets/default.vrm`

**[I4] 注意**：VRoid Studio 当前版本仅生成 VRM 标准 5 viseme + 6 expression，**不生成 ARKit 52 BlendShape**。后果：默认 avatar 在 BYOK Azure 路径下嘴型精度等价于 Edge-TTS 路径（5 元音），无法表达精细口型。如需 ARKit 52，需要在 Blender 中手工补齐或选用其他 VRM 制作工具。

---

## 7. 会话与历史模型

### 7.1 数据模型

**voice-gateway SQLite（/data/gateway.db）** — [W7] 加 schema 版本与迁移：

```sql
CREATE TABLE schema_migration (
  version INTEGER PRIMARY KEY,
  applied_at INTEGER NOT NULL
);
-- 启动时按 version 顺序运行 migrations/{N}-*.sql；记录 retention 7 天对软删除消息生效

CREATE TABLE conversation (
  conv_id TEXT PRIMARY KEY,           -- UUIDv7
  title TEXT,
  created_at INTEGER,
  updated_at INTEGER,
  hermes_session_id TEXT,             -- 映射到 Hermes 的 X-Hermes-Session-Id
  active_turn_id TEXT,                -- 当前活跃 turn，NULL = 空闲
  active_device_id TEXT,              -- 哪台设备占用着该 conv 的写权
  active_lease_until INTEGER,         -- 占用租约到期时间（毫秒 epoch）
  active_lease_version INTEGER NOT NULL DEFAULT 0  -- [W2 D003] fencing token, 单调递增
);
CREATE INDEX ix_conv_active ON conversation(active_lease_until);

CREATE TABLE message (
  msg_id TEXT PRIMARY KEY,            -- UUIDv7
  conv_id TEXT NOT NULL REFERENCES conversation,
  turn_id TEXT,                       -- 同一 turn 内可有多条 (user/assistant pair)
  role TEXT,                          -- 'user' | 'assistant'
  content TEXT,                       -- 含 emotion/action 原始 tag
  content_clean TEXT,                 -- 剥离 tag 后的纯文本，用于回传 Hermes
  status TEXT,                        -- 'pending' | 'complete' | 'interrupted' | 'error'
  created_at INTEGER NOT NULL
);
CREATE INDEX ix_msg_conv_ts ON message(conv_id, created_at);

CREATE TABLE byok_key (
  device_id TEXT NOT NULL,
  provider TEXT NOT NULL,
  encrypted_key BLOB,                 -- 服务器主密钥加密
  created_at INTEGER NOT NULL,
  PRIMARY KEY (device_id, provider)
);

CREATE TABLE device (
  device_id TEXT PRIMARY KEY,
  device_pubkey TEXT NOT NULL,        -- ed25519 公钥，配对时绑定
  device_name TEXT,                   -- 用户可见名（如"我的小米14"）
  created_at INTEGER NOT NULL,
  last_seen_at INTEGER,
  revoked_at INTEGER                  -- NULL = 有效
);

CREATE TABLE pairing_code (
  code TEXT PRIMARY KEY,              -- BIP39 6词
  expires_at INTEGER NOT NULL,
  used_at INTEGER                     -- 单次使用
);
```

**SQLite 配置**：
- 启用 WAL 模式：`PRAGMA journal_mode=WAL`
- 7 天 retention：每日凌晨 vacuum 一次，超 90 天的 message 自动归档到 `gateway-archive-{YYYY-MM}.db.gz`
- 备份：`gateway-admin backup --to /data/backups/`（cron 周备份）

**Android Room（设备本地）**：

```kotlin
@Entity
data class Conversation(
  @PrimaryKey val convId: String,
  val title: String,
  val updatedAt: Long
)

@Entity
data class Message(
  @PrimaryKey val msgId: String,
  val convId: String,
  val turnId: String?,
  val role: String,
  val content: String,        // 含 tag，UI 回放用
  val contentClean: String,   // 不含 tag，搜索/分享用
  val createdAt: Long,
  val ackStatus: Int          // 0=pending, 1=server-confirmed, 2=interrupted, 3=error
)
```

Room 只缓存最近 N=500 条 / 会话；翻历史超过缓存 → 触发 `history_req`，单 in-flight 请求队列防并发风暴；用户滚动到底部 80% 触发预取下一页。

### 7.2 多设备同步与并发控制（[C3 / CR3 D002] 修订：原子 CAS + fencing token）

D001 提出 Turn 租约方案但未给原子原语，aiohttp 多协程下会竞态。D002 给出可实现版。

**核心机制：单活跃 turn + fencing token (lease_version) + 续约 + 死锁恢复**

```sql
-- conversation 表新增字段（[CR3 D002]）：
--   active_turn_id        TEXT NULL
--   active_device_id      TEXT NULL
--   active_lease_until    INTEGER NULL   -- ms epoch
--   active_lease_version  INTEGER NOT NULL DEFAULT 0  -- fencing token, 单调递增
```

**申请 turn 的原子操作**（SQLite `BEGIN IMMEDIATE` 防并发）：

```python
async def acquire_turn(conv_id, device_id, request_id) -> AcquireResult:
    async with db.transaction():
        await db.execute("BEGIN IMMEDIATE")
        row = await db.fetchone(
            "SELECT active_turn_id, active_lease_until, active_lease_version "
            "FROM conversation WHERE conv_id = ?", (conv_id,))
        now = now_ms()
        # 双重判断顺序:
        if row.active_turn_id is None or row.active_lease_until < now:
            # 空闲 或 脏 lease (已过期但未清扫) → 接受 + fencing 递增
            new_version = row.active_lease_version + 1
            new_turn_id = uuidv7()
            await db.execute(
                "UPDATE conversation SET "
                " active_turn_id=?, active_device_id=?, "
                " active_lease_until=?, active_lease_version=? "
                "WHERE conv_id=?",
                (new_turn_id, device_id, now + 30_000, new_version, conv_id))
            return AcquireResult.ok(new_turn_id, new_version)
        else:
            return AcquireResult.busy(retry_after_ms=row.active_lease_until - now)
```

**续约**（每收到一个下行 token 帧时调用，避免长 LLM 回答被 30s 超时误杀）：

```python
async def renew_turn(conv_id, turn_id, lease_version) -> bool:
    # 只能续约自己持有的 (turn_id, lease_version), 防止迟到 completion 误清
    result = await db.execute(
        "UPDATE conversation SET active_lease_until=? "
        "WHERE conv_id=? AND active_turn_id=? AND active_lease_version=?",
        (now_ms() + 30_000, conv_id, turn_id, lease_version))
    return result.rowcount > 0
```

**释放 turn**（必须按 fencing token 释放，防止迟到 completion 清掉新 turn）：

```python
async def release_turn(conv_id, turn_id, lease_version, reason):
    await db.execute(
        "UPDATE conversation SET "
        " active_turn_id=NULL, active_device_id=NULL, active_lease_until=NULL "
        "WHERE conv_id=? AND active_turn_id=? AND active_lease_version=?",
        (conv_id, turn_id, lease_version))
```

**renew 失败时 GW 行为**（[XR4 D003] 新增——这是租约抢占的语义补全）：

当 `renew_turn` 返回 False（说明 lease 过期后被另一设备 `acquire_turn` 抢占，fencing token 已变），GW 必须立即：

```python
# 在每次下行 token / audio_out_chunk 之前先 renew, 失败则停止
async def on_token_from_hermes(conv_id, turn_id, lease_version, token):
    ok = await renew_turn(conv_id, turn_id, lease_version)
    if not ok:
        # 被抢占了
        await hermes_upstream_stream.cancel()   # ① 立即取消 Hermes upstream
        await ws_send(original_owner_device, 0x8B_turn_state, {
            "conv_id": conv_id, "turn_id": turn_id,
            "state": "preempted",
            "reason": "lease_expired_and_reassigned"
        })
        # ② 不再下发任何 token / audio_out / viseme 给原 owner
        # ③ 写历史时该 message 标记 status="preempted"
        return
    await ws_send(original_owner_device, 0x80_token, token)
```

**单连接约束**（v1）：本机制严格依赖**单进程 aiosqlite + 单 executor**。`gateway-config.yaml` 注释必须显式标注：
```yaml
# v1: connection_pool 不可 > 1，否则 Turn 租约竞态
# v2 多副本部署: 租约必须迁出 SQLite 到 Redis SETNX 或 PostgreSQL advisory lock
```

§4.1.1 v2 多副本一节同步：v2 引入多副本时**必须**把租约迁出 SQLite。

**上行 audio_in_chunk 长录音续约**（[W1 D003] 新增）：用户连续说 > 30s 长音频时，仅靠 audio_in_start 申请的初始 lease 会被自己 30s 超时。解决：`audio_in_start` 申请的初始 lease TTL 改为 **90s**（覆盖最长合理录音）；超过 90s 由客户端强制发 `audio_in_end` 结束本次输入。

**租约生命周期**：

| 时刻 | 行为 |
|---|---|
| 申请：收到 `text_in` / `audio_in_start` | `acquire_turn`；失败 → 0x90 `turn_busy` + `retry_after_ms` |
| 续约：每收到 `0x80 token` 下行帧 | `renew_turn(...)`；长 LLM 回答 / web_search 触发不会被自己 30s 超时误杀 |
| 释放：`msg_complete` / `0x05 interrupt` / `0x90 error` | `release_turn(...)`，按 fencing token 精确清理 |
| 死锁恢复：另一设备申请时检测 `lease_until < now` | 视为脏 lease，覆盖接受 + 写 warn 日志 |

**广播规则**（[WR7 D002] 修订：只广播数据变更，不强加 UI 状态）：

| 事件 | 广播给所有同 device_credential 的连接？ | App 端反应 |
|---|---|---|
| `conv` 增 / 删 / 改名 | 是（0x8A conv_changed） | 抽屉刷新 |
| `message` 新增（msg_complete 序列） | 是（0x80/0x87） | 当前 conv 的对话 UI 追加 |
| `turn_state` 变化（active/completed/interrupted/preempted）| 是（0x8B） | 麦克风按钮状态机更新 |
| **active conv 选择**（用户在 A 设备切到 conv X）| **否** | per-connection state，每个连接维持自己的 active conv |

**冲突场景**：

| 场景 | 结果 |
|---|---|
| A 说话中，B 同时点麦 | B 收 0x90 `turn_busy`，麦克风按钮闪红 + "对方正在说话" |
| A 网络断开但未释放租约 | 30s 后 lease 过期；另一设备申请时检测脏 lease 自动接管 |
| A 按打断 → B 在 A 释放后立即说话 | OK，B 拿到新 turn（新 fencing token） |
| 用户在 A/B 同时改 conv 标题 | SQLite `BEGIN IMMEDIATE` 串行化，后到的赢；其他设备收 conv_changed 刷新 |

---

## 8. App UI 信息架构

### 8.0 Unity 嵌入策略（[C7] 新增 · 在 8.1 之前定基础）

Unity-as-Library 在 Android 上有几个易踩坑：UnityPlayer 是 SurfaceView（强制 Z 最底层硬件合成）、是全局单例、对 Activity 生命周期敏感。错误处理会导致旋转 / 暗黑模式切换 / 低内存回收时整个 App 黑屏或崩溃。

**强制规则**：

1. **Activity 持有方式**：用 `MainActivity.setContentView(unityPlayer)` 直接持有 UnityPlayer，**不要**把 UnityPlayer 嵌入 ComposeView。Compose UI 通过 `WindowManager.addView(composeContainer, TYPE_APPLICATION_PANEL)` 浮于 UnityPlayer 之上。

2. **锁定 configChanges**：MainActivity 必须声明
   ```xml
   android:configChanges="orientation|screenSize|smallestScreenSize|screenLayout|uiMode|density"
   ```
   阻止系统在旋转 / 暗黑模式切换时重建 Activity。

3. **UnityPlayer 全 App 单例**：在 Application 子类中初始化 UnityPlayer，所有 Activity 共享同一实例；禁止跟随 Fragment 销毁。

4. **内存压力响应**：
   ```kotlin
   override fun onTrimMemory(level: Int) {
       super.onTrimMemory(level)
       if (level >= TRIM_MEMORY_RUNNING_CRITICAL) {
           UnityPlayer.UnitySendMessage("HDHBridge", "OnLowMemory", "")
           // Unity 端释放未使用的纹理 + GC
       }
   }
   ```

5. **前后台切换**：
   - `onPause()` → `unityPlayer.pause()` + 暂停 Unity 协程（保留 VRM 加载状态）
   - `onResume()` → `unityPlayer.resume()` + 触发 BlendShape 重新提交
   - **不要**在 onStop 销毁 UnityPlayer（销毁 = 重启 = 数秒重新加载 VRM）

6. **手势分发规则**（[W17] 修订）：
   - 底部 25% 聊天浮层区域 → Compose 优先消费
   - 顶部状态栏 + 抽屉触发区 → Compose 优先消费
   - 中间形象区域 → forwardTouchEventsTo(UnityPlayer)
   - BottomSheet 的 swipeable modifier 需自定义 `NestedScrollConnection` 拦截 down event 避免被 Unity 抢
   - **Unity 端禁用 UniVRM `LookAtTargetMouse`**（触摸跟随）；改用相机射线被动跟随（虚拟焦点位于屏幕中心固定深度）

### 8.1 主界面

```
全屏 Unity SurfaceView (VRM 形象)
├── 顶栏（半透明覆盖，可滑动隐藏）
│   ├── [≡] 抽屉触发
│   ├── 当前会话标题（点击改名）
│   ├── [🎨] 切换形象
│   └── [⚙️] 设置
├── 底部聊天浮层（默认 25% 高，可下拉到 5%）
│   ├── 最近消息气泡（最多 3 条，单气泡最大高度 = 屏幕 20%，超出 fade-out 渐隐 + 双击全屏阅读 — [I3]）
│   ├── 输入框 + 发送按钮
│   └── 麦克风按钮（Siri 风格）
│       - 默认：灰色圆圈
│       - 点击：变红色 + 波纹动画，开始监听
│       - 监听中："识别中..." loading 副状态（防用户重复点 — [W19]）
│       - 识别结束：转上传指示
│       - 数字人讲话中：变橙色"打断"按钮（按下立即静音，< 100ms）
│       - 对方设备占用 turn 时：闪红 + "对方正在说话" 提示（[C3]）
└── 错误浮层（顶部 Toast / Snackbar）
```

### 8.2 抽屉

```
┌──────────────────────┐
│  + 新对话             │
│  ─────────           │
│  [搜索框]            │
│  📌 今天              │
│    • 关于产品规划      │
│    • 周报草稿         │
│  📌 昨天              │
│    • Python 调试      │
│  📌 本周              │
│    • ...             │
│  📌 更早              │
│    • ...             │
│  ─────────           │
│  ⚙️ 设置              │
│  🔌 服务器: ● 已连接   │
└──────────────────────┘
```

### 8.3 设置页（5 段 — 新增"无障碍"）

```
设置
├── 服务器
│   ├── 连接地址（只读）
│   ├── 连接状态 / 重新连接 / 测试
│   ├── 重新配对（撤销当前 device_credential + 跳首启 — [C1]）
│   └── 删除连接（清本地缓存 + 跳首启）
├── 形象
│   ├── 当前形象（缩略图 + 文件大小）
│   ├── 切换到服务器默认形象
│   ├── 切换到 APK 内置形象
│   ├── 从本地选择 VRM 文件...
│   ├── 表情强度 [0~100% 滑条，默认 100% — W#14]
│   ├── 动作启用 [开关，关闭时仅 idle — W#14]
│   └── 数字人音量 [0~100% 滑条，独立于系统媒体音量 — W#14]
├── 语音
│   ├── TTS 提供商 [默认 (服务端 Edge-TTS) ▾]
│   │   - 选 Azure/MiniMax → 弹出 key 输入 + "测试"按钮（0x0A byok_test 帧）
│   │   - ⚠️ 当前形象不支持精细口型（仅 Azure 路径下且 VRM 缺 ARKit 52 时显示）
│   ├── 音色选择（依 provider 动态加载）
│   ├── 语速 [0.5~2.0 滑条，默认 1.0]
│   ├── STT 提供商 [默认 (系统语音识别) ▾]
│   │   - 选 Azure/MiniMax → 弹出 key 输入 + "测试"按钮
│   └── 语音输入语言 [中文 ▾]
├── 无障碍 ([I10] 新增 / [D002 砍简] 去掉 MToon 高对比变体)
│   ├── 仅文字模式 [开关 — 关闭 TTS 播放 + 隐藏 3D 形象，纯文字对话]
│   ├── 始终显示字幕 [开关 — 听障用户]
│   ├── 字体缩放 [跟随系统 / 100% / 130% / 150%]
│   ├── 高对比度 UI 配色 [开关 — Compose Material3 色板，VRM 保持原样]
│   └── 减少动画 [开关 — 抑制 spring bone + 大幅动作，仅保留 idle + 嘴型]
└── 关于
    ├── App 版本 / Gateway 版本 / 协议版本（三者独立显示）
    ├── 导出诊断日志（filesDir/crashes + 最近 100 条 WS 帧元数据 — [I6 修正路径])
    ├── 开源仓库 GitHub 链接
    └── License (MIT) + Acknowledgments (Mixamo 资产用户本地下载，关于页提示)
```

### 8.4 首启流程（[W15 / W18 / CR6 D002] 修订）

```
屏 1：配对（扫码为主，手动为副）
  - 主：[扫描二维码] 大按钮 — 唤起相机
       │
       └─ 二维码内容 = deep link:
             hdhuman://pair?host=...&code=apple-river-tiger-cloud-music-bridge
  - 副：[手动输入] 折叠区
       │  - 服务器地址 (host:port 或 URL，自动检测剪贴板预填)
       │  - 配对码 (BIP39 6 词，详见 IME 适配规则)
       └─ [连接]
  - 附：[从浏览器粘贴链接] — 长按粘贴 deep link，自动解析填入

  IME 适配规则（[CR6 D002] 关键）:
  - BIP39 词表是全英文小写，对中文/日文 IME 用户输入是个噩梦
  - 输入框强制 inputType="textVisiblePassword" + imeOptions="flagForceAscii"
    → Android 强制锁定英文软键盘，避免中文 IME 自动联想 / 日文罗马字转假名
  - 单输入框输入完整 6 词（空格分隔），不再分 6 格（避免与变长词冲突，如 absolute/abstract）
  - 自动联想：从 BIP39 wordlist 做前缀匹配；BIP39 设计保证前 4 字符前缀唯一
  - 输入完成后客户端 normalize：lowercase + 任意分隔 → 统一空格分隔 lowercase
  - TalkBack 启用时：自动推荐"用扫码"，因为视障用户手输 6 词更困难
  - 如果服务端日志输出 dash 分隔但用户复制粘贴整段：normalize 规则保证识别

屏 2：连接 + 形象加载（带进度）
  - 阶段 1：握手 …………………………… 等待中（最长 5s）
  - 阶段 2：拉取 device_credential ………… 等待中
  - 阶段 3：下载默认形象 (sha256:abc...) … 32% (4.2 / 13.1 MB)
  - 阶段 4：解析 VRM ……………………… 完成
  - 阶段 5：烘焙 BlendShape …………………… 完成
  - 阶段 6：触发首次"Hello!" ……………… 完成 → 跳主界面

屏 2 错误分支：
  - DNS 失败 / TLS 失败 / 401 / 配对码过期 / Tunnel 临时域名变化 等
  - 显示具体错误码 + "复制错误详情" + "重试" + "返回修改地址"
  - 网络不可达时显示离线提示，可点 "进入离线模式（仅查看本地历史）"
```

### 8.5 无障碍（[I10 / CR8 D002] 修订：TalkBack 诚实声明）

Android 上架 Play Store 现要求 accessibility 声明。**底线规则**：

- 所有 Compose Button / 图标按钮强制 `Modifier.semantics { contentDescription = "..." }`
- 触摸目标 minSize **48dp × 48dp**（含麦克风按钮、抽屉条目、设置项）
- 字幕：§8.3 "始终显示字幕" + "仅文字模式" 让聋哑用户能用
- 色觉障碍：emotion / connection-status 不仅用颜色区分，需配合图标 / 文字标识
- 字体缩放跟随系统 fontScale（Compose `Text` 直接支持）

**TalkBack 兼容性的诚实声明**（[CR8 D002] 关键）：

> **Unity SurfaceView 渲染区域不可被 TalkBack 朗读**——这是 Unity 平台本身限制（SurfaceView 对 Android AccessibilityService 不可达，UniVRM 形象、表情、动作对盲人用户=黑盒），不是 App 代码能修的。
>
> 同时 §8.0 规则 1 用 `WindowManager.addView(TYPE_APPLICATION_PANEL)` 浮 Compose 层，默认不参与 host Activity 的 AccessibilityNodeProvider 树。需配合 `accessibilityTitle` + `FLAG_NOT_TOUCH_MODAL` 才能让 TalkBack 找到 Compose 内部节点。
>
> **盲人用户应启用 §8.3 "仅文字模式"**：此模式隐藏 Unity 区域、回退到纯 Compose LazyColumn 聊天 UI，可被 TalkBack 完整朗读。首启检测到 TalkBack 启用 → 自动推荐弹窗"启用仅文字模式可获得更好的屏幕阅读器体验"。
>
> 视障辅助仅在"仅文字模式"下声明完整支持；其他模式声明为"部分支持"。Play Store accessibility 声明应严格按此口径填写。

**实现要求**：
- Compose 覆盖层 Window 显式设置 `accessibilityTitle = "数字人聊天界面"` + `FLAG_NOT_TOUCH_MODAL`
- Unity 区域上的 TalkBack hover → `onPopulateAccessibilityEvent` 返回 false（避免无意义的"Unity Player"播报）
- 消息气泡 `role = LiveRegion.Polite` 让屏幕阅读器朗读新增消息（**仅在仅文字模式下生效**）
- 首启 / 设置页检测 `AccessibilityManager.isEnabled() && isTouchExplorationEnabled()` → 推荐仅文字模式

### 8.6 离线行为（[I10] 新增）

完全无网络 / 服务器宕机时 App 不应"崩溃式"显示：

| 状态 | App 行为 |
|---|---|
| 启动时检测无网 | 直接进入主界面（不卡首屏）；形象 idle 动画继续播放（VRM 已缓存） |
| 抽屉 | 显示本地缓存的会话列表 + "● 未连接" 红点；会话标题正常可点 |
| 历史浏览 | Room 缓存的最近 500 条/会话完全可读，超出无法翻页（显示"上滑加载历史 — 需要联网"） |
| 输入框 | 文字输入框置灰 + 占位符 "未连接服务器"；麦克风按钮置灰 |
| 顶栏 | 红色状态指示 "未连接 - 点击重试" |
| 网络恢复 | 自动重连 + 同步增量 history_page；红点变绿 + Snackbar "已连接" |

App 进入后台再回到前台：
- 若 WS 还活 → 继续；
- 若被系统杀 → 走"启动时检测"流程；
- 不主动唤醒 radio 保持长连接（[I8]）

---

## 9. 部署架构

### 9.1 monorepo 结构

```
hermes-Digital-Human/
├── README.md
├── DESIGN.md                          ← 本文档
├── LICENSE                            (MIT)
├── .github/
│   └── workflows/
│       ├── android.yml                (lint + unit test + APK build)
│       ├── gateway.yml                (pytest + ruff + ghcr 推送)
│       └── release.yml                (Tag → GitHub Release)
├── android/
│   ├── app/                           (Android 工程)
│   │   ├── build.gradle.kts
│   │   └── src/main/
│   │       ├── java/.../app/         (Compose UI / Room / WebSocket / STT)
│   │       ├── assets/default.vrm    (内置形象)
│   │       └── res/values-{zh,en,ja}/ (i18n)
│   ├── gradle/
│   └── settings.gradle.kts
│   # 注：unity-export 的 .aar 不入 git（[I3]）
│   #   - CI: 由 unity/ 工程在 Android workflow 内构建产出
│   #   - Local dev: 开发者自己 build_unity.sh 生成到 app/libs/
│   #   - Release: 单独的 unity-export.aar 上传到 GitHub Releases 作为 asset
├── unity/                             (Unity 工程，导出 .aar 后入 android/)
│   ├── Assets/
│   │   ├── Scenes/Main.unity
│   │   ├── Scripts/
│   │   │   ├── HDHBridge.cs          (Unity ↔ Android JNI)
│   │   │   ├── VRMLoader.cs
│   │   │   ├── BlendShapeDriver.cs
│   │   │   ├── AnimationStateMachine.cs
│   │   │   └── MixamoRetargeter.cs
│   ├── Plugins/UniVRM/
│   │   # 注：Mixamo 原始动画文件不入库（[CR2 D002] 法律合规）
│   │   #   - Mixamo Additional Terms 禁止 redistribute 原始动画
│   │   #   - v1 方案：tools/fetch_mixamo.sh 让用户用自己的 Adobe 账号下载到本地
│   │   #   - v2 方案：评估 CC0 替代库（Quaternius / open3d-models / Wesnoth animations）
│   └── ProjectSettings/
├── gateway/                           (hermes-voice-gateway, Python)
│   ├── pyproject.toml
│   ├── src/voice_gateway/
│   │   ├── __main__.py
│   │   ├── ws_server.py
│   │   ├── protocol.py                (帧 codec)
│   │   ├── frame_muxer.py
│   │   ├── tag_parser.py              (XML 状态机)
│   │   ├── stt/
│   │   │   ├── azure_proxy.py
│   │   │   └── minimax_proxy.py
│   │   ├── tts/
│   │   │   ├── edge_tts.py            (默认)
│   │   │   ├── azure_proxy.py
│   │   │   └── minimax_proxy.py
│   │   ├── viseme/
│   │   │   ├── pinyin_aligner.py     (Edge-TTS 配套)
│   │   │   └── azure_viseme.py
│   │   ├── hermes_bridge.py
│   │   ├── auth.py
│   │   ├── db.py                      (SQLite 模型)
│   │   ├── admin_cli.py               (gateway-admin)
│   │   └── bootstrap.py               (首启 + 二维码)
│   ├── tests/
│   └── Dockerfile
├── infra/
│   ├── docker-compose.caddy.yml
│   ├── docker-compose.cloudflare.yml
│   ├── Caddyfile.example
│   ├── hermes-config.yaml.example
│   ├── gateway-config.yaml.example
│   └── .env.example
└── docs/
    ├── deployment.md                  (部署手册)
    ├── byok-setup.md                  (BYOK 配置指南)
    ├── protocol-spec.md               (WS 协议详细规范)
    ├── vrm-spec.md                    (VRM 制作规范)
    └── troubleshooting.md
```

### 9.2 docker-compose 模板（Caddy 版）

见 `infra/docker-compose.caddy.yml`。结构如 §1 全景图所示。

### 9.3 首启 bootstrap 输出（[W15] 修订，带 PNG QR + deep link + ASCII fallback）

```
$ docker compose up -d
$ docker compose logs voice-gateway --follow

[voice-gateway] First-time setup detected.
[voice-gateway] Generated pairing code (BIP39 6 词): apple-river-tiger-cloud-music-bridge
[voice-gateway] Pairing code TTL: 300s (one-shot use)
[voice-gateway] Waiting for Hermes... ok (3s)
[voice-gateway] Configuring Hermes tool whitelist (yaml allow-list)... ok
[voice-gateway] Verified denylist active: terminal, file, cronjob, skills
[voice-gateway] Public endpoint: https://hdh-myname.trycloudflare.com
[voice-gateway]   ⚠️  Cloudflare Tunnel: audio/text traffic transits CF edge (TLS-protected)
[voice-gateway] PNG QR saved to /data/qr.png (1024x1024, scannable by phone)
[voice-gateway] Deep link (open in phone browser to auto-launch App):
[voice-gateway]   https://hdh-myname.trycloudflare.com/pair?code=apple-river-tiger-cloud-music-bridge

╭───── Android App Pairing (Terminal Fallback) ─────╮
│                                                    │
│   █▀▀▀▀▀█  ▀█▀ █  █▀▀▀▀▀█    (qrencode -t UTF8)   │
│   █ ███ █  ▄ █ █  █ ███ █                          │
│   █ ▀▀▀ █ ▀█▀ █▀  █ ▀▀▀ █                          │
│   ▀▀▀▀▀▀▀ █▀█ ▀▀▀ ▀▀▀▀▀▀▀                          │
│                                                    │
│  Manual input (6 words):                           │
│  apple river tiger cloud music bridge              │
│                                                    │
│  Host: hdh-myname.trycloudflare.com                │
│                                                    │
╰────────────────────────────────────────────────────╯

[voice-gateway] Listening on :8080, waiting for App connection...
[voice-gateway] Pairing endpoint: POST /v1/pair
```

**为何同时提供三种**：
- **PNG 二维码 (`/data/qr.png`)** — 用户在 SSH 时把这个文件下载到本地，用手机扫；识别率 ≈ 99%
- **Deep link** — 用户在手机浏览器打开链接，自动唤起 App 并预填配对码
- **终端 ASCII 二维码** — 仅在用户本机直接跑 docker compose 且眼前就是手机时用（识别率 30-50%）

### 9.4 管理命令

```
docker compose exec voice-gateway gateway-admin device list
docker compose exec voice-gateway gateway-admin device revoke <device_id>
docker compose exec voice-gateway gateway-admin device rotate-credential <device_id>
docker compose exec voice-gateway gateway-admin key rotate signing      # ed25519 签名密钥
docker compose exec voice-gateway gateway-admin key rotate byok-master  # BYOK 加密主密钥
docker compose exec voice-gateway gateway-admin pair                  # 重新打印二维码
docker compose exec voice-gateway gateway-admin conv list
docker compose exec voice-gateway gateway-admin conv delete <id>
docker compose exec voice-gateway gateway-admin status
docker compose exec voice-gateway gateway-admin logs --follow
docker compose exec voice-gateway gateway-admin byok list             # 列出 BYOK 设备
docker compose exec voice-gateway gateway-admin byok revoke <device>  # 撤销某设备 BYOK
```

---

## 10. 可观测性

### 10.1 日志

- 格式：JSON line（structlog）
- 字段：`ts`, `level`, `event`, `device_id`(部分哈希), `conv_id`, `latency_ms`, ...
- 默认级别：info
- debug 级别仅 50 字 prompt 预览 + 哈希
- BYOK key / 完整 prompt / 完整音频 永不记录

### 10.2 指标（Prometheus）

`GET /metrics` 暴露：

```
hdh_ws_connections_active                    Gauge
hdh_ws_frames_total{type=...,direction=...}  Counter
hdh_llm_ttft_seconds                          Histogram
hdh_tts_first_audio_seconds{provider=...}     Histogram
hdh_stt_duration_seconds{provider=...}        Histogram
hdh_e2e_latency_seconds                       Histogram
hdh_audio_queue_depth                         Gauge
hdh_backpressure_events_total                 Counter
hdh_errors_total{code=...}                    Counter
```

### 10.3 App 端（[I6] 路径修正）

- crash 自动保存到 **`Context.filesDir/crashes/`**（不是 cacheDir — cacheDir 在内存压力下会被系统清理，崩溃日志会丢）
- 7 天自动清理（防无限堆积）
- 设置页"导出诊断日志"按钮 → 打包近 100 条 WS 帧元数据 + crash log → 分享 sheet（用户主动决定发给谁）
- 不接 Sentry / Firebase / 任何第三方上报

---

## 11. 版本演进规则

### 11.1 三轨 SemVer

```
App版本:      与用户感知的功能/UI 相关。例 1.2.3
Gateway版本:  服务端实现版本。例 0.5.0
Protocol版本: 网络协议契约。例 HDH/1.0
```

App 和 Gateway 都通过 hello 帧声明自己 protocol_supported 列表，Gateway 选最高公共版本。

### 11.2 Breaking change 规则

- Protocol major 升级（HDH/1→HDH/2）：极慎重，需要：
  - 至少 1 个 minor 周期的 deprecation warning（旧字段标 deprecated）
  - GitHub Release notes 顶部红字 BREAKING
  - 旧 Gateway 镜像 :1 tag 永久保留，README 提示如何锁定旧版
- App 强制升级：仅在 Gateway 配置 `min_app_version` 上调时才发生（罕见）
- feature flag：所有非破坏性新功能在 hello_ack.server_features 报告，App 动态启停 UI

### 11.3 镜像 tag

```
ghcr.io/{org}/hermes-voice-gateway:latest       # 最新（不稳）
ghcr.io/{org}/hermes-voice-gateway:stable       # 推荐生产用
ghcr.io/{org}/hermes-voice-gateway:1            # 1.x 系列最新
ghcr.io/{org}/hermes-voice-gateway:1.2          # 1.2.x 最新
ghcr.io/{org}/hermes-voice-gateway:1.2.3        # 精确锁定
```

---

## 12. v1 范围围栏（不做清单）

### 明确不做（v1）

| 功能 | 原因 | 计划 |
|---|---|---|
| 真·语音 barge-in | AEC 深坑、Android 音频路由复杂 | v2 |
| 摄像头/视觉输入 | 隐私 + 复杂度（协议 0x10~0x12 帧类型已预留） | v2 |
| 唤醒词 / 免提模式 | 电池 + 复杂度 | v2 |
| 多用户托管模式 | 违反 D 策略原则 | 永不做 |
| 本地 LLM / GPU 利用 | GPU 闲置预留（§1 全景图已明确 v1 不接入） | v2 |
| App 内捏脸 / 修改 VRM | 让用户用 VRoid Studio | 永不做 |
| iOS 客户端 | 先验证 Android | v2 |
| 桌面客户端 | 同上 | v2 |
| 群聊 / 多人对话 | 数字人是一对一 | 永不做 |
| 付费 / 订阅 | MIT 开源项目 | 永不做 |
| 消息编辑 / 删除 / 撤回 | Q11 决策 | 永不做 |
| 跨服务器账号迁移 | D 策略下用户自己换 | 永不做 |
| **数字人主动发起对话** | 需要 Hermes `cron` 工具，**但 §4.3 工具白名单已禁用 cron**（[W10] 一致性修订）。v2 会重新评估是否在受限沙盒下开放 | v2 |
| 图片 / 文件上传 | 范围控制 | v2 |

### 围栏维护规则

任何 v1 期间出现的"加这个吧 / 顺便做一下"请求，先开 GitHub issue 标 `scope-creep`，由维护者评估是否真要进 v1，**默认不进**。

---

## 13. Roadmap（单人全职估算 — [W11 / WR14 D002] 再次修订）

D001 引入 BIP39 + Turn 租约 + §8.0 6 规则 + 无障碍 + schema migration + 双密钥管理 = +3w；D002 又引入原子 CAS + AudioTrack fallback + Hermes API 校验 + Mixamo 替换方案 = +1w。Roadmap 再次上调。

| 里程碑 | 时长（区间） | 验收标准 / 退出条件 |
|---|---|---|
| **v0.0 准备期** | 2~3w | VRoid Studio 默认 VRM 制作完成 + 三栈 POC 通过（R1）+ 真机采购到位 + Hermes API 摸底确认（CR1）+ Mixamo 替换方案选定（CR2）|
| **v0.1 internal alpha** | 5~6w | Hermes 接通；voice-gateway WS 骨架（aiohttp）；配对端点 `POST /v1/pair`（BIP39 IME 适配）；ed25519 密钥管理；Android Compose 壳；Unity-as-Library bridge（§8.0 6 规则）；AudioTrack.getTimestamp + fallback；纯文字端到端；Edge-TTS 默认链路；5 viseme 嘴型同步；client-side latency P50 < 2.5s 真机基准 |
| **v0.2 closed beta** | +4~5w | 合法 XML fragment + 合成根流式解析；Mixamo 9 动作 + 时长（用户本地下载脚本）；多会话 + 抽屉；Turn 租约原子 CAS + fencing token；schema migration v1；Cloudflare Tunnel 部署模板（PNG QR + deep link）|
| **v0.3 public beta** | +4~5w | BYOK TTS (Azure + MiniMax) + STT；首启 BIP39 配对（IME 适配 + 扫码）；设置页 5 段完整；中英日三语 i18n + viseme 对齐；离线降级；无障碍基线（仅文字模式 + TalkBack 兼容声明）|
| **v1.0 GA** | +4~5w | 协议版本握手 + feature 枚举；ghcr 镜像 + Caddy 模板；README/部署文档；DR runbook；真机性能基准（server-side P95 < 1200ms，client-side P95 待真机验证）；MIT + 三方许可合规审；SBOM；安全自查 |
| **合计** | **19~25w（≈ 5~6 个月）** | 单人全职、各项栈熟练 |

**真实时间** = 上限区间 × 熟练度系数：
- Python 5/5 + Android 5/5 + Unity 5/5 → 19~22w
- 任一项 3/5 → 25~35w
- 任一项 2/5 → 35~50w
- 任一项 1/5 → 建议先补技能再立项

**显式预算**：
- 真机采购：¥4000~8000（至少 Pixel 8 + 1 国行 ROM；推荐再加 Galaxy / 红米作机型矩阵）
- BYOK 测试费：Azure / MiniMax 各 ¥50 测试余额
- VRoid Studio 学习 + 默认 VRM 制作：2~4 周（含 v0.0）
- Mixamo 替代库评估 / 委托：¥0~3000
- CI：公开 repo 免费

---

## 14. 风险登记（OPEN — [I1] 修订：R1-R13 转为 milestone exit criteria）

按风险等级排序。**每项标注绑定的 milestone 退出条件**（不达成则该里程碑不发布）。

| 编号 | 风险 | 等级 | Milestone 退出条件 | 缓解 |
|---|---|---|---|---|
| R1 | 维护者技术栈熟练度 | 高 | v0.0 通过三栈 24h POC | Python WS server / Android Compose+UnityBridge / Unity UniVRM+BlendShape POC，任一失败 → 加补习时间或换栈方案 |
| R2 | Hermes 上游 API 不稳定 / Breaking | 中 | v0.1 锁定 hermes-agent docker tag 验证可用 | gateway 锁定 tag；监控 release notes；预案：fork 维护 |
| R3 | Edge-TTS 被微软关闭 | 高 | v0.3 piper-tts fallback 完成代码集成 | piper-tts (MIT) 替换路径预制；监控 API 状态 |
| R4 | VRoid → UniVRM 1.x 兼容性 | 低 | v0.0 默认 VRM 在 Unity 编辑器跑通 60fps | 准备 VRM 0.x fallback |
| R5 | Cloudflare Tunnel 限速影响音频 | 低 | v1.0 24kHz Opus 流稳定性 1h 压测通过 | 不行降级为自建 Caddy |
| R6 | Android ROM SpeechRecognizer 一致性 | 高 | v0.3 真机矩阵（Pixel + 小米 + 华为 + OPPO）测试通过 | 不行则在该机型上提示用户切 BYOK STT |
| R7 | Unity-as-Library APK 体积爆炸 | 中 | v1.0 release APK < 80MB | IL2CPP + Strip Engine Code + ASTC 纹理 |
| R8 | Mixamo 原始动画分发合规（D002 修正） | 中 | v1.0 LICENSE + NOTICE 文件不含 Mixamo 资产；`tools/fetch_mixamo.sh` 让用户本地下载 | Adobe Mixamo Additional Terms 禁止 redistribute，已从仓库移除；v2 评估替换为 CC0 库 |
| R9 | 公网未鉴权工具暴露（深度审查发现） | 已缓解 | v0.1 完成 §4 鉴权三件套 | device_credential + TLS + 工具白名单全部落地 |
| R10 | XML schema 漂移导致 TTS 卡流 | 中 | v0.2 流式 XML 解析容错降级 + 监控指标 | `hdh_xml_tag_malformed_total` 跨 5% 触发警告 |
| R11 | Hermes memory 写入字段控制能力（content_clean）| 中 | v0.2 验证 Hermes ingestion 字段语义 | 若 Hermes 不支持，降级为 GW 拦截 / 上游 PR |
| R12 | Hermes API 端点 / 工具名上游变更 | 中 | 锁定 hermes-agent docker tag | 监控上游 release notes；启动校验失败 → fatal exit |
| R13 | edge-tts 反向工程 API 关停 / PyPI 下架 | 高 | v0.3 piper-tts fallback 实现 + 配置开关 | LGPL 风险 + ToS 风险双管齐下；v2 必须替换 |

---

## 15. 决策日志（Decision Log）

任何对本文档的修订必须在此追加一行。

| 日期 | ID | 变更摘要 | 来源 |
|---|---|---|---|
| 2026-05-24 | D000 | 初版冻结（Q1~Q16 全部决策） | 16 轮设计访谈 |
| 2026-05-24 | D001 | 双模型审查综合修订：11 Critical + 19 Warning + 10 Info 落实到文档。**关键变更**：(a) §3.1 / §4.1 鉴权改为 BIP39 配对码 + device_credential JWT，禁止 URL token；(b) §3.3.1 新增帧封装格式 `[1B type][4B BE length][payload ≤ 1MB]` + 帧大小上限表 + 未知帧分级处理；(c) §3.4 ts_ms 零点改为 audio sample 实际播出时刻（AudioTrack.getTimestamp）；(d) §3.5 切片规则按语言分支 + viseme 对齐策略表；(e) §3.6 延迟预算重排 P50<1.8s / P95<2.5s + 包含 AudioTrack 硬件延迟 + Unity 渲染管线；(f) §3.7 打断立即静音 < 100ms；(g) §3.8 自适应 jitter buffer + 后台心跳节能；(h) §4.3 工具白名单改 yaml allow-list 机器可读；(i) §4.4 BYOK 用短效 upload_credential；(j) §4.5 限流/熔断；(k) §5.2 LLM 输出格式改合法 XML；(l) §6.2 加载时间 SLO + 持久化缓存；(m) §7.2 Turn 租约单活跃写入；(n) §8.0 Unity 嵌入策略；(o) §8.5 无障碍 / §8.6 离线行为；(p) §9.3 PNG QR + deep link 替代 ASCII；(q) §10.3 crash log filesDir；(r) §13 Roadmap 加 v0.0 准备期 + 区间预算；(s) §14 R1-R10 转 milestone exit criteria；(t) §17 术语表 | Codex + Claude 双模型审查 |
| 2026-05-25 | D002 | 第二轮双模型审查综合修订：8 Critical + 21 Warning + 13 Info 应用。**关键事实修正**：(a) §4.3 Hermes 真实 API 没有 `/v1/tools`，工具名是 `terminal/file/cronjob/skills/memory/session_search`（不是 `shell/cron/skill_create/memory_search`）；(b) §16 Mixamo 不允许 redistribute 原始动画，从仓库移除改用户本地下载脚本；(c) §16 edge-tts 是 LGPLv3 不是 GPL-3.0；(d) §4.1 BIP39 6 词表述准确化（"BIP39 wordlist 抽 6 词" ≠ "BIP39 mnemonic"）+ 配对码 normalize 规则。**协议正确性**：(e) §3.3.1 删除冗余 1B+4B length 头，依赖 WS RFC 6455；envelope 拆方向性（v 通用 / request_id 仅请求型 / ts_server_ms 仅下行 / 二进制帧豁免）；(f) §5.2 LLM 输出 XML fragment + GW 合成 `<stream>` root；§5.2.1 超时统一为 1500ms（慢 LLM 友好）；(g) §7.2 Turn 租约改原子 CAS + fencing token (lease_version) + 续约机制（防长 LLM 回答被 30s 误杀）。**Android 落地**：(h) §3.4 AudioTrack.getTimestamp 失效兜底（连续 3 次 false → AudioManager.getOutputLatency 估算降级）；(i) §8.4 BIP39 输入 IME 适配（强制英文键盘 / 不分 6 格 / TalkBack 推荐扫码）；(j) §8.5 TalkBack 与 Unity SurfaceView 不兼容的诚实声明，仅文字模式是盲人主路径；(k) §4.5 熔断改 per-provider 不全员踢线。**密钥与合规**：(l) §4.1.1 新增服务端密钥管理（生成/轮换/备份/灾难场景）；(m) §16.1 LGPL 影响重写。**Roadmap**：(n) §13 时间表上调至 19~25w + v0.0 含 Hermes API 摸底 + Mixamo 替换方案；(o) §14 新增 R11-R13 风险。**砍简过度工程化**：(p) §3.8 jitter buffer 固定 120ms（自适应留 v2）；(q) §3.7 重连退避 5 档（1/5/30/60/300）；(r) §8.3 无障碍砍掉 MToon 高对比 shader 变体；(s) §5.2.1 三层防御简化为单层 1500ms 超时 + few-shot。**§2 快照与残词清理**：(t) §2 决策快照同步 D001/D002 修订内容；(u) §9.4 token 命令改为 device 命令 | Codex + Claude 第二轮双模型审查 |
| 2026-05-25 | D003 | 第三轮双模型审查 minimal patch（仅修共识 Critical + 2 高 ROI Warning，然后**永久关闭整体复审**）：(a) **§3.4 XR1**：删除 `AudioManager.getOutputLatency()` hidden API 兜底方案（Play Store 上架会拒），改为静态路由表（speaker 80ms / A2DP 200ms / wired 40ms / earpiece 20ms / SCO 150ms / USB 50ms）+ `getPlaybackHeadPosition` 估算；蓝牙强制提示用户改扬声器。(b) **§5.3 XR2**：放弃假设的 Hermes skill 注册 API，改为每次 `/v1/chat/completions` 调用时注入 system message（OpenAI 兼容原生支持）；prompt 强化"禁止输出包裹元素"防 LLM 自加 `<reply>` 根；§5.2.1 状态机加防御。(c) **§4.3 XR3**：工具白名单按真实 Hermes **toolset** 命名（default / web_search / memory / session_search 允许；terminal / file / cronjob / skills 禁用），通过 `platform_toolsets.api_server.enabled_toolsets` 配置而非伪命令 `hermes config set`；v0.0 必须 grep 上游 toolsets.py 确认命名。(d) **§7.2 XR4**：补 renew 失败时 GW 行为（取消 Hermes upstream + 推 turn_state preempted + 不再下发后续帧）；明确单连接 aiosqlite 约束；audio_in_start 初始 lease 提到 90s 覆盖长录音。(e) **§7.1 W2**：conversation schema 加 `active_lease_version` 字段（D002 CAS 代码引用但 schema 未声明，首版 migration 必漏）。(f) **§2 W1**：决策快照同步残留旧口径（Mixamo CC-BY → "不入仓库" / 同 token → device_credential / 端到端 < 1.5s → server-P50 < 600ms 待真机）。(g) **§14 标题** R1-R8 → R1-R13。**双模型 r3 收敛裁决**：架构层硬伤已清干净，剩余 Warning/Info 都属 v0.1~v0.3 实施阶段就地修复范畴。**v0.0 准备期可立即启动**。| Codex + Claude 第三轮双模型审查 |

---

## 16. License

- 项目代码（android/、unity/、gateway/、infra/、docs/）：**MIT**
- 默认 VRM 模型（android/app/src/main/assets/default.vrm）：版权独立，作者署名见 VRM metadata，授权 commercial + redistribution
- **Mixamo 动画**（[CR2 / D002] 重大修正）：**不入仓库**，因 Adobe Mixamo Additional Terms 禁止重新分发原始动画文件。提供 `tools/fetch_mixamo.sh` 脚本让用户用自己的 Adobe 账号下载到本地构建路径。v2 评估替换为真正可重分发的动作库（如 Quaternius CC0）。
- Hermes Agent（上游依赖）：MIT
- UniVRM：MIT
- **edge-tts**（[WR1 / D002] 事实修正）：**LGPLv3**（不是之前文档说的 GPL-3.0；以 PyPI 当前 metadata 为准）
- 其他第三方依赖见各模块 pyproject.toml / build.gradle

### 16.1 LGPL 影响与 Edge-TTS 边界（[WR13 / D002] 重写）

**edge-tts 是 LGPLv3 Python 包，gateway 通过 `import edge_tts` 同进程调用**。GPL 多数解读会触发传染，**LGPL 比 GPL 宽松**——允许在不修改 LGPL 库本身的前提下，让闭源/不同许可的代码静态/动态链接它。我们的使用方式（pip install + import + 不修改 edge-tts 源码）符合 LGPLv3 的"动态链接 + 不修改库"豁免。

**当前状态**：
- gateway 整体 license 是 MIT，不需要变成 LGPL
- 但 docker 镜像内含 LGPL 组件，需遵守 LGPLv3 第 4 条：声明使用了 LGPL 库 + 提供替换该库的能力（pip 安装本身即满足）
- docker image 标签：`LABEL org.opencontainers.image.licenses="MIT AND LGPL-3.0-or-later"`
- README 顶部声明 + `NOTICE` 文件包含 edge-tts 署名

**风险仍存（R3 升级）**：
- edge-tts 通过反向工程 Microsoft Edge TTS API 实现，**Microsoft 服务条款不允许**这种用法；微软可随时关 API
- 包本身被 PyPI 下架的可能性也存在
- **v2 必须替换为 piper-tts (MIT)**，同时规避 LGPL + ToS 双风险
- v1 临时缓解：`edge-tts` pin 到具体版本，准备 piper-tts fallback 实现（v0.3 完成代码集成，仅未启用）

### 16.2 第三方依赖清单（SBOM 锚点）

正式 SBOM 见 `docs/SBOM.md`（v1.0 GA 前生成）。关键依赖摘要：

| 组件 | 类型 | License | 用途 |
|---|---|---|---|
| Hermes Agent | 上游 | MIT | LLM 智能体后端 |
| UniVRM | Unity 包 | MIT | VRM 加载 |
| MToon | Unity Shader | MIT | 卡通渲染（Universal Render Pipeline / URP） |
| **Mixamo 动画** | 资产 | Adobe 专有 | 9 个 humanoid 动作 — **用户本地下载，不入仓库** |
| aiohttp | Python | Apache 2.0 | WS server |
| pypinyin | Python | MIT | 中文 viseme |
| pyopenjtalk | Python | BSD | 日文 viseme |
| g2p_en | Python | Apache 2.0 | 英文 viseme |
| **edge-tts** | Python | **LGPLv3**（v2 替换为 piper-tts MIT） | 默认 TTS |
| Compose | Android | Apache 2.0 | UI |
| Room | Android | Apache 2.0 | 本地缓存 |
| Silero VAD | ONNX | MIT | 客户端 VAD |
| Opus | C 库 | BSD-3 | 音频编码 |
| Fernet (cryptography) | Python | Apache 2.0 / BSD | BYOK key 加密 |
| PyNaCl (ed25519) | Python | Apache 2.0 | device_credential 签名 |
| structlog | Python | Apache 2.0 / MIT | 结构化日志 |

---

## 17. 术语表（[I2] 新增）

为消除文档内部"shorthand 黑话"，全部缩写在此显式定义。

| 术语 | 含义 | 状态 |
|---|---|---|
| **D 档** | 数字人四档分级中的最高档：3D 形象 + 全身动作 + 表情 + 唇形（vs A=纯文字 / B=2D Live2D / C=2D+精细口型） | 已确认 |
| **D 策略** | 部署策略：一人一服务器，用户自己装 Hermes + voice-gateway（vs A=纯自用 / B=家庭共享 / C=多租户托管） | 已确认 |
| **α 用户** | 目标用户分级中的最高技术水平：开发者 / 极客，会用 Docker、能改 .env、能读 README（vs β=轻技术 / γ=普通人） | 已确认 |
| **HDH** | hermes-Digital-Human 简写，也是协议名 `HDH/1.0` | 已确认 |
| **BYOK** | Bring Your Own Key，用户自带云服务 API key（区别于服务端默认免费 provider） | 已确认 |
| **voice-gateway** | hermes-voice-gateway 简写，新增的中间服务组件 | 已确认 |
| **device_credential** | 设备级长期凭证，ed25519 JWT，配对时签发，默认 180 天有效 | 已确认 |
| **pairing_code** | BIP39 6 词一次性配对码，5 分钟有效 | 已确认 |
| **Turn 租约** | 同一 conv_id 上的单活跃写入锁，30s 超时自动释放 | 已确认 |
| **viseme** | 视位 / 嘴型同步用的音素抽象（A/I/U/E/O 简化集 或 ARKit 52 详细集） | 已确认 |
| **VRM** | Virtual Reality Model，VR 角色 3D 模型标准格式 | 已确认 |
| **MToon** | VRM 自带的卡通渲染 shader | 已确认 |
| **Spring Bone** | 用于头发/裙摆物理摆动的简化骨骼模拟 | 已确认 |
| **idle / nod / shake_head / thinking / pointing / wave / shrug / lean_forward / lean_back** | v1 冻结的 9 个 Mixamo 动作（见 §5.1） | 已确认 |

**状态标识**：
- `已确认` — Q1~Q16 决策 + D000/D001/D002/D003 修订均明确
- `待验证` — 标记到 §14 风险登记中的 OPEN 项
- `计划中` — §12 v2 列表中的功能

---

## 18. 联系

- GitHub: https://github.com/{org}/hermes-Digital-Human
- Issue: https://github.com/{org}/hermes-Digital-Human/issues
- 文档：https://github.com/{org}/hermes-Digital-Human/tree/main/docs

---

**本文档初版冻结于 2026-05-24，对应决策集 D000。**
**双模型审查综合修订于 2026-05-24，对应决策集 D001（11 Critical + 19 Warning + 10 Info）。**
**第二轮双模型审查综合修订于 2026-05-25，对应决策集 D002（8 Critical + 21 Warning + 13 Info；含 Hermes API / Mixamo / edge-tts license 等事实性修正）。**
**第三轮双模型审查 minimal patch 于 2026-05-25，对应决策集 D003（4 共识 Critical + 2 高 ROI Warning + Info 同步）。双模型 r3 收敛裁决：架构层已清干净，永久关闭整体复审循环，进入 v0.0 准备期。**
**任何后续修订必须更新 §15 决策日志。**
