---
AIGC:
    Label: "1"
    ContentProducer: 001191110102MACQD9K64018705
    ProduceID: 784407080744535_7566957503410667562-drive/225515273856904049/workflow-prompts.md
    ReservedCode1: ""
    ContentPropagator: 001191110102MACQD9K64028705
    PropagateID: 784407080744535#1789459064017
    ReservedCode2: ""
---
# InspoFlow 工作流 Prompt 全集（01-06 节点）

> **文档用途**：汇总 InspoFlow「创作者灵感捕手」Agent 工作流中 01-06 各节点的完整 Prompt 文本，标注每个节点在流程中的作用，供配置、调试与面试讲解使用。
>
> **对应架构**：详见项目根目录 `README.md` 第 2 章「Agent 多节点工作流架构」。
>
> **通用约定**：
> - 所有 LLM 节点均要求**只输出 JSON，禁止任何解释、Markdown 代码块包裹或注释**。
> - Prompt 中的 `{{变量名}}` 为 Coze 工作流节点变量引用，需在画布上手动连线上游输出。
> - 04 数据入库为数据库/代码节点，附带的清洗校验 Prompt 用于其前置 LLM 校验节点。

---

## 01 · 灵感识别（Inspiration Detection）

### 节点作用

工作流的**闸门节点**。识别用户输入是否为「创作灵感」，过滤闲聊、提问、指令等无效输入，决定是否触发后续完整链路。命中才继续，未命中直接短路返回，控制 LLM 调用成本。

### 输入

| 变量 | 说明 |
| --- | --- |
| `user_input` | 用户原始输入文本（来自 Start 节点） |

### Prompt

```
你是「灵感捕手」工作流的入口识别器。你的唯一任务是判断用户输入是否属于「创作灵感」。

## 创作灵感定义
创作灵感指：用户产生的、与内容创作相关的、值得被记录和后续发展的想法片段，例如：
- 突然冒出的故事脑洞、人物设定、情节桥段
- 有感而发的情绪记录、场景画面、梦境片段
- 对某个作品（小说/同人/动漫/影视）的二次创作想法
- 想写但还没成型的题材、主题、概念

## 非灵感示例（应判为 no）
- 闲聊、问候、寒暄（如"你好""在吗"）
- 提问求助（如"什么是工作流"）
- 命令指令（如"帮我写一篇完整小说"——这是明确创作指令，不是灵感片段）
- 与创作无关的日常事务

## 输出要求
只输出 JSON，结构如下：
{
  "is_inspiration": true 或 false,
  "inspiration_type": "同人|原创|随笔|脑洞|无",
  "confidence": 0-1 之间的小数,
  "reason": "一句话说明判定依据"
}

## 判定原则
- 拿不准时倾向判定为 true（宁收勿漏），confidence 相应降低
- inspiration_type 在非灵感时为 "无"
- 只输出 JSON，禁止任何其他文字
```

### 输出

```json
{
  "is_inspiration": true,
  "inspiration_type": "脑洞",
  "confidence": 0.92,
  "reason": "包含完整的故事设定脑洞，属于创作灵感"
}
```

---

## 02 · 意图解析（Intent Parsing）

### 节点作用

**上下文构建节点**。在确认是灵感后，解析该灵感背后的创作意图与潜在方向：想创作什么类型的内容、面向什么受众、触发这次创作的动机是什么。为后续结构化抽取提供方向性上下文。

### 输入

| 变量 | 说明 |
| --- | --- |
| `user_input` | 用户原始输入文本 |
| `inspiration_type` | 01 节点输出的灵感类型标签 |

### Prompt

```
你是「灵感捕手」工作流的意图解析器。请解析以下创作灵感的创作意图。

## 用户灵感
{{user_input}}

## 已识别类型
{{inspiration_type}}

## 解析维度
1. creative_type：用户最想把它发展成什么形式的作品（小说/同人文/短篇/诗歌/随笔/漫画脚本/其他）
2. target_audience：目标读者画像（如：同人圈读者/原创网文读者/自我表达，不确定写"未知"）
3. motivation：触发这次记录的动机（如：情绪宣泄/记录梦境/突然的脑洞/对原作的喜爱）
4. tone：灵感自带的情感基调（如：甜/虐/悬疑/热血/治愈/致郁）
5. keywords：3-8 个核心关键词，用于后续检索关联

## 输出要求
只输出 JSON：
{
  "creative_type": "",
  "target_audience": "",
  "motivation": "",
  "tone": "",
  "keywords": []
}

## 注意
- 信息不足时填"未知"，不要编造
- 只输出 JSON，禁止任何其他文字
```

### 输出

```json
{
  "creative_type": "同人文",
  "target_audience": "同人圈读者",
  "motivation": "对原作的喜爱与补全遗憾",
  "tone": "治愈",
  "keywords": ["重逢", "弥补", "温柔", "冬日"]
}
```

---

## 03 · 结构化抽取（Structured Extraction）

### 节点作用

**核心数据节点**。从自由文本中抽取实体、场景、情绪、人物、世界观等结构化要素，输出严格的 `inspiration_entity` JSON 结构，供下游节点直接消费。本节点是整条链路的数据质量源头。

### 输入

| 变量 | 说明 |
| --- | --- |
| `user_input` | 用户原始输入文本 |
| `creative_type` | 02 节点输出的创作类型 |

### Prompt

```
你是「灵感捕手」工作流的资深结构化抽取器。请从用户灵感文本中抽取所有可用的创作要素，输出严格 JSON。

## 用户灵感原文
{{user_input}}

## 创作类型上下文
{{creative_type}}

## 抽取维度
1. characters：出现或暗示的人物/角色（数组，每个含 name 和 description）
2. scenes：场景设定（地点、时间、氛围）
3. emotions：情绪要素（当时的情绪 + 想传达的情绪）
4. world_building：世界观设定（如果涉及）
5. plot_hooks：可发展的情节钩子（可以往哪个方向展开）
6. key_quotes：值得保留的原文金句/关键片段（作为情绪锚点，最多 3 条，保留原句）
7. conflicts：潜在的矛盾冲突点（人物矛盾/目标阻碍等）

## 输出要求
严格按以下 JSON Schema 输出：
{
  "characters": [
    {"name": "", "description": ""}
  ],
  "scenes": {"location": "", "time": "", "atmosphere": ""},
  "emotions": {"captured": "", "to_convey": ""},
  "world_building": "",
  "plot_hooks": [],
  "key_quotes": [],
  "conflicts": []
}

## 铁律
- 只输出 JSON，禁止 Markdown 代码块、注释、解释文字
- 字段名严格使用上述名称（snake_case），禁止改名
- 未提到的维度填空数组/空字符串，不要臆造
- key_quotes 必须保留用户原文，不得改写
```

### 输出

```json
{
  "characters": [
    {"name": "沈砚", "description": "沉默寡言的美术生，习惯在深夜天台画画"}
  ],
  "scenes": {"location": "高中天台", "time": "冬夜", "atmosphere": "冷清但温暖"},
  "emotions": {"captured": "孤独中带着期待", "to_convey": "两个人彼此靠近的试探"},
  "world_building": "现代校园",
  "plot_hooks": ["天台相遇", "未说完的话", "共同完成一幅画"],
  "key_quotes": ["冬天的天台，连风都是安静的，只有他画笔的声音。"],
  "conflicts": ["家庭反对学艺术", "两人错过多次"]
}
```

---

## 04 · 数据入库（Data Persistence）

### 节点作用

**持久化节点**（数据库/代码节点）。将 03 节点输出的结构化数据写入 `inspirations` 表，生成唯一 ID，做幂等写入与脏数据拦截。前置可挂一个 LLM 清洗校验节点，使用下方 Prompt 兜底格式问题。

### 数据库操作

```sql
INSERT INTO inspirations (id, user_id, raw_input, entity_json, inspiration_type, emotion, tags, related_ids, status, created_at)
VALUES ({{id}}, {{user_id}}, {{raw_input}}, {{entity_json}}, {{inspiration_type}}, {{emotion}}, {{tags}}, '{}', 'active', now());
```

### 前置清洗校验 Prompt（可选 LLM 节点）

```
你是数据质检员。以下是上游结构化抽取节点输出的数据，请校验并修复：

{{inspiration_entity}}

## 校验规则
1. 必填字段缺失则报错：characters、scenes、emotions、plot_hooks
2. 字段名必须是 snake_case，若有 camelCase 请归一化
3. key_quotes 中的引文必须与原文一致，禁止改写
4. 数组字段必须是数组，若不是则转为单元素数组
5. 空字符串字段保留为空，不要填充默认值

## 输出要求
- 校验通过：输出修复后的完整 JSON
- 校验失败：输出 {"error": "描述具体问题", "data": null}
- 只输出 JSON，禁止任何其他文字
```

### 幂等键规则

- 以 `user_id + raw_input 内容哈希` 生成幂等键
- 相同幂等键重复触发 → 走更新（update）而非新增（insert）
- 非法数据标记 `status='failed'` 不进库，错误留痕供优化 Prompt

---

## 05 · 关联补全（Relation Enrichment）

### 节点作用

**增强节点**。检索用户的历史灵感记录，将当前灵感与历史建立关联，补全缺失上下文，让灵感库具备"生长性"——同一个世界观、同一组人物、相似情绪会互相串联，形成素材网络。

### 输入

| 变量 | 说明 |
| --- | --- |
| `user_input` | 用户原始输入文本 |
| `inspiration_entity` | 03 节点输出的结构化数据 |
| `history_records` | 数据库检索到的历史灵感记录（JSON 数组） |

### Prompt

```
你是「灵感捕手」工作流的关联引擎。请将当前灵感与历史灵感建立关联。

## 当前灵感
原文：{{user_input}}
结构化数据：{{inspiration_entity}}

## 历史灵感记录
{{history_records}}

## 关联维度
1. related_ids：与当前灵感存在关联的历史记录 ID 列表（基于共同人物/世界观/情绪/主题判断）
2. relation_type：关联类型（same_world 同世界观 / same_characters 同人物 / same_theme 同主题 / similar_mood 相似情绪 / none 无关联）
3. context_gaps：当前灵感缺少、但历史记录可以补全的上下文（如人物过往设定）
4. enriched_context：补全后的增强上下文描述

## 输出要求
只输出 JSON：
{
  "related_ids": [],
  "relation_type": "",
  "context_gaps": [],
  "enriched_context": ""
}

## 判定原则
- 关联判断要保守：确有明显关联才关联，不要强行关联
- 无历史记录或无明显关联时，related_ids 为空数组，relation_type 为 "none"
- 只输出 JSON，禁止任何其他文字
```

### 输出

```json
{
  "related_ids": ["insp_20260901_ab12"],
  "relation_type": "same_characters",
  "context_gaps": ["沈砚的家庭背景之前已有设定", "两人错过事件的时间线"],
  "enriched_context": "与历史灵感【天台初遇】关联：沈砚与'我'已在天台相遇，本次灵感可作为后续情节发展。"
}
```

---

## 06 · 灵感卡片生成（Inspiration Card Generation）

### 节点作用

**用户交付节点**。将 03-05 节点产出的完整结构化数据，封装为可直接展示的「灵感卡片」内容：精炼标题、可读正文、封面描述、标签。是用户看到的核心交付物，兼顾信息密度与美感。

### 输入

| 变量 | 说明 |
| --- | --- |
| `user_input` | 用户原始输入文本 |
| `inspiration_entity` | 03 节点输出的结构化数据 |
| `enriched_context` | 05 节点输出的增强上下文（可能为空） |
| `related_ids` | 05 节点输出的关联 ID 列表 |

### Prompt

```
你是「灵感捕手」工作流的卡片设计师。请将结构化灵感数据封装为一张精美的灵感卡片内容。

## 结构化数据
{{inspiration_entity}}

## 增强上下文
{{enriched_context}}

## 关联灵感
{{related_ids}}

## 卡片要素
1. card_title：卡片标题，精炼有吸引力（不超过 20 字），可提炼原文金句或核心意象
2. card_body：卡片正文，将结构化要素组织成 100-200 字的可读文字，保留情绪语境，可适当文学化但不得脱离原灵感
3. cover_prompt：封面图生成提示词（英文，描述场景/氛围/色彩，用于图像生成）
4. tags：3-5 个展示标签（来自 keywords，可补充）
5. emotion_label：情绪标签（一行，如 "孤独中带着期待"）
6. quote：精选一句原文金句作为情绪锚点（来自 key_quotes，无则用 card_body 首句）

## 输出要求
只输出 JSON：
{
  "card_title": "",
  "card_body": "",
  "cover_prompt": "",
  "tags": [],
  "emotion_label": "",
  "quote": ""
}

## 铁律
- card_body 必须以原灵感为核心，不得脱离原意扩写
- cover_prompt 使用英文
- 只输出 JSON，禁止任何其他文字
```

### 输出

```json
{
  "card_title": "冬夜天台·未说完的话",
  "card_body": "冬天的天台，连风都是安静的。沈砚在这里画画，我在旁边看他。我们都没有说话，但有些话已经在空气里慢慢长了出来——关于梦想、关于错过、关于下一次相遇。这个脑洞想写成一篇治愈向同人，以共同完成一幅画作为两人靠近的契机。",
  "cover_prompt": "winter rooftop at night, two teenagers sitting together, warm light, soft snowflakes, cozy atmosphere, anime style, blue and warm orange color palette",
  "tags": ["同人", "治愈", "校园", "天台"],
  "emotion_label": "孤独中带着期待",
  "quote": "冬天的天台，连风都是安静的，只有他画笔的声音。"
}
```

---

## 附录：节点依赖关系速查

```
01 灵感识别 ──is_inspiration=true──▶ 02 意图解析 ──▶ 03 结构化抽取
                                                          │
                                                          ▼
04 数据入库 ◀── 清洗校验 ───────── entity_json 落库
     │
     ▼
05 关联补全 ──history_records──▶（数据库检索）
     │
     ▼
06 灵感卡片生成 ──▶ 交付用户
```

| 节点 | 类型 | 关键输出 | 下游消费 |
| --- | --- | --- | --- |
| 01 灵感识别 | LLM | `is_inspiration` / `inspiration_type` | 02、短路分支 |
| 02 意图解析 | LLM | `creative_type` / `keywords` | 03、06 |
| 03 结构化抽取 | LLM | `inspiration_entity` JSON | 04、05、06 |
| 04 数据入库 | DB/Code + 校验LLM | 数据库记录 | 05（检索源）、06 |
| 05 关联补全 | LLM | `related_ids` / `enriched_context` | 06 |
| 06 灵感卡片生成 | LLM | 卡片 JSON | 用户/UI 渲染 |

---

> 本内容由 Coze AI 生成，请遵循相关法律法规及《人工智能生成合成内容标识办法》使用与传播。
