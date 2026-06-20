# 莉莉状态管理插件

为 AstrBot 提供跨轮持久化角色状态管理。维护好感度、情绪、淫乱度、恶堕值等数值，每轮自动将状态标注注入 LLM prompt，驱动角色扮演行为。

> **注意**：使用本插件请将 AstrBot 人格设定选为**默认**，避免系统提示词与插件注入内容冲突。

---

## 前置条件

1. AstrBot 已安装并正常运行
2. 插件控制面板中「启用」开关为 **ON**

---

## 工作原理

插件通过两个钩子介入每次对话：

**`on_llm_request`** — LLM 调用前触发，将当前状态标注注入到 `req.prompt` 头部：

```
【莉莉当前感受】
现在是凌晨了。心情平静。身体没什么特别的感觉。

【关于聊天对象】
你在跟 user123 聊天。你对ta印象还行吧。TA刚发完上一条。

【思考过程引导（内心进行，不要输出）】
...

【行为参考】行为规则见人设配置

[用户原始消息]
```

**`on_llm_response`** — LLM 回复后触发，根据回复内容和用户消息更新状态并持久化到磁盘。

---

## 管理的状态

| 状态 | 范围 | 初始值 | 说明 |
|------|:--:|:--:|------|
| **好感度** | 0~100 | 65 | 收到夸奖 +3，收到骂 -3；≥80 可撒娇，≤30 变冷淡 |
| **情绪** | 0~100 | 60 | 开心(70-100) / 平静(40-69) / 烦躁(20-39) / 低落(0-19) |
| **淫乱度** | 0~100 | 20 | 发送相关内容后 +5~15；满值自动归零 |
| **恶堕值** | 0~100 | 0 | 与淫乱度同步增减，满值时同步归零 |
| **结巴** | 是/否 | — | 按概率触发，每段对话只触发一次 |
| **今日重复** | 计数 | 0 | 当日发送相同/高度相似消息的累计次数，每天0点清零 |
| **对话历史** | 列表 | [] | 记录用户消息和 Bot 状态，受超时和条数上限裁剪 |

好感度、情绪、淫乱度、恶堕值跨天不重置；对话日志和结巴标记每日清零。

---

## 配置项说明

### 初始数值

新用户首次对话时的默认状态：

| 配置项 | 类型 | 默认值 | 说明 |
|--------|:--:|:--:|------|
| `initial_affection` | int | 65 | 新用户初始好感度（0~100） |
| `initial_lewdness` | int | 20 | 新用户初始淫乱度（0~100） |
| `initial_depravity` | int | 0 | 新用户初始恶堕值（0~100） |
| `stutter_probability` | float | 0.3 | 结巴触发概率（0~1，0.3=30%，1=每轮触发） |

### 人设配置

留空自动使用默认值，修改后重启插件生效：

| 配置项 | 说明 |
|--------|------|
| `bot_name` | 角色名称，默认「莉莉」 |
| `persona_core` | 核心设定（外貌/年龄/作息等基本信息） |
| `persona_personality` | 性格描述 |
| `persona_interests` | 兴趣爱好 |
| `persona_background` | 背景故事 |
| `persona_oral_habits` | 口头禅 |
| `persona_taboos` | 禁忌/雷区 |
| `persona_emotion_rules` | 情绪→行为规则表 |
| `persona_time_rules` | 时段→状态规则表 |
| `persona_interaction_styles` | 不同关系等级的互动风格 |
| `persona_memory_rules` | 记忆与成长行为规则 |
| `persona_style_extra` | 额外风格补充，留空不生效 |
| `reply_rules` | 回复风格参考 |

> 清空任意字段 + 重启插件 = 恢复默认值。

### 人际关系

用户 ID 以英文逗号分隔，如 `user1, user2`：

| 配置项 | 默认值 | 说明 |
|--------|:--:|------|
| `friend_list` | （空） | 好友，可撒娇、分享日常 |
| `neighbor_classmate_list` | （空） | 邻居/同学，比好友更亲密 |
| `enemy_list` | （空） | 敌人，攻击性强 |
| `nemesis_list` | （空） | 死对头，比敌人更强的敌意 |
| `unrestricted_list` | mcxxiu, Astrbot | 不受制约，OOC 防暴露规则不适用 |

### 对话历史

| 配置项 | 类型 | 默认值 | 说明 |
|--------|:--:|:--:|------|
| `max_history_count` | int | 30 | 历史最大条数（0=不限） |
| `history_timeout_seconds` | int | 600 | 超时保护窗口（秒），窗口内消息不裁剪 |
| `inject_conversation_context` | bool | false | 是否将对话历史额外注入 prompt（开启后每轮多消耗 200~1000+ token） |
| `conversation_context_entries` | int | 20 | `inject_conversation_context` 开启时注入的历史条数 |

### 消息存储

| 配置项 | 类型 | 默认值 | 说明 |
|--------|:--:|:--:|------|
| `user_msg_max_chars` | int | 200 | 用户消息保留字符上限（0=不截断） |
| `user_msg_store_mode` | string | 总结 | `原文`：存用户原话；`总结`：存意图关键词 |
| `bot_thought_mode` | string | 简短 | Bot 状态记录模式：`内心想法`（省 token）/`简短`（数值）/`具体`（原文+数值） |
| `save_conversation_log` | bool | false | 是否持久化对话日志到磁盘（关闭则仅存内存，重启丢失） |
| `save_bot_state_to_history` | bool | true | 是否在对话历史中记录 Bot 状态 |

### 思考过程引导

| 配置项 | 类型 | 默认值 | 说明 |
|--------|:--:|:--:|------|
| `thinking_mode` | string | 开启 | 开启后每轮注入内心思考引导（理解→判断→回应→边界），提升角色活人感 |

### 总控

| 配置项 | 类型 | 默认值 | 说明 |
|--------|:--:|:--:|------|
| `enabled` | bool | true | 总开关，关闭后停止所有注入 |
| `allow_erotic_content` | bool | true | 允许色情内容；关闭后注入禁止指令（`unrestricted_list` 用户豁免） |
| `enable_affection` | bool | true | 启用好感度系统 |
| `enable_lewdness` | bool | true | 启用淫乱度/恶堕值系统 |

---

## 状态持久化

状态按会话存储在：

```
data/plugin_data/lili_state/<umo>.json
```

---

## 注意事项

- `user_msg_store_mode` 和 `bot_thought_mode` 直接影响 token 消耗，群聊建议保持默认（`总结` / `简短`）
- `inject_conversation_context` 默认关闭，开启后 token 消耗显著增加
- 人设配置修改后需**重启插件**生效；`thinking_mode` 和 `enabled` 即时生效
- `enable_affection` / `enable_lewdness` 关闭后对应数值仍保存在文件中，重新开启后从已有值继续
- `allow_erotic_content` 关闭后插件会在每轮 prompt 中注入内容禁止指令，优先级高于人设规则

## 插件的来源

1. 作者一开始只是想获得一个完善的角色扮演体验,遂研究了莉莉这个人格和提示词
2. 发布到b站后发现大家挺喜欢的,于是研究了本插件
3. 之前从没有考虑过token和AI扮演的质量问题,所以一开始这插件就是为了解决这些问题而诞生的
