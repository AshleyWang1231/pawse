# Pawse MVP 实现计划

> **For implementer:** TDD 贯穿始终。先写测试，确认失败，再写实现，确认通过。

**目标:** 完成 Pawse 跨平台桌面休息提醒应用 MVP

**架构:** Tauri v2 + React/TypeScript/Vite 前端 + TailwindCSS + Framer Motion 动画 + OpenAI DALL·E 3 + rembg 透明背景 + SQLite 本地存储

**开发平台:** Ubuntu（优先）；打包目标 Ubuntu + macOS + Windows

**工作目录:** `apps/pawse/`

---

## Task 1: 初始化 Tauri + React 项目

**Files:**
- Create: `apps/pawse/` (Tauri v2 项目根目录)
- Create: `apps/pawse/src/` (React 前端)
- Create: `apps/pawse/src-tauri/` (Rust 后端)

**Step 1: 创建项目**
```bash
cd ~/.openclaw/workspace
# Tauri v2 交互式创建，选择 React + TypeScript + Vite
cargo create-tauri-app pawse --template react-ts --manager npm
```

**Step 2: 验证项目可用**
```bash
cd apps/pawse
npm install
npm run tauri dev
```
预期: 窗口弹出显示默认 React 页面

**Step 3: 提交**
```bash
git add apps/pawse/ && git commit -m "feat: init Tauri v2 + React TS project"
```

---

## Task 2: 安装配置 TailwindCSS + Framer Motion

**Files:**
- Modify: `apps/pawse/package.json`
- Modify: `apps/pawse/src/App.tsx`
- Modify: `apps/pawse/src/index.css`

**Step 1: 安装依赖**
```bash
cd apps/pawse
npm install -D tailwindcss @tailwindcss/vite
npm install framer-motion
```

**Step 2: 配置 TailwindCSS v4 (PostCSS 方式)**
- 修改 `vite.config.ts` 加入 tailwind 插件
- 在 `src/index.css` 加入 `@import "tailwindcss"`

**Step 3: 在 App.tsx 加一个 Framer Motion 测试动画**
```tsx
import { motion } from "framer-motion";

function App() {
  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.5 }}
      animate={{ opacity: 1, scale: 1 }}
      transition={{ duration: 1 }}
      className="flex items-center justify-center h-screen bg-gradient-to-br from-purple-400 to-blue-500"
    >
      <h1 className="text-4xl font-bold text-white">Pawse 🐾</h1>
    </motion.div>
  );
}
```

**Step 4: 验证**
```bash
npm run tauri dev
```
预期: Pawse logo 渐变出现

**Step 5: 提交**
```bash
git add . && git commit -m "feat: add TailwindCSS + Framer Motion"
```

---

## Task 3: 基础路由 + 页面骨架

**Files:**
- Create: `apps/pawse/src/pages/Welcome.tsx`
- Create: `apps/pawse/src/pages/UploadPet.tsx`
- Create: `apps/pawse/src/pages/GeneratePet.tsx`
- Create: `apps/pawse/src/pages/Dashboard.tsx`
- Create: `apps/pawse/src/pages/Settings.tsx`
- Modify: `apps/pawse/src/App.tsx`
- Install: `react-router-dom`

**Step 1: 安装 react-router-dom**
```bash
npm install react-router-dom
```

**Step 2: 创建 5 个页面组件（每个含标题和基本布局）**

每个页面是简单的 Tailwind 布局，例如:
```tsx
// Welcome.tsx
export default function Welcome() {
  return (
    <div className="flex flex-col items-center justify-center h-screen">
      <h1 className="text-3xl">Welcome to Pawse</h1>
      <p>Your pet cares about you</p>
    </div>
  );
}
```

**Step 3: App.tsx 配路由**
```tsx
import { BrowserRouter, Routes, Route } from "react-router-dom";
import Welcome from "./pages/Welcome";
import UploadPet from "./pages/UploadPet";
// ...

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Welcome />} />
        <Route path="/upload" element={<UploadPet />} />
        <Route path="/generate" element={<GeneratePet />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </BrowserRouter>
  );
}
```

**Step 4: 验证**
```bash
npm run tauri dev
```
预期: 路由可切换，能看到每个页面的标题

**Step 5: 提交**
```bash
git add . && git commit -m "feat: routing + 5 page skeletons"
```

---

## Task 4: SQLite 数据库层

**Files:**
- Create: `apps/pawse/src/services/database.ts`
- Modify: `apps/pawse/src-tauri/Cargo.toml`
- Add: `tauri-plugin-sql`

**Step 1: 安装 tauri-plugin-sql**
```bash
cd apps/pawse
npm install @tauri-apps/plugin-sql
cd src-tauri && cargo add tauri-plugin-sql --features sqlite && cd ..
```

**Step 2: 注册插件**
- 在 `src-tauri/src/lib.rs` 中注册 `tauri_plugin_sql::init()`
- 在 `src-tauri/tauri.conf.json` 中配置 capabilities

**Step 3: 创建 database.ts 服务层**
```typescript
// src/services/database.ts
import Database from "@tauri-apps/plugin-sql";

let db: Database | null = null;

export async function initDB() {
  db = await Database.load("sqlite:pawse.db");
  await db.execute(`
    CREATE TABLE IF NOT EXISTS pets (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      image_path TEXT NOT NULL,
      generated_asset_path TEXT,
      style TEXT DEFAULT 'cartoon',
      created_at TEXT DEFAULT (datetime('now'))
    )
  `);
  await db.execute(`
    CREATE TABLE IF NOT EXISTS settings (
      id INTEGER PRIMARY KEY CHECK (id = 1),
      work_duration INTEGER DEFAULT 50,
      break_duration INTEGER DEFAULT 10,
      overlay_mode TEXT DEFAULT 'gentle',
      animation_enabled INTEGER DEFAULT 1,
      auto_launch INTEGER DEFAULT 0
    )
  `);
  // Insert default settings if not exists
  await db.execute(`
    INSERT OR IGNORE INTO settings (id) VALUES (1)
  `);
  return db;
}

export function getDB() {
  if (!db) throw new Error("DB not initialized");
  return db;
}
```

**Step 4: 验证 DB 初始化**
```bash
npm run tauri dev
```
在浏览器 devtools 中检查 DB 是否正常创建

**Step 5: 提交**
```bash
git add . && git commit -m "feat: SQLite database layer with pets & settings tables"
```

---

## Task 5: 工作计时器系统

**Files:**
- Create: `apps/pawse/src/services/timer.ts`
- Create: `apps/pawse/src/hooks/useTimer.ts`
- Create: `apps/pawse/src/stores/timerStore.ts`

**Step 1: 安装 zustand（状态管理）**
```bash
npm install zustand
```

**Step 2: 创建 timerStore**
```typescript
// timerStore.ts
import { create } from 'zustand';

type TimerState = 'IDLE' | 'WORKING' | 'BREAK_TRIGGERED' | 'OVERLAY_ACTIVE' | 'BREAK_FINISHED';

interface TimerStore {
  state: TimerState;
  workElapsed: number;      // seconds
  breakElapsed: number;     // seconds
  workDuration: number;     // seconds (default 50min = 3000s)
  breakDuration: number;    // seconds (default 10min = 600s)
  isIdle: boolean;
  
  startWork: () => void;
  pauseWork: () => void;
  triggerBreak: () => void;
  dismissOverlay: () => void;
  finishBreak: () => void;
  tick: () => void;
  setIdle: (idle: boolean) => void;
}
```

**Step 3: 创建 useTimer hook**
监听键盘/鼠标事件，更新 idle 状态和 tick。

**Step 4: 创建 timer.ts 服务**
```typescript
// 导出 ActivityTracker 类
// 监听 mousemove, keydown, mousedown
// 5 分钟无操作 → setIdle(true)
// 有操作 → setIdle(false)
```

**Step 5: 写测试**
```bash
# 测试状态机转换逻辑
cd apps/pawse && npx vitest
```

**Step 6: 提交**
```bash
git add . && git commit -m "feat: timer state machine with idle detection"
```

---

## Task 6: Overlay 窗口

**Files:**
- Modify: `apps/pawse/src-tauri/src/lib.rs` — 多窗口管理
- Create: `apps/pawse/src/overlay/OverlayWindow.tsx`
- Create: `apps/pawse/src/overlay/OverlayApp.tsx` (独立入口)
- Modify: `apps/pawse/src-tauri/tauri.conf.json`

**Step 1: Rust 侧创建 Overlay 窗口**
```rust
// 在 lib.rs 中添加新窗口:
// - always_on_top: true
// - transparent: true
// - decorations: false
// - skip_taskbar: true
```

**Step 2: 创建 OverlayApp.tsx**
类似一个新 Vite 入口，专门渲染 Overlay 窗口。它通过 Tauri event 系统接收主窗口消息。

**Step 3: 验证窗口**
```bash
npm run tauri dev
```
预期: 主窗口 + 一个小 Overlay 透明窗口同时出现

**Step 4: 提交**
```bash
git add . && git commit -m "feat: overlay window with always-on-top transparent"
```

---

## Task 7: 宠物上传 UI

**Files:**
- Create: `apps/pawse/src/components/FileUpload.tsx`
- Modify: `apps/pawse/src/pages/UploadPet.tsx`

**Step 1: 创建 FileUpload 组件**
- 拖拽 + 点击上传
- 支持 PNG/JPG/WebP
- 限制 10MB
- 预览原图

**Step 2: 图片校验**
- 文件类型检查
- 文件大小检查
- 错误提示

**Step 3: 提交**
```bash
git add . && git commit -m "feat: pet photo upload with drag-drop & validation"
```

---

## Task 8: AI 宠物生成（OpenAI + rembg）

**Files:**
- Create: `apps/pawse/src/services/openai.ts`
- Create: `apps/pawse/src/services/rembg.ts`
- Modify: `apps/pawse/src/pages/GeneratePet.tsx`

**Step 1: 创建 openai.ts**
```typescript
// 调用 OpenAI Images API (DALL·E 3)
// 参数: 用户照片 (base64) + 风格 prompt
// 返回: 生成图片 URL

const API_KEY = await getFromConfig(); // 或 tauri-plugin-store

async function generatePetImage(
  userPhoto: string, // base64
  style: 'cartoon' | 'pixel' | 'sticker' | 'cozy',
): Promise<string> {
  // POST https://api.openai.com/v1/images/generations
  // model: "dall-e-3"
  // 通过 image 参数 + prompt 进行 image-to-image 变体
}
```

**Step 2: rembg 后处理**
通过 Tauri shell 插件调用本地 Python rembg:
```typescript
// rembg.ts
async function removeBackground(inputPath: string, outputPath: string) {
  await Command.create("python3", [
    "-m", "rembg",
    "i", inputPath, outputPath
  ]).execute();
}
```

**Step 3: 生成页面 UI**
- 显示上传的宠物照片
- 风格选择器（4 种风格）
- "生成"按钮 + 加载状态
- 生成结果预览

**Step 4: 提交**
```bash
git add . && git commit -m "feat: AI pet generation with DALL-E 3 + rembg"
```

---

## Task 9: 动画系统

**Files:**
- Create: `apps/pawse/src/animations/idle.ts`
- Create: `apps/pawse/src/animations/blink.ts`
- Create: `apps/pawse/src/animations/sleep.ts`
- Create: `apps/pawse/src/animations/pawBlock.ts`
- Create: `apps/pawse/src/components/PetCharacter.tsx`

**Step 1: PetCharacter 组件**
```tsx
// 桌宠主组件
// 接收: animation state, pet image, position
// 通过 Framer Motion 驱动不同动画

interface PetCharacterProps {
  petImage: string;
  state: 'idle' | 'blink' | 'sleep' | 'pawBlock';
  position: Position;
}
```

**Step 2: 各动画实现**
- idle: 底部浮动 + 呼吸缩放
- blink: 短暂闭眼（不透明度变化模拟眨眼）
- sleep: 旋转 90° + 缩放 (Zzz 文字)
- pawBlock: 从底部伸出爪子覆盖屏幕区域

**Step 3: 提交**
```bash
git add . && git commit -m "feat: pet animation system with idle/blink/sleep/pawBlock"
```

---

## Task 10: 提醒系统集成

**Files:**
- Modify: `apps/pawse/src/stores/timerStore.ts`
- Modify: `apps/pawse/src/overlay/OverlayWindow.tsx`
- Create: `apps/pawse/src/components/ReminderBanner.tsx`

**Step 1: 触发逻辑**
- 达到工作时长 → timerStore 触发 `triggerBreak()`
- Overlay 窗口收到事件 → 显示宠物
- 宠物显示提醒文案 + 倒计时

**Step 2: 自定义文案**
- 支持用户设置提醒文本
- 随机选择或用户自定义

**Step 3: 休息倒计时**
在 Overlay 上显示剩余休息时间

**Step 4: 提交**
```bash
git add . && git commit -m "feat: break reminder system with custom text"
```

---

## Task 11: 设置系统

**Files:**
- Modify: `apps/pawse/src/pages/Settings.tsx`
- Create: `apps/pawse/src/services/settings.ts`

**Step 1: 设置界面**
- 工作时间滑块（15-120 分钟）
- 休息时间滑块（1-30 分钟）
- Overlay 模式选择（温柔/半屏/全屏）
- 动画开关
- 开机启动开关
- 宠物名称输入
- 宠物风格选择
- 提醒文案编辑

**Step 2: 设置持久化**
- 通过 database.ts 读写 SQLite
- 修改即时生效

**Step 3: 提交**
```bash
git add . && git commit -m "feat: settings page with persistence"
```

---

## Task 12: Dashboard 主控制台

**Files:**
- Modify: `apps/pawse/src/pages/Dashboard.tsx`
- Create: `apps/pawse/src/components/WorkStatus.tsx`
- Create: `apps/pawse/src/components/PetPreview.tsx`
- Create: `apps/pawse/src/components/TimerDisplay.tsx`

**Step 1: Dashboard 布局**
- 顶部: 宠物头像 + 名称
- 中间: 计时器圆环显示
- 下方: 快捷操作（开始工作 / 休息 / 设置）

**Step 2: 计时器圆环**
使用 SVG circle + CSS animation 显示进度

**Step 3: 提交**
```bash
git add . && git commit -m "feat: main dashboard with timer display"
```

---

## Task 13: 流程串联 + 首次启动引导

**Files:**
- Modify: `apps/pawse/src/App.tsx`
- Modify: `apps/pawse/src-tauri/src/lib.rs`

**Step 1: 首次启动检测**
- 检查 pets 表是否有数据
- 无数据 → 跳转到 Welcome → Upload → Generate 流程
- 有数据 → 直接进 Dashboard

**Step 2: 完整流程串联**
- Welcome → 上传宠物 → AI 生成 → 进入 Dashboard
- Dashboard ←→ Settings
- 计时器触发 → Overlay 出现 → 休息结束 → 回到 Dashboard

**Step 3: 提交**
```bash
git add . && git commit -m "feat: full onboarding flow + state integration"
```

---

## Task 14: 打包配置

**Files:**
- Modify: `apps/pawse/src-tauri/tauri.conf.json`
- Create: `apps/pawse/src-tauri/ubuntu/build.conf`
- Create: 打包脚本

**Step 1: Ubuntu 打包配置**
```bash
npm run tauri build
```
输出: `.deb` / `.AppImage`

**Step 2: 配置 macOS/Windows（在对应 CI 环境运行）**
- macOS: `.dmg`
- Windows: `.msi`

**Step 3: 验证打包结果**
安装包大小 < 30MB

**Step 4: 提交**
```bash
git add . && git commit -m "chore: packaging config for Ubuntu/macOS/Windows"
```

---

## 附录：Python rembg 安装

```bash
# 安装 rembg (需要在运行环境中安装)
pip install rembg

# 首次运行会下载模型 ~200MB
python3 -m rembg i input.jpg output.png
```

## 附录：OpenAI API Key 配置

通过 Tauri plugin-store 安全存储 API Key:
- 首次使用时弹出输入框
- 存储到 OS keychain / encrypted local file
- 可在 Settings 中修改
