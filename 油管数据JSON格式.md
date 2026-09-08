# 数据目录说明

本目录保存“2026 年 FIFA 世界杯佛得角队相关舆情”项目的 YouTube 采集数据。数据来自官方 YouTube Data API v3，采集范围为 UTC 时间 `[2026-05-28T00:00:00Z, 2026-07-27T00:00:00Z)`。

## 目录结构

```text
data/
├─ raw/                         API 原始响应（评论作者字段已脱敏）
│  └─ youtube/<event_id>/
│     ├─ search/                分查询、排序方式保存的搜索响应
│     ├─ videos/                按批次保存的视频详情响应
│     └─ comments/              按视频和页码保存的一级评论响应
├─ interim/                     采集断点和逐条标准化中间数据
│  └─ youtube/
│     ├─ quota_ledger.json      实际 API 用量账本
│     └─ <event_id>/
│        ├─ state.json          断点、失败项和处理进度
│        ├─ search_index.json   视频与召回查询的对应关系
│        └─ documents/          每个视频或评论一个 JSON 文件
├─ processed/                   可直接交给分析程序的最终数据
│  └─ youtube_<event_id>.json
└─ README.md                    本说明文件
```

`raw_path` 均相对于本 `data` 目录。例如：

```text
raw/youtube/<event_id>/comments/<video_id>/page_0001.json
```

## 最终数据结构

最终统一 JSON 位于：

```text
processed/youtube_evt_2026_fifa_world_cup_cape_verde.json
```

顶层结构：

```json
{
  "schema_version": "1.0.0",
  "event_id": "evt_2026_fifa_world_cup_cape_verde",
  "event_name": "2026年FIFA世界杯佛得角队相关舆情",
  "collected_at": "...",
  "collection_window": {
    "start": "2026-05-28T00:00:00Z",
    "end_exclusive": "2026-07-27T00:00:00Z"
  },
  "queries": [],
  "documents": [],
  "failed_items": [],
  "stats": {}
}
```

### `queries`

记录使用过的检索式，包括：

- `query_id`：稳定的查询编号；
- `q`：发送给 YouTube 的检索文本；
- `relevance_language`：检索排序使用的语言提示；
- `orders`：视频搜索排序方式；
- `unique_candidates`：该查询召回的去重候选视频数。

### `documents`

视频和一级评论都作为独立 Document 保存。通用字段包括：

- `doc_id`：数据集内稳定、唯一的文档 ID；
- `doc_type`：`video` 或 `comment`；
- `source_id`、`source_name`：来源平台；
- `platform_object_id`：YouTube 原始对象 ID；
- `canonical_url`：对应的公开视频或评论链接；
- `published_at`：UTC 发布时间；
- `content_text`：视频简介或评论正文；
- `language`：本地识别或平台提供的语言；
- `raw_path`：对应原始 API 响应路径；
- `content_hash`：规范化文本的 SHA-256，用于一致性检查和去重；
- `query_ids`：召回所属视频的查询编号；
- `matched_keywords`：本地相关性规则命中的词组；
- `metrics`：播放、点赞、评论或回复数量的采集时快照；
- `duplicate_of`：跨对象去重关系，未关联时为 `null`；
- `extra`：平台和采集策略的补充信息。

视频文档的 `extra` 还包含从原始 `videos.list` 响应提升的公开元数据：

- `category_id`、`tags`、`thumbnails`；
- `default_language`、`default_audio_language`、`localized`；
- `live_broadcast_content`；
- `duration`、`dimension`、`definition`、`caption`、`projection`；
- `licensed_content`、`content_rating`；
- `upload_status`、`privacy_status`、`license`、`embeddable`；
- `public_stats_viewable`、`made_for_kids`、`contains_synthetic_media`。

这些字段来自已经保存的原始视频响应；将它们补充到最终数据不需要新增 API 请求。字段为 `null`、空数组或空对象，表示 YouTube 没有为该公开视频返回相应信息。

视频文档的关系字段：

```text
parent_doc_id = null
root_doc_id   = 视频自身 doc_id
thread_id     = youtube:<video_id>
```

评论文档的关系字段：

```text
parent_doc_id = 所属视频 doc_id
root_doc_id   = 所属视频 doc_id
thread_id     = youtube:<video_id>
```

本次只采集一级评论，不存在 `reply` 文档。评论使用 YouTube `relevance` 排序，每个视频最多请求 5 页、最多保留 300 条时间窗内评论。因此评论是平台热度/相关性样本，并非完整评论全集；这一性质记录在：

```json
{
  "collection_order": "relevance",
  "sampling_method": "youtube_relevance_ranking",
  "is_complete_comment_set": false
}
```

### `failed_items`

记录未能读取的对象和原因，包括处理阶段、错误代码、重试次数以及是否可重试。评论关闭、视频删除或转为私密不会中断其他对象的采集。

### `stats`

记录文档数量、过滤数量、重复文本组、评论停止原因、系统代理检测结果，以及已经实际发生的 API 调用量。项目已移除运行前用量预估，只保留实际用量账本和保护线控制。

## 证据回溯

知识图谱或分析结果可以按以下链路回到证据：

```text
Event(event_id)
  └─ Document(doc_id)
       ├─ Source(source_id)
       ├─ Query(query_ids)
       ├─ 在线页面(canonical_url)
       └─ 原始响应(raw_path → platform_object_id)
```

`content_hash` 校验的是规范化后的文本内容，不是整个原始响应文件的哈希。在线 URL 可能因视频删除、转私密或评论移除而失效，此时以本地脱敏后的 `raw_path` 作为采集时证据快照。

## 隐私与密钥

- 评论作者显示名不写入最终数据；
- 评论作者频道 ID 经 HMAC-SHA256 后保存为 `author_id_hash`；
- 评论原始响应写盘前会删除作者显示名、频道 ID、频道 URL 和头像 URL；
- API key、作者哈希密钥和系统代理地址不会写入本目录；
- 项目密钥位于被 Git 忽略的 `.secrets/youtube.env`，不要复制到 `data` 目录。

## 当前批次概况

当前完整采集结果包括：

- 视频文档：1,718 条；
- 一级评论文档：62,065 条；
- 文档总数：63,783 条；
- 被本地相关性规则过滤的视频：607 条；
- 失败项：10 个，其中视频不可用 2 个、评论关闭 8 个；
- 实际 API 调用：`search.list` 80 次、`videos.list` 48 次、`commentThreads.list` 2,129 次。

若之后重新运行采集器，本节数字可能不再与最新结果一致，应以最终 JSON 的 `stats` 和 `interim/youtube/quota_ledger.json` 为准。

## 使用注意事项

1. 分析程序优先读取 `processed` 中的最终统一 JSON；
2. 需要核验来源时，根据 `raw_path` 读取 `raw` 中的响应，再用 `platform_object_id` 定位对象；
3. 不要手动修改 `interim/state.json`，否则可能破坏断点续传；
4. 不要把 `raw` 或 `interim/documents` 当作额外样本重复计数；
5. 移动数据时应整体保留 `raw`、`interim` 和 `processed` 的相对目录结构。
