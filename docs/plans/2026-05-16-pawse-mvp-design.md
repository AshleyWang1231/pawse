# Pawse MVP 设计文档

## 项目概述

- **产品名称**: Pawse
- **定位**: 跨平台桌面休息提醒应用
- **核心价值**: "你的宠物会关心你工作太久"
- **开发平台**: Ubuntu（优先）→ macOS / Windows（打包目标）

## 技术栈（最终确认）

| 层 | 技术 | 原因 |
|---|---|---|
| 桌面框架 | Tauri v2 | 包小、内存低、三平台支持 |
| 前端 | React + TypeScript + Vite | AI coding 成熟度高 |
| UI | TailwindCSS | AI 生成质量高 |
| 动画 | Framer Motion | 桌宠动画所需 |
| 图片生成 | OpenAI DALL·E 3 + rembg 后处理透明化 | 透明背景，成本可控 |
| 本地存储 | SQLite (tauri-plugin-sql) | 本地离线，无需后端 |

## AI 选型决定

- **生成**: OpenAI DALL·E 3 API
- **透明背景**: 后端用 `rembg`（本地 Python 库）做背景移除
- **缓存**: 生成结果存本地 `generated_pets/` 目录

## 架构概览

```
┌─────────────────────────────┐
│      Tauri Shell (Rust)     │
│  ┌───────────────────────┐  │
│  │  React Frontend (TS)  │  │
│  │  - Welcome            │  │
│  │  - Upload Pet         │  │
│  │  - Generate Pet       │  │
│  │  - Dashboard          │  │
│  │  - Settings           │  │
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │  Overlay Window       │  │
│  │  - Always on top      │  │
│  │  - Framer Motion anim │  │
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │  Tauri Plugins        │  │
│  │  - SQLite             │  │
│  │  - Shell (rembg)      │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
         ↕    ↕    ↕
   ┌─────┐ ┌────┐ ┌──────────┐
   │SQLite│ │OS  │ │OpenAI API│
   │     │ │Evts│ │+ rembg   │
   └─────┘ └────┘ └──────────┘
```

## 状态机

```
IDLE
  │  (app just opened / no session)
  ▼
WORKING
  │  (keyboard/mouse detected)
  ├── idle > 5min → IDLE
  └── work_duration reached → BREAK_TRIGGERED
                                 │
                                 ▼
                            OVERLAY_ACTIVE
                                 │  (pet appears)
                                 ├── dismiss → WORKING
                                 └── break_duration finished → BREAK_FINISHED
                                                                     │
                                                                     ▼
                                                                WORKING
```

## 数据库结构（SQLite）

```sql
CREATE TABLE pets (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  image_path TEXT NOT NULL,
  generated_asset_path TEXT,
  style TEXT DEFAULT 'cartoon',
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE settings (
  id INTEGER PRIMARY KEY CHECK (id = 1),
  work_duration INTEGER DEFAULT 50,   -- minutes
  break_duration INTEGER DEFAULT 10,  -- minutes
  overlay_mode TEXT DEFAULT 'gentle',  -- gentle | half_block | full_block
  animation_enabled INTEGER DEFAULT 1,
  auto_launch INTEGER DEFAULT 0
);
```

## 模块拆分（开发顺序）

### Module A — App Shell (Tauri 基础框架)
- A1. 初始化 Tauri + React 项目
- A2. 配置 TailwindCSS + Framer Motion
- A3. 基础窗口管理（主窗口 + Overlay 窗口）
- A4. 路由系统

### Module B — 工作计时器
- B1. 工作状态检测（键盘/鼠标/活跃度）
- B2. 计时器逻辑
- B3. 空闲检测（5 分钟无操作→暂停）

### Module C — Overlay 系统
- C1. Tauri Overlay 窗口
- C2. 宠物定位（四角/底部/顶部）
- C3. 阻止工作模式（温柔/半屏/全屏）

### Module D — 宠物上传
- D1. 上传 UI（拖拽 + 点击）
- D2. 图片校验（格式/大小）
- D3. 图片预览

### Module E — AI 宠物生成
- E1. OpenAI DALL·E 3 调用
- E2. rembg 后处理透明化
- E3. 本地缓存
- E4. 风格模板

### Module F — 动画系统
- F1. Idle 呼吸动画
- F2. Blink 眨眼
- F3. Sleep 趴下
- F4. Paw Block 挡屏幕

### Module G — 提醒系统
- G1. 提醒触发
- G2. 自定义文案
- G3. 休息倒计时

### Module H — 设置系统
- H1. 时间设置
- H2. 宠物设置
- H3. 启动设置

## 全局验收标准

- [x] App 在 macOS + Windows + Ubuntu 上可运行
- [ ] 上传宠物照片 → 生成透明背景桌宠
- [ ] 工作达到阈值 → 宠 Overlay 提醒
- [ ] 动画：呼吸 + 眨眼 + 挡屏幕
- [ ] 用户可修改工作时长、提醒文案、宠物风格
- [ ] 内存 < 250MB，CPU 空闲 < 5%，安装包 < 30MB

## 非目标（不做）

- 用户社区 / 云同步 / 多宠物互动 / AI 对话 / 视频生成 / 复杂骨骼动画 / 手机端 / 在线账号
