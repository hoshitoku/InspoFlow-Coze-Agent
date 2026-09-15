---
AIGC:
    Label: "1"
    ContentProducer: 001191110102MACQD9K64018705
    ProduceID: 784407080744535_7566957503410667562-drive/225515273856904049/database-schema.md
    ReservedCode1: ""
    ContentPropagator: 001191110102MACQD9K64028705
    PropagateID: 784407080744535#1789459249370
    ReservedCode2: ""
---
# InspoFlow 数据库 Schema 文档

> **文档用途**：定义 InspoFlow「创作者灵感捕手」的数据库表结构、索引设计与读写规范，供工作流节点（04 数据入库、05 关联补全、06 卡片生成）对接使用。
>
> **对应设计**：README.md 第 3 章「数据流设计」、workflow-prompts.md 第 04/05 节点。
>
> **数据库**：PostgreSQL（Coze 空间级 Supabase 数据库）
>
> **版本**：v1.0（schema_version=1）

---

## 1. 数据库概览

### 1.1 设计原则

1. **原文优先**：`raw_input` 永远保留用户原始输入，结构化数据只是索引而非替代——保证信息不丢失。
2. **结构化可检索**：核心要素以 JSONB 落库，同时提取高频检索字段（类型/情绪/标签）为独立列，兼顾灵活性与查询性能。
3. **幂等写入**：以幂等键保证同一灵感重复触发不产生脏数据。
4. **可演进**：`schema_version` 字段支持结构版本管理，便于后续迁移。

### 1.2 Schema 清单

| Schema | 用途 |
| --- | --- |
| `public` | 业务数据（灵感主表） |
| `knowledge` | 预留（后续素材库/知识沉淀） |

---

## 2. 核心表：`inspirations`（灵感主表）

### 2.1 表定义

```sql
CREATE TABLE IF NOT EXISTS public.inspirations (
    -- ===== 主键与标识 =====
    id                TEXT PRIMARY KEY,           -- 灵感唯一 ID（格式：insp_YYYYMMDD_xxxx）
    user_id           TEXT NOT NULL,              -- 创作者 ID（Coze 用户标识）
    idempotency_key   TEXT UNIQUE NOT NULL,       -- 幂等键：user_id + raw_input 哈希，防重复写入

    -- ===== 原始内容 =====
    raw_input         TEXT NOT NULL,              -- 用户原始输入（保留原文，禁止截断）
    media_url         TEXT,                       -- 多模态输入时的媒体 URL（图片/语音，可选）
    media_type        TEXT,                       -- 媒体类型：text / image / audio（默认 text）

    -- ===== 结构化数据 =====
    entity_json       JSONB NOT NULL DEFAULT '{}'::jsonb,  -- 03 节点结构化抽取结果（characters/scenes/emotions/world_building/plot_hooks/key_quotes/conflicts）
    inspiration_type  TEXT NOT NULL,              -- 创作类型：同人 / 原创 / 随笔 / 脑洞
    creative_type     TEXT,                       -- 02 节点意图解析的作品形式（小说/同人文/短篇/诗歌等）
    emotion           TEXT,                       -- 情绪标签（保留情绪语境，如"孤独中带着期待"）
    tags              TEXT[] NOT NULL DEFAULT '{}',         -- 标签数组（用于检索关联）

    -- ===== 关联 =====
    related_ids       TEXT[] NOT NULL DEFAULT '{}',         -- 关联灵感 ID 列表（05 节点回写）

    -- ===== 状态与版本 =====
    status            TEXT NOT NULL DEFAULT 'active',       -- active 正常 / archived 归档 / used 已转化 / failed 入库失败
    schema_version    INT NOT NULL DEFAULT 1,               -- 结构版本号，支持迁移

    -- ===== 时间戳 =====
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),   -- 创建时间
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),   -- 更新时间（关联补全/状态变更时刷新）
    deleted_at        TIMESTAMPTZ,                          -- 软删除时间（NULL=未删除）
    CONSTRAINT inspirations_status_check CHECK (status IN ('active', 'archived', 'used', 'failed'))
);

-- 更新时间自动刷新
CREATE OR REPLACE FUNCTION public.set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_inspirations_updated_at
    BEFORE UPDATE ON public.inspirations
    FOR EACH ROW
    EXECUTE FUNCTION public.set_updated_at();
```

### 2.2 字段详解

| 字段 | 类型 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- | --- |
| `id` | TEXT | ✅ | - | 主键，格式 `insp_YYYYMMDD_xxxx`（日期 + 随机后缀），由入库节点生成 |
| `user_id` | TEXT | ✅ | - | 创作者 ID，贯穿全流程的全局变量，用于隔离不同用户数据 |
| `idempotency_key` | TEXT | ✅ | - | 幂等键 = `user_id + raw_input` 哈希；唯一约束，重复触发走更新而非新增 |
| `raw_input` | TEXT | ✅ | - | 用户原始输入，**永不截断**，是信息不丢失的兜底 |
| `media_url` | TEXT | - | NULL | 多模态输入的媒体文件 URL（图片/语音），纯文本时为空 |
| `media_type` | TEXT | - | `text` | 输入类型：text / image / audio |
| `entity_json` | JSONB | ✅ | `{}` | 03 节点输出的完整结构化结果，见 2.3 |
| `inspiration_type` | TEXT | ✅ | - | 灵感大类：同人 / 原创 / 随笔 / 脑洞（01 节点判定） |
| `creative_type` | TEXT | - | NULL | 想发展的作品形式：小说 / 同人文 / 短篇 / 诗歌 / 漫画脚本等（02 节点） |
| `emotion` | TEXT | - | NULL | 情绪标签，保留"当时的情绪语境"（02/03 节点） |
| `tags` | TEXT[] | ✅ | `{}` | 标签数组，供检索与关联（02 节点 keywords 扩展） |
| `related_ids` | TEXT[] | ✅ | `{}` | 关联灵感 ID 列表（05 节点回写），形成素材网络 |
| `status` | TEXT | ✅ | `active` | active / archived / used / failed |
| `schema_version` | INT | ✅ | 1 | 结构版本号，低版本数据按默认值补齐兼容 |
| `created_at` | TIMESTAMPTZ | ✅ | now() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | ✅ | now() | 更新时间，触发器自动刷新 |
| `deleted_at` | TIMESTAMPTZ | - | NULL | 软删除时间；非 NULL 视为已删除，查询默认过滤 |

### 2.3 `entity_json` 结构（03 节点输出）

```json
{
  "characters": [
    {"name": "沈砚", "description": "沉默寡言的美术生"}
  ],
  "scenes": {
    "location": "高中天台",
    "time": "冬夜",
    "atmosphere": "冷清但温暖"
  },
  "emotions": {
    "captured": "孤独中带着期待",
    "to_convey": "两个人彼此靠近的试探"
  },
  "world_building": "现代校园",
  "plot_hooks": ["天台相遇", "未说完的话"],
  "key_quotes": ["冬天的天台，连风都是安静的。"],
  "conflicts": ["家庭反对学艺术"]
}
```

| 子字段 | 类型 | 说明 |
| --- | --- | --- |
| `characters` | array | 人物列表，`{name, description}` |
| `scenes` | object | 场景设定 `{location, time, atmosphere}` |
| `emotions` | object | 情绪要素 `{captured, to_convey}` |
| `world_building` | string | 世界观设定 |
| `plot_hooks` | array | 可发展的情节钩子 |
| `key_quotes` | array | 原文金句（**必须保留用户原文，不得改写**，作情绪锚点） |
| `conflicts` | array | 潜在矛盾冲突点 |

---

## 3. 索引设计

### 3.1 索引清单

```sql
-- 按用户 + 时间查询（列表页、历史检索）
CREATE INDEX IF NOT EXISTS idx_inspirations_user_created
    ON public.inspirations (user_id, created_at DESC)
    WHERE deleted_at IS NULL;

-- 按灵感类型筛选
CREATE INDEX IF NOT EXISTS idx_inspirations_type
    ON public.inspirations (user_id, inspiration_type)
    WHERE deleted_at IS NULL;

-- 按状态筛选（定时清洗扫描用）
CREATE INDEX IF NOT EXISTS idx_inspirations_status
    ON public.inspirations (status)
    WHERE deleted_at IS NULL;

-- 标签检索（05 关联补全：按标签找历史灵感）
CREATE INDEX IF NOT EXISTS idx_inspirations_tags
    ON public.inspirations USING GIN (tags);

-- JSON 内的人物名检索（可选，灵感量大时启用）
CREATE INDEX IF NOT EXISTS idx_inspirations_entity_characters
    ON public.inspirations USING GIN ((entity_json -> 'characters'));
```

### 3.2 索引选择说明

| 索引 | 场景 | 理由 |
| --- | --- | --- |
| `user_created` | 列表页/时间线 | 高频查询，联合索引走最左前缀 |
| `type` | 按类型筛选 | 常规业务筛选 |
| `status` | 清洗任务扫描 | 部分索引（WHERE deleted_at IS NULL）控制体积 |
| `tags` | 关联补全检索 | GIN 索引支持数组包含查询 `tags @> '{标签}'` |
| `entity_characters` | 人物关联检索 | 仅灵感量级大时启用，避免写入开销 |

---

## 4. 读写规范（节点对接）

### 4.1 写入（04 数据入库节点）

```sql
-- 幂等写入：存在则更新，不存在则新增
INSERT INTO public.inspirations
    (id, user_id, idempotency_key, raw_input, media_url, media_type,
     entity_json, inspiration_type, creative_type, emotion, tags)
VALUES
    ({{id}}, {{user_id}}, {{idempotency_key}}, {{raw_input}}, {{media_url}}, {{media_type}},
     {{entity_json}}, {{inspiration_type}}, {{creative_type}}, {{emotion}}, {{tags}})
ON CONFLICT (idempotency_key)
DO UPDATE SET
    entity_json = EXCLUDED.entity_json,
    inspiration_type = EXCLUDED.inspiration_type,
    tags = EXCLUDED.tags,
    updated_at = now();
```

**写入前校验（04 节点清洗逻辑）**：
- `raw_input` 非空（空壳记录直接拒绝）
- `entity_json` 可被 JSON.parse（格式非法标记 failed）
- 必填字段齐全：`inspiration_type` / `emotion` 缺失时允许为空，但 `raw_input` 必须存在
- 校验失败 → `status='failed'` 入库 + 错误留痕，**不静默丢弃**

### 4.2 查询（05 关联补全节点）

```sql
-- 检索同用户的历史灵感（排除自身、排除已删除）
SELECT id, raw_input, entity_json, tags, inspiration_type, emotion
FROM public.inspirations
WHERE user_id = {{user_id}}
  AND deleted_at IS NULL
  AND id <> {{current_id}}
  AND status IN ('active', 'used')
  AND created_at >= now() - interval '180 days'   -- 窗口期，避免全量扫描
ORDER BY created_at DESC
LIMIT 50;

-- 按标签关联检索（候选集收窄）
SELECT id, raw_input, entity_json
FROM public.inspirations
WHERE user_id = {{user_id}}
  AND deleted_at IS NULL
  AND tags && {{current_tags}}                     -- 数组有交集
LIMIT 20;
```

### 4.3 关联回写（05 节点后置）

```sql
-- 回写关联关系（双向）
UPDATE public.inspirations
SET related_ids = array_append(related_ids, {{related_id}}),
    updated_at = now()
WHERE id = {{current_id}} AND NOT ({{related_id}} = ANY(related_ids));
```

### 4.4 卡片生成读取（06 节点）

```sql
SELECT id, raw_input, entity_json, inspiration_type, emotion, tags, related_ids
FROM public.inspirations
WHERE id = {{current_id}};
```

---

## 5. 状态机

```
                 ┌─────────────┐
  用户触发 ────▶ │  processing  │（工作流执行中，可选中间态）
                 └─────────────┘
                       │ 成功
                       ▼
                 ┌─────────────┐
                 │   active     │ ◀─── 默认状态，灵感可被检索/关联
                 └─────────────┘
                  │          │
        用户归档   │          │ 灵感被用于创作
                  ▼          ▼
          ┌───────────┐  ┌─────────┐
          │ archived   │  │  used    │（已转化，仍可关联）
          └───────────┘  └─────────┘

  写入校验失败 → failed（不进入正常流转，留痕供优化）
  软删除     → deleted_at 非 NULL（查询默认过滤，可恢复）
```

| 状态 | 含义 | 可检索 | 说明 |
| --- | --- | --- | --- |
| `active` | 正常 | ✅ | 默认状态，参与关联补全 |
| `archived` | 已归档 | ❌ | 用户手动归档，不再参与检索 |
| `used` | 已转化 | ✅ | 已被用于创作，仍保留在素材网络中 |
| `failed` | 入库失败 | ❌ | 校验未通过，留痕供 Prompt 优化 |

---

## 6. 脏数据治理

| 问题 | 检测 SQL | 处置 |
| --- | --- | --- |
| 重复记录 | `idempotency_key` 唯一约束 | 入口拦截，重复触发走 UPDATE |
| 空壳记录 | `raw_input = ''` 或 NULL | 定时扫描删除/标记 failed |
| 孤立关联 | `related_ids` 指向已删/不存在 ID | 定时任务修复，清理无效元素 |
| 格式漂移 | `schema_version < 当前版本` | 读取侧按默认值补齐 + 一次性迁移脚本升级 |

**示例：清理孤立关联**

```sql
-- 找出包含失效关联的记录（配合应用层校验）
SELECT id, related_ids
FROM public.inspirations
WHERE deleted_at IS NULL
  AND EXISTS (
    SELECT 1 FROM unnest(related_ids) AS rid
    WHERE rid NOT IN (SELECT id FROM public.inspirations WHERE deleted_at IS NULL)
  );
```

---

## 7. 版本与迁移

### 7.1 迁移规范

1. 每次结构变更递增 `schema_version`（当前 v1）。
2. 新增字段一律允许 NULL 或带默认值，保证低版本数据可读。
3. 提供一次性回填脚本，将旧数据升级到新结构（不依赖应用层逐个修补）。
4. 破坏性变更（删字段/改类型）先做数据备份 + 双写窗口。

### 7.2 变更记录

| 版本 | 日期 | 变更内容 |
| --- | --- | --- |
| v1.0 | - | 初始结构：灵感主表 + 索引 + 状态机 + 幂等设计 |

---

## 8. 预留扩展

| 表/能力 | 说明 | 触发场景 |
| --- | --- | --- |
| `users` 创作者维度表 | 沉淀创作者画像（创作偏好/常用类型/灵感活跃度） | 后续个性化推荐 |
| `knowledge` 素材库 | 结构化素材沉淀（人物档案/世界观设定库） | 灵感转化为长篇创作 |
| `inspiration_versions` 版本表 | 记录灵感的多次编辑历史 | 支持灵感回溯/草稿迭代 |
| 全文检索 | `raw_input` / `entity_json` 的 tsvector 索引 | 灵感量大后支持全文搜索 |

---

