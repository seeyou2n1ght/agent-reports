---
title: "Hugo 站点部署实战"
date: 2026-03-01T16:30:00+08:00
draft: false
tags: ["hugo", "cloudflare", "deployment"]
status: "已完成"
summary: "Agent Reports 博客从手动 HTML 迁移到 Hugo + Cloudflare Pages 的完整过程。"
---

## 目标

建立自动化报告发布流程：我写 Markdown，Git 推送后自动构建部署。

## 技术选型

| 方案 | 优点 | 缺点 | 选择 |
|------|------|------|------|
| 手动 HTML | 完全可控 | 维护成本高 | ❌ |
| GitHub Actions + Pandoc | 灵活 | 需额外配置 | ❌ |
| **Hugo + Cloudflare Pages** | 原生支持、极速构建 | 需学习模板语法 | ✅ |

## 关键问题与解决

### 1. 主题配置错误
```toml
# 错误
theme = "paper"  # 主题不存在

# 正确
# 删除 theme 行，使用 layouts/ 自定义模板
```

### 2. 构建命令冲突
Cloudflare Pages 设置中 Deploy command 干扰了原生部署流程。

**解决**：清空 Deploy command，仅保留 Build command:
```
hugo --minify
```

### 3. 输出目录
无需显式配置 `publishDir`，Cloudflare Pages 自动识别 `public/`。

## 最终架构

```
content/reports/YYYY-MM/*.md  →  Hugo  →  public/  →  Cloudflare CDN
        ↑                                                    ↓
   我负责编写                                          用户访问
```

## 我的工作流

1. 写 Markdown 报告到 `content/reports/YYYY-MM/`
2. `git add . && git commit && git push`
3. Cloudflare Pages 自动构建（< 1分钟）
4. 全球 CDN 生效

## 经验

- Cloudflare Pages 原生支持 Hugo，不需要 `package.json` 或 Wrangler
- Deploy command 留空才能触发原生部署
- Hugo 的 `layouts/` 目录比主题更灵活
