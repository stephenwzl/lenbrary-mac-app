# 03 — PRD: MVP (v0.1)

## 目标

验证核心价值："照片丢进来就能找到"。

用户导入一批照片 → App 自动备份、去重、生成 AI 描述 → 用户能通过自然语言搜索找到照片。

## 功能清单

### F1 — 本地文件夹导入

**用户故事**：我有一个文件夹的照片，想导入到 PhotoVault。

- 拖拽文件夹到 App 窗口，或通过菜单选择文件夹
- App 递归扫描文件夹内的所有图片文件
- 支持格式：JPEG, PNG, HEIC, TIFF, RAW (ARW/CR2/NEF/DNG)
- 导入时自动复制到 PhotoVault 存储目录，原文件不动
- 导入完成后显示摘要（导入 X 张，跳过 Y 张已存在）

**不支持**：
- 设备自动检测（v0.2）
- 在线导入

### F2 — EXIF 提取

**用户故事**：照片导入后，我能看到拍摄信息。

- 提取字段：拍摄时间、相机型号、镜头、光圈、快门、ISO、焦距、GPS
- 存入 SQLite
- 在详情面板显示

### F3 — 存储管理

**用户故事**：照片存到哪里了，我不用操心。

- 默认存储路径：`~/PhotoVault/`
- 按日期自动分目录：`2026/04-15/`
- 文件命名保留原始文件名
- 可在设置中修改存储路径
- 不删除、不修改原图

### F4 — 去重

**用户故事**：我导入了同一批照片两次，App 能识别并跳过。

- 精确去重：文件 hash（MD5/SHA256）比对
- 相似去重：pHash 感知哈希，识别同场景不同设备/不同张
- 相似照片自动分组，标记"重复"
- 去重结果存入 SQLite

### F5 — AI 索引

**用户故事**：照片导入后，App 自动给每张照片生成描述。

- 调用多模态 API（GPT-4o Vision / Claude Vision）
- 生成内容：自然语言描述、场景标签、质量评分（0-100）
- 批量处理，不阻塞用户操作
- 描述存入 SQLite + FTS5 索引
- API Key 在设置中配置
- 索引进度在底部状态栏显示

### F6 — 照片网格

**用户故事**：打开 App 就能看到我所有的照片。

- `LazyVGrid` 缩略图展示
- 按日期分组，组间有日期分隔条（`2026年4月15日 · 23张`）
- 缩略图自适应列数（根据窗口宽度）
- 无限滚动加载
- 点击选中（主色高亮边框）
- 缩略图缓存：内存 + 磁盘双层

### F7 — 自然语言搜索

**用户故事**：我想找"夕阳"相关的照片。

- 顶部搜索栏，Cmd+F 聚焦
- 输入即搜，实时结果
- 搜索范围：AI 生成的描述 + EXIF 信息
- 搜索结果替换网格内容
- 显示结果数量
- ESC / × 清空搜索

### F8 — 标记好片/废片

**用户故事**：浏览时我想快速标记哪些照片好、哪些要删。

- 右键菜单或快捷键标记：
  - 好片 ⭐（孔雀绿角标）
  - 废片（陶土色角标）
- 标记存入 SQLite
- 详情面板中也有标记按钮

### F9 — 照片详情面板

**用户故事**：点击一张照片，能看到详细信息。

- Cmd+I 切换显示/隐藏
- 显示：大缩略图、拍摄时间、设备/镜头、EXIF 参数、AI 描述、质量评分
- 操作：标记好片、标记废片、在 Finder 中显示

### F10 — 后台处理状态

**用户故事**：我知道 App 在处理什么、还剩多少。

- 底部状态栏常驻
- 显示：EXIF 提取进度、去重完成数、AI 索引进度（已索引/总数 + 预计剩余时间）
- 无任务时显示照片总数 / 存储占用

## 技术规格

### 架构

```
┌──────────────────────────────────────┐
│  SwiftUI Frontend                    │
│  - NavigationSplitView (三栏)        │
│  - LazyVGrid (网格)                  │
│  - 搜索栏 + 详情面板                 │
└──────────────┬───────────────────────┘
               │ IPC (本地)
┌──────────────┴───────────────────────┐
│  Processing Engine (Node.js/Python)  │
│  - Importer (文件扫描+复制)          │
│  - EXIF Extractor (exiftool)         │
│  - Deduplicator (hash + pHash)       │
│  - AI Indexer (多模态 API 调用)      │
│  - Search (ripgrep + SQLite FTS5)    │
└──────────────┬───────────────────────┘
               │
┌──────────────┴───────────────────────┐
│  Storage                             │
│  - 文件系统 (~/PhotoVault/YYYY/MM-DD/)│
│  - SQLite (元数据/描述/评分/去重记录) │
│  - 缩略图缓存                        │
└──────────────────────────────────────┘
```

### 数据库 Schema

```sql
-- 照片表
CREATE TABLE photos (
  id INTEGER PRIMARY KEY,
  file_path TEXT NOT NULL UNIQUE,
  original_path TEXT,           -- 原始来源路径
  file_name TEXT NOT NULL,
  file_hash TEXT,               -- 精确去重用
  phash TEXT,                   -- 感知哈希，相似去重用
  file_size INTEGER,
  width INTEGER,
  height INTEGER,
  format TEXT,                  -- jpeg/png/heic/raw
  
  -- EXIF
  taken_at DATETIME,
  camera TEXT,
  lens TEXT,
  aperture REAL,
  shutter_speed TEXT,
  iso INTEGER,
  focal_length REAL,
  gps_lat REAL,
  gps_lng REAL,
  
  -- AI
  ai_description TEXT,
  ai_tags TEXT,                 -- JSON array
  quality_score INTEGER,        -- 0-100
  
  -- 状态
  is_good BOOLEAN DEFAULT 0,
  is_bad BOOLEAN DEFAULT 0,
  duplicate_group_id INTEGER,   -- 相似照片分组
  
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  indexed_at DATETIME
);

-- FTS5 全文搜索索引
CREATE VIRTUAL TABLE photos_fts USING fts5(
  ai_description,
  ai_tags,
  camera,
  content='photos',
  content_rowid='id'
);

-- 重复组表
CREATE TABLE duplicate_groups (
  id INTEGER PRIMARY KEY,
  best_photo_id INTEGER,        -- 推荐保留的照片
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 关键依赖

| 依赖 | 用途 |
|------|------|
| SwiftUI | 前端框架 |
| ImageCaptureCore | 设备检测（v0.2） |
| exiftool | EXIF 提取 |
| sharp / libvips | 缩略图生成、pHash 计算 |
| ripgrep | 内容搜索 |
| SQLite + FTS5 | 元数据存储 + 全文搜索 |
| GPT-4o Vision / Claude Vision | AI 索引 |

## 验收标准

MVP 完成后，以下流程能跑通：

1. 拖拽一个包含 100 张照片的文件夹到 App
2. 照片出现在网格中，按日期分组
3. 点击照片，能看到 EXIF 和 AI 描述
4. 搜索"夕阳"，能找到相关照片
5. 标记几张好片，刷新后标记还在
6. 导入重复照片，自动跳过
7. 底部状态栏显示处理进度

---

_最后更新：2026-04-21_
