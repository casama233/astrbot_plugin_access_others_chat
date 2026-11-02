# access_others_chat_history

[简体中文](README.md) · [繁體中文](README.zh-TW.md) · **English**

An AstrBot plugin providing **cross-conversation chat history access** to the language model, so it can consult context from other sessions or platforms.

## Features

- **Automatic private-to-group context injection (v1.1.0):** While a user talks to the bot in a group, optionally attach the **same sender's** recent DM history with that bot to the current request. No explicit tool call is required.
- Query the latest N private messages for a user or group messages for a group.
- Support adapters such as WeChat `weixin_qty`, `webchat`, and OneBot `aiocqhttp`.
- Register an LLM tool for on-demand history lookup.

## How it works

1. **Automatic injection:** `@filter.on_llm_request` reads the same sender's recent private chat before a group LLM request (default **10 messages / 2,000 characters**). It renders a "memory reference" block at the end of this request's context as `role="system"`.
2. **On-demand tool:** `@filter.llm_tool` exposes `access_others_chat_history` for lookups selected by the LLM.

The injected block has an `_no_save` marker; AstrBot's `llm_request.py` `_save_to_history` logic excludes it from the saved group conversation, so it should not accumulate across turns.

## Configuration (AstrBot WebUI → Plugin settings)

| Setting | Type | Default | Meaning |
| --- | --- | --- | --- |
| `inject_enabled` | bool | `true` | Master switch for automatic DM-to-group injection |
| `inject_user_whitelist` | list | `[]` | Limit to listed sender `user_id` values; **empty means all users** |
| `inject_max_messages` | int | `10` | Number of DM messages, range 1–50 |
| `inject_max_chars` | int | `2000` | Maximum injected text length |

> **Privacy warning:** The bot's group responses are visible to everyone in the group. They could indirectly disclose private-chat details. Restrict `inject_user_whitelist` to trusted users before enabling this capability in sensitive settings.

## Tool usage

The `subject_id` may be a full UMO such as `weixin_qty:FriendMessage:o9cq8...` (recommended), or a shortened `o9cq8...` in the default platform (automatically prefixed with `default:FriendMessage:`).

UMO (Unified Message Origin) uses `{platform_id}:{message_type}:{session_id}`, matching AstrBot `MessageSession`.

| Adapter | Platform ID | Example private UMO |
| --- | --- | --- |
| WeChat | `weixin_qty` | `weixin_qty:FriendMessage:o9cq8...` |
| WebChat | `webchat` | `webchat:FriendMessage:qty!uuid...` |
| OneBot / QQ | `aiocqhttp` | `aiocqhttp:FriendMessage:123456789` |
| Telegram | `telegram` | `telegram:FriendMessage:123456789` |

The platform ID can be taken from `event.unified_msg_origin`.

Use `isGroup=false` for private history (`FriendMessage`) or `isGroup=true` for group history (`GroupMessage`). `length` defaults to **20** (range 1–100).

For example, a group user asks the bot about a poem discussed privately. The plugin can attach that user's 10 latest DMs. The LLM can also call:

```text
access_others_chat_history(isGroup=false, subject_id="weixin_qty:FriendMessage:o9cq808...", length=10)
```

## Internals and files

The plugin accesses `self.context.conversation_manager` using `get_curr_conversation_id(uid)` and `get_conversation(uid, cid)`. Results filter out non-`user`/`assistant` roles and non-text data such as images. See `docs/PLAN.md` and `docs/SPEC.md` for source-level design.

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

## License and upstream

Licensed under **AGPL-3.0**; follow its corresponding-source obligations when modifying and deploying. This repository is a fork of [KonmaKanSinPack/astrbot_plugin_access_others_chat](https://github.com/KonmaKanSinPack/astrbot_plugin_access_others_chat). Preserve upstream copyright and notices. Fork modifications are not represented as original upstream work, and the repository license does not extend to third-party assets, data or dependencies.
