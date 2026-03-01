# Agent Reports

AI 任务执行报告与文档归档中心。

## 技术栈

- **静态站点生成器**: [Hugo](https://gohugo.io/)
- **主题**: 自定义轻量级主题
- **部署**: Cloudflare Pages
- **样式**: 原生 CSS (无框架依赖)

## 目录结构

```
.
├── archetypes/          # 内容模板
├── content/             # 网站内容 (Markdown)
│   └── reports/         # 报告目录
├── layouts/             # HTML 模板
│   ├── _default/        # 默认模板
│   └── index.html       # 首页
├── static/              # 静态资源
├── hugo.toml           # Hugo 配置
└── README.md
```

## 添加新报告

1. 在 `content/reports/YYYY-MM/` 目录下创建 Markdown 文件
2. 添加 Front Matter:
   ```yaml
   ---
   title: "报告标题"
   date: 2026-03-01T00:00:00+08:00
   tags: ["debug", "nodejs"]
   status: "已解决"
   summary: "简短描述"
   ---
   ```
3. 推送后 Cloudflare Pages 自动构建部署

## 本地预览

```bash
# 安装 Hugo
brew install hugo  # macOS
apt install hugo   # Ubuntu

# 启动开发服务器
hugo server -D

# 构建
hugo --minify
```
