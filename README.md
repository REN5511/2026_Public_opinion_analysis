# B站帖子与一级评论采集器

程序分成两个阶段：`bilibili_crawler.py` 按关键词采集作为主体的公开视频（帖子），`bilibili_comments.py` 读取已有视频事件 JSON，补充作为辅助材料的一级评论。评论阶段基于 Scrapy 做有限并发、自动限速、HTTP 重试和断点落盘。

## 快速开始

需要 Python 3.10 或更高版本。先安装依赖：

```powershell
python -m pip install -r requirements.txt
```

采集视频：

```powershell
python bilibili_crawler.py --keyword "人工智能" --event-name "B站人工智能讨论"
```

结果保存在 `output/evt_*.json`，原始接口响应保存在 `output/raw/bilibili/evt_*/`。每条文档的 `raw_path` 可以回溯到对应原始响应。

也可以直接读取课程组编写的 Markdown/YAML 关键词配置：

```powershell
python bilibili_crawler.py `
  --config "世界杯关键词.md" `
  --event-name "2026世界杯佛得角队舆情" `
  --order pubdate `
  --pages 50 `
  --max-results 5000 `
  --output-dir "output/cape_verde"
```

程序支持多个必选 OR 组、配置内日期范围、多个排序、断点续跑和日期分片。结果量较大时可用 `--slice-days 7` 将时间窗口按周切分，减少搜索结果上限造成的遗漏；再次运行默认复用已有原始响应并合并已有汇总数据。

更完整的示例：

```powershell
python bilibili_crawler.py `
  --keyword "人工智能" `
  --keyword "大模型" `
  --event-name "B站人工智能与大模型讨论" `
  --all-keyword "人工智能" `
  --any-keyword "大模型,AI" `
  --exclude-keyword "炒股,广告" `
  --start-date 2026-08-01 `
  --end-date 2026-09-07 `
  --order pubdate `
  --pages 2 `
  --max-results 30
```

关键词过滤规则与课程文档一致：

- `--all-keyword`：所有词都必须出现（AND）。
- `--any-keyword`：至少出现一个词（OR）。
- `--exclude-keyword`：出现任意一个就排除（NOT）。

选项可以重复，也可以用中英文逗号分隔。过滤范围是标题、作者、简介和标签。

## 输出字段

顶层字段为 `event_id`、`event_name`、`collected_at`、`documents` 和 `failed_items`。文档字段为：

```json
{
  "doc_id": "doc_...",
  "source_id": "bilibili_com",
  "source_name": "哔哩哔哩",
  "canonical_url": "https://www.bilibili.com/video/BV...",
  "title": "...",
  "author": "...",
  "published_at": "2026-09-01",
  "content_text": "标题、简介与标签",
  "raw_path": "raw/bilibili/evt_.../..._page_1.json",
  "content_hash": "sha256:..."
}
```

搜索接口不提供视频字幕，因此 `content_text` 只保存公开搜索结果中真实存在的标题、简介和标签，不虚构正文。缺失作者或日期会写成 JSON `null`。

## 采集一级评论

对现有视频事件 JSON 补充评论，例如世界杯数据：

```powershell
python bilibili_comments.py `
  --input-json "output/cape_verde/evt_2138e5c3c89b854d.json" `
  --start-date 2026-05-28 `
  --end-date 2026-07-26 `
  --comment-pages 5 `
  --comments-per-video 100 `
  --concurrency 3 `
  --no-obey-robots
```

先验证小样本时增加：

```powershell
  --max-videos 2 --comment-pages 1
```

默认不会覆盖输入文件。最终统一数据写入：

```text
output/cape_verde/processed/bilibili_evt_2138e5c3c89b854d.json
```

评论采集约束：

- 只请求评论区主列表，不调用楼中楼回复接口；
- 只保留 `root=0` 且 `parent=0` 的一级评论；
- 接口响应中附带的二级回复预览在 raw 落盘前删除；
- 评论作者显示名不落盘，作者 UID 经过 HMAC-SHA256 保存；
- 用户头像、明文 UID 和 IP 属地不落盘；
- 每个视频的评论页依赖上一页游标，因此单视频内串行，多个视频之间有限并发。

Scrapy 默认遵守 `api.bilibili.com/robots.txt`，当前会阻止评论接口请求。如果课程已经完成数据采集授权和合规确认，需要像既有搜索程序一样直接访问接口，必须显式传入 `--no-obey-robots`；程序不会在后台自动关闭该保护。

首次运行会在 `.secrets/bilibili_author_hmac.key` 生成本地作者哈希密钥，该目录已被 Git 忽略。若匿名访问无法读取评论，可以通过 `--cookie-file` 提供 Cookie；Cookie 不会写入 raw、状态或最终 JSON。

评论断点和逐条文档位于：

```text
output/cape_verde/interim/bilibili/<event_id>/
```

原始评论页位于：

```text
output/cape_verde/raw/bilibili/<event_id>/comments/<BV号>/page_0001.json
```

最终 JSON 使用 `stats.video_documents`、`stats.comment_documents` 和 `stats.documents_total` 分别记录帖子、评论及总条数，不应使用文本行数估算条目数。

## 注意事项

- 程序不绕过登录、验证码或访问限制。若 `failed_items` 出现 HTTP 412、`-352` 或 `-412`，请停止频繁请求，稍后重试。
- 默认翻页间隔 1.5 秒；不建议把 `--interval` 调得过低。
- 接口和网页结构可能变化，课程项目中应保留 `raw/` 目录，便于重新解析与排错。
- 请遵守 B 站服务条款、robots 规则与相关法律，不采集个人敏感信息。

## 测试

测试使用本地模拟响应，不会请求 B 站：

```powershell
python -m unittest discover -s tests -v
```
very good

## 海外公开数据

Guardian、Google News、Bing News、Reddit、Lemmy、Stack Exchange 等海外来源的世界杯采集代码和数据已整理到 `外网` 分支。文件结构、数据规模、字段和复现方式见 [外网数据说明.md](外网数据说明.md)。
