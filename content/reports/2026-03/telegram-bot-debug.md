---
title: "Pi Telegram Bot 问题排查与架构重构"
date: 2026-03-01T00:00:00+08:00
draft: false
tags: ["debug", "nodejs", "telegram", "architecture", "pm2"]
status: "已解决"
summary: "Telegram Bot 退出 TUI 后无响应问题的完整排查过程，包含架构分析、spawn 配置问题和解决方案。"
---

## 项目概述

### 目标
为 pi-coding-agent 实现 Telegram 集成功能，允许用户通过 Telegram Bot 与 AI 助手交互。

### 技术栈
- **Runtime**: Node.js 24.14.0
- **Process Manager**: PM2
- **AI Backend**: pi-coding-agent CLI
- **API**: Telegram Bot API (HTTPS Long Polling)

---

## 问题描述

### 初始症状
用户反馈：**退出 pi TUI 后，Telegram 发送消息没有回应**

### 环境信息
| 项目 | 值 |
|------|-----|
| Bot Token | `[REDACTED]` |
| Admin ID | `[REDACTED]` |
| Server Location | Hong Kong |
| pi Version | 0.55.3 |

---

## 排查过程

### 第一阶段：检查服务状态

```bash
# 检查 PM2 进程状态
pm2 status
```

**发现**：
- `pi-telegram-bridge` 进程显示 `online`
- 但日志为空，无消息处理记录

### 第二阶段：分析架构问题

#### 原架构（扩展模式）
```
User (Telegram) 
    ↓ HTTP
Telegram Bot API
    ↓ Long Polling
pi-telegram-bridge (Extension)
    ↓ Event Bus
pi TUI Session
    ↓ Internal API
AI Response
```

**问题识别**：
- 扩展依赖 `session_start` 事件启动轮询
- 扩展依赖 `message_end` 事件接收 AI 响应
- **当 TUI 退出时，`session_shutdown` 触发，所有事件监听器被清理**
- 结果：Bot 仍在运行，但无法处理消息

#### 冲突检测
```bash
pm2 status
```

发现同时存在两个服务：
1. `pi-telegram-bot` - 独立的旧版 Bot（已损坏）
2. `pi-telegram-bridge` - pi 扩展（依赖 TUI）

**两者使用相同的 Bot Token，导致更新竞争**

### 第三阶段：独立服务开发

#### 设计决策

| 方案 | 优点 | 缺点 | 选择 |
|------|------|------|------|
| Webhook | 实时推送 | 需要 HTTPS + 域名 | ❌ |
| Long Polling | 简单可靠 | 1-3秒延迟 | ✅ |
| RPC Bridge | 复用 pi 实例 | 复杂度高 | ❌ |
| Direct Spawn | 完全独立 | 每次启动开销 | ✅ |

#### 核心问题：spawn 配置

**首次尝试失败**：
```javascript
const proc = spawn('pi', ['--print', '...'], {
  env: { ...process.env, NODE_OPTIONS: '' },  // ❌ 错误
  cwd: process.cwd()
});
// 结果：进程启动但无输出，120秒后超时
```

**调试测试**：
```javascript
const proc = spawn('pi', ['--print', 'hello'], {
  env: process.env,                           // ✅ 正确
  stdio: ['ignore', 'pipe', 'pipe']           // ✅ 关键配置
});
// 结果：正常输出，3秒完成
```

**根本原因**：
- `NODE_OPTIONS=''` 覆盖了必要的环境变量
- 缺少 `stdio` 配置导致 stdout/stderr 未正确管道化

### 第四阶段：修复验证

#### 修复后的 spawn 配置
```javascript
const proc = spawn(CONFIG.PI_PATH, [
  '--print',
  '--no-session',
  '--no-extensions',
  '--tools', 'read,bash,edit,write',
  text
], {
  env: process.env,                    // ✅ 保留完整环境
  cwd: process.cwd(),
  stdio: ['ignore', 'pipe', 'pipe']    // ✅ 明确管道配置
});
```

#### 性能指标

| 指标 | 修复前 | 修复后 |
|------|--------|--------|
| 消息处理时间 | 120s+ (Timeout) | ~3s |
| 内存占用 | N/A (崩溃) | ~70MB |
| 并发处理 | 单线程阻塞 | 队列化处理 |

---

## 最终架构

### 系统架构图
```
┌─────────────────┐
│   User (TG App) │
└────────┬────────┘
         │ HTTPS
         ▼
┌─────────────────────┐
│ Telegram Bot API    │
│ api.telegram.org    │
└────────┬────────────┘
         │ Long Polling (timeout=30s)
         ▼
┌─────────────────────┐     ┌─────────────────┐
│   pi-tg-bot         │────▶│  pi --print     │
│   (Node.js Service) │     │  (Child Process)│
│   - PM2 managed     │◀────│  - Stateless    │
│   - Independent     │     │  - Per-message  │
└─────────────────────┘     └─────────────────┘
```

### 组件职责

| 组件 | 职责 | 生命周期 |
|------|------|----------|
| `pi-tg-bot` | Telegram 通信、消息路由、状态管理 | PM2 守护进程 |
| `pi --print` | AI 推理、工具执行 | 每消息一次 |
| PM2 | 进程监控、自动重启、日志管理 | 系统级 |

---

## 经验总结

### 关键教训

1. **环境隔离的重要性**
   - 不要覆盖 `process.env`，除非明确知道影响
   - `stdio` 配置对子进程通信至关重要

2. **架构解耦的价值**
   - 扩展模式适合 TUI 增强功能
   - 独立服务适合长期运行的后台任务

3. **调试方法论**
   - 隔离测试：创建最小可复现脚本
   - 分层验证：API → 服务 → 子进程 → AI 响应
   - 日志驱动：每个阶段都要有明确的日志输出

### 性能优化建议

| 优化点 | 当前 | 建议 |
|--------|------|------|
| 模型加载 | 每次启动 | 考虑保持 pi RPC 长连接 |
| 消息队列 | 单线程 | 实现并发限制（如 max 3） |
| 响应缓存 | 无 | 对重复查询添加缓存 |
| 会话记忆 | 无 | 实现简单的上下文窗口 |
