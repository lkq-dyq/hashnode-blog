# hashnode-blog · 秦梓恒的技术写作源（GitHub 同步发布到 Hashnode）

本仓库是 [Hashnode](https://hashnode.com) 博客的 **Markdown 源**。`posts/` 下的每篇 `.md` 推送到 `main` 后，由
[`Hashnode/publish-github-action`](https://github.com/Hashnode/publish-github-action) 自动发布 / 更新到你的 Hashnode 发布页。

博客地址（发布后）：`https://<你的handle>.hashnode.dev`
对应 GitHub Pages 技术博客：`https://lkq-dyq.github.io/`

## 目录结构

```
hashnode-blog/
├── posts/                  # 每篇一个 .md，frontmatter 见下
│   ├── datamind-zero-code-data-analysis-agent.md
│   ├── creditguard-thin-file-credit-risk.md
│   ├── pdf-source-code-indent-reconstruction.md
│   └── pet-recognition-monitoring-system.md
├── .github/workflows/publish-to-hashnode.yml
└── README.md
```

## 文章 frontmatter 规范（Hashnode 官方字段）

```yaml
---
title: "标题"
slug: stable-slug            # 稳定 slug，是更新匹配键，务必保持不变
subtitle: 秦梓恒的技术笔记
tags: llm,agent,python        # 逗号分隔，最多 15 个；未知标签自动创建
publishedAt: 2026-09-23T09:00:00Z
saveAsDraft: false           # true = 发草稿
enableToc: true
hideFromCommunity: false
canonical: https://lkq-dyq.github.io/...   # 原文在 Pages 博客的地址，避免 SEO 重复
---
正文（Markdown）……
```

## 首次发布设置（一次性）

1. **注册 Hashnode**：https://hashnode.com → 创建发布页，得到地址如 `qinziheng.hashnode.dev`（handle 可不同于 GitHub 用户名）。
2. **生成 Personal Access Token**：Hashnode → Settings → Developer → 生成 PAT。
3. **配置仓库 Secret**：本仓库 Settings → Secrets and variables → Actions → 新建仓库密钥
   - Name：`HASHNODE_PAT`
   - Value：上面的 PAT
4. **改 publication-host**：编辑 `.github/workflows/publish-to-hashnode.yml`，把 `lkq-dyq.hashnode.dev` 改成你的发布地址。
5. **推送**：`git push origin main` → Action 自动把 4 篇发布到 Hashnode。

之后：改 `posts/` 里任意 `.md` 再 push，对应文章**原地更新**；删除 `.md` **不会**删 Hashnode 上的文章（需去后台手动删）。

## 把 DataMind / CreditGuard 做成系列

这两篇在内容上是同一工具链的深度文。想归成系列：
1. 在 Hashnode 后台创建一个 Series（slug 例如 `ai-data-tools`）。
2. 给这两篇 `.md` 的 frontmatter 各加一行 `seriesSlug: ai-data-tools`，push 即归入系列。

## 注意事项

- **canonical 已指向 GitHub Pages 博客**：这是跨平台转载的最佳实践，避免搜索引擎判重复内容。
- **草稿**：新文章想先存草稿，把 `saveAsDraft: true` 推上去；定稿后改回 `false` 再推一次。
- **封面图**：`cover:` 可填图片 URL 或仓库内相对路径（Hashnode 会自动上传到其 CDN）；当前 4 篇未配封面，发布后可补。
- **语言**：当前为中文，与 GitHub Pages 博客 / CSDN 一致；如需国际受众，可换用 `outputs/medium/` 下的英文版。
