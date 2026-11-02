# access_others_chat_history

[简体中文](README.md) · **繁體中文** · [English](README.en.md)

讓 AstrBot 的大型語言模型能**跨會話存取聊天紀錄**，在與你對話時取得其他平台／會話的歷史上下文，提供跨平台記憶參照。

## 功能

- **群聊自動參照私聊（v1.1.0）**：同一使用者在群聊向 bot 發話時，插件可在該次 LLM 請求中自動注入他與 bot 的近期私聊，不必由模型主動呼叫工具。
- 讀取指定使用者最近 N 則私聊，或指定群組最近 N 則訊息。
- 支援微信 `weixin_qty`、`webchat`、OneBot `aiocqhttp` 及其他平台適配器。
- 註冊工具供模型按需呼叫。

## 運作方式

1. **自動注入**：`@filter.on_llm_request` 在群聊 LLM 請求送出前，讀取**同一發話者**與該 bot 的最近私聊（預設 **10 則／2,000 字元**），把它標為「記憶參考」並以 `role="system"` 注入該次請求的上下文末尾。
2. **按需工具**：`@filter.llm_tool` 提供 `access_others_chat_history`，讓模型在有需要時讀取其他會話紀錄。

注入區塊附帶 `_no_save` 標記；AstrBot 的 `llm_request.py` `_save_to_history` 流程會排除它，不會寫回群聊對話歷史，也不會逐輪累積。

## 設定（AstrBot WebUI → 插件設定）

| 欄位 | 類型 | 預設 | 說明 |
| --- | --- | --- | --- |
| `inject_enabled` | bool | `true` | 群聊自動注入總開關 |
| `inject_user_whitelist` | list | `[]` | 只對列出的使用者 `user_id` 注入；**空白表示所有使用者** |
| `inject_max_messages` | int | `10` | 私聊訊息數上限，1–50 |
| `inject_max_chars` | int | `2000` | 每次注入的文字字元上限 |

> **隱私警告：** 群聊中的 bot 回覆對全部群成員可見，可能間接揭露私聊資訊。請只對可信賴的使用者啟用 `inject_user_whitelist`，避免意外公開私密上下文。

## 工具用法

`subject_id` 可使用完整 UMO（建議）`weixin_qty:FriendMessage:o9cq8...`，或在預設平台使用簡寫 `o9cq8...`（自動補上 `default:FriendMessage:`）。

UMO（Unified Message Origin）格式為 `{platform_id}:{message_type}:{session_id}`，對應 AstrBot 的 `MessageSession`。

| 適配器 | 平台 ID | 私聊 UMO 範例 |
| --- | --- | --- |
| 微信 | `weixin_qty` | `weixin_qty:FriendMessage:o9cq8...` |
| WebChat | `webchat` | `webchat:FriendMessage:qty!uuid...` |
| OneBot／QQ | `aiocqhttp` | `aiocqhttp:FriendMessage:123456789` |
| Telegram | `telegram` | `telegram:FriendMessage:123456789` |

平台 ID 可從 `event.unified_msg_origin` 取得。

`isGroup=false` 查私聊（`FriendMessage`）；`isGroup=true` 查群聊（`GroupMessage`）。`length` 預設 20，範圍 1–100。

例如，群聊使用者問「@bot 剛剛私聊提到的那首詩如何？」時，插件可自動注入此人最近 10 則私聊。模型亦可自行呼叫：

```text
access_others_chat_history(isGroup=false, subject_id="weixin_qty:FriendMessage:o9cq808...", length=10)
```

## 技術與檔案

透過 `self.context.conversation_manager`，依次使用 `get_curr_conversation_id(uid)` 和 `get_conversation(uid, cid)`。歷史會過濾非 `user`／`assistant` 角色及圖片等非文字內容。來源層設計見 `docs/PLAN.md`、`docs/SPEC.md`。

```text
astrbot_plugin_access_others_chat_history/
├── main.py
├── history_utils.py
├── _conf_schema.json
├── docs/PLAN.md
├── docs/SPEC.md
├── tests/test_inject.py
├── README.md
├── CHANGELOG.md
├── metadata.yaml
└── LICENSE
```

## 許可與上游來源

採用 **AGPL-3.0**。修改及部署時應遵守 AGPL 的對應原始碼義務。本倉庫是 [KonmaKanSinPack/astrbot_plugin_access_others_chat](https://github.com/KonmaKanSinPack/astrbot_plugin_access_others_chat) 的 fork；保留上游授權與著作權聲明，分支修改不冒認為原作者原作，也不延伸授權到第三方資產、資料及相依套件。
