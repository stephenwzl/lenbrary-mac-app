# PhotoVault

照片丢进来就行，不会丢，会帮你整理。

## 项目结构

```
photovault/
├── engine/           # Processing Engine (Node.js)
│   ├── src/
│   │   ├── db/       # SQLite schema & migrations
│   │   ├── import/   # 文件扫描 & 导入
│   │   ├── exif/     # EXIF 提取
│   │   ├── dedup/    # 去重 (hash + pHash)
│   │   ├── ai/       # AI 索引 (多模态 API)
│   │   ├── search/   # FTS5 全文搜索
│   │   ├── api/      # HTTP API (前端调用)
│   │   └── index.js  # 入口
│   └── package.json
├── PhotoVault/       # SwiftUI App (Xcode 项目)
└── docs/             # 文档 (链接到设计文档)
```

## 架构

SwiftUI 前端 ← HTTP (localhost) → Processing Engine (Node.js) → SQLite + 文件系统

## 开发

### Processing Engine

```bash
cd engine
npm install
npm run dev
```

### SwiftUI App

用 Xcode 打开 `PhotoVault/` 目录。

## 文档

- [PRODUCT.md](docs/PRODUCT.md) — 产品概览 & ROADMAP
- [DESIGN.md](docs/DESIGN.md) — 设计语言
- [PRD-MVP.md](docs/PRD-MVP.md) — MVP 需求 (v0.1)
- [PRD-V2.md](docs/PRD-V2.md) — 后续版本规划 (v0.2 ~ v1.0)
