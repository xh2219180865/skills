---
name: frontend-ui
description: 配合 constitution.md 使用的 React 18 + TypeScript 前端开发技能。集成 "Design System" 策略，强制要求通过 Ant Design Token + Tailwind CSS 定制独特 UI 主题（拒绝默认样式）。包含 Axios 拦截器(code!=0异常)、Zustand 状态管理及严格类型定义。Use this skill to generate production-grade React code that is both strictly typed and aesthetically distinctive.
---

# 前端工程与设计规范 (Frontend Engineering & Design)

本技能用于生成**高质量、高颜值、高健壮性**的 React 前端代码。**必须配合 `constitution.md` 使用。**

## 核心规则 (Red Lines)

### 红线 1：工程与类型 (Engineering)
- **禁止**：使用 Class Component。**必须**全量使用 Function Component (FC) + Hooks。
- **禁止**：在 Props 或 State 中使用 `any`。**必须**定义明确的 Interface。
- **必须**：所有 API 响应必须通过 `CommonResult` 包装，前端拦截器自动处理 `code != 0` 业务异常。
- **必须**：ID 字段（Long 类型）在前端 Interface 中必须声明为 `string`，防止精度丢失。

### 红线 2：设计与美学 (Design & Aesthetics)
- **禁止**：直接使用 Ant Design 默认的“阿里蓝”主题。**必须**配置 `ConfigProvider` 的 Design Token。
- **禁止**：生成千篇一律的 AI 风格界面。**必须**根据业务场景（如：极简、工业、SaaS）设定明确的色彩与排版策略。
- **优先**：使用 Tailwind CSS 处理布局 (`flex`, `grid`)、间距 (`p-6`, `gap-4`) 和排版 (`text-slate-600`)。
- **防御**：AntD 组件样式定制需通过 Token 全局配置，特殊情况使用 `css modules`，避免行内 `style`。

### 红线 3：状态与交互 (State & Interaction)
- **状态**：全局状态使用 Zustand，局部状态使用 `useState`。
- **反馈**：所有异步操作必须有 `loading` 状态，失败必须有 `message.error`（拦截器兜底），成功必须有反馈。

## 标准模版 (Standard Templates)

### 模版 0：设计系统配置 (Theme Config) —— **项目启动必选**

**目录**: `src/theme/antd.ts` & `tailwind.config.js`
**原则**: 建立 AntD 与 Tailwind 的颜色映射，确立“反平庸”的视觉基调。

```typescript
// src/theme/antd.ts
import type { ThemeConfig } from 'antd';

// 示例：采用 "Modern Slate" 风格（深灰主色，锐利圆角）
export const themeConfig: ThemeConfig = {
  token: {
    colorPrimary: '#0f172a', // 对应 Tailwind slate-900
    borderRadius: 2,         // 锐利风格
    fontFamily: '"Inter", "PingFang SC", sans-serif',
    colorBgContainer: '#ffffff',
    colorTextBase: '#334155', // slate-700
  },
  components: {
    Button: {
      controlHeight: 40, // 更大的点击区域
      fontWeight: 500,
    },
    Table: {
      headerBg: '#f8fafc', // slate-50
      headerSplitColor: 'transparent',
    },
    Card: {
      boxShadowTertiary: '0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px -1px rgba(0, 0, 0, 0.1)', // Tailwind shadow-sm
    }
  }
};