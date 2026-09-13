---
title: 海外代购项目 · 设计系统 Style Kit
date: 2026-08-26
tags: [design-system, style-kit, frontend, ui, tokens]
source: ai-conversation
---

# 海外代购项目 · 设计系统 Style Kit

> **用途**：给 AI / Pencil / pencli 读取的唯一设计规范源。所有新页面、新组件必须严格对齐本文件，禁止出现第二套设计令牌。
> **参考基线**：`原型-代购商城.html`
> **生效范围**：整个「海外代购业绩提升」项目下的所有原型 / 页面 / 组件

---

## 一、Design Tokens（绝对不可修改，所有页面必须共用同一套）

### 1. 颜色体系

| Token 名 | 亮色值 | 暗色值 | 语义 |
|---|---|---|---|
| `--brand` | `#7C3AED` | `#A78BFA` | 品牌主色（紫）· 主按钮 / 强调 / 链接 hover |
| `--brand-light` | `#EDE9FE` | `#2E1065` | 品牌浅色 · 选中态背景 / badge |
| `--brand-dark` | `#5B21B6` | `#C4B5FD` | 品牌深色 · 主按钮 hover |
| `--accent` | `#059669` | `#34D399` | 成功 / 现货 / 正收益 |
| `--accent-light` | `#D1FAE5` | `#064E3B` | 成功背景 |
| `--warning` | `#D97706` | `#FBBF24` | 警告 / 低库存 / 陈旧数据 |
| `--warning-light` | `#FEF3C7` | `#451A03` | 警告背景 |
| `--danger` | `#DC2626` | `#F87171` | 错误 / 缺货 / 删除 |
| `--danger-light` | `#FEE2E2` | `#450A0A` | 错误背景 |
| `--surface` | `#FFFFFF` | `#1F2937` | 一级表面 · 卡片 / header / 弹窗 |
| `--surface-muted` | `#F9FAFB` | `#111827` | 二级表面 · 页面背景 / 搜索框背景 |
| `--surface-alt` | `#F3F4F6` | `#374151` | 三级表面 · sort-tab hover / badge-neutral / 图片占位 |
| `--border` | `#E5E7EB` | `#374151` | 默认边框 · 卡片 / 表格行 / 分隔线 |
| `--border-strong` | `#D1D5DB` | `#4B5563` | 强边框 · outline-btn / 重点分隔 |
| `--text` | `#111827` | `#F9FAFB` | 主文字 · 标题 / 正文 |
| `--text-secondary` | `#4B5563` | `#D1D5DB` | 次文字 · 描述 / 次要信息 |
| `--text-muted` | `#9CA3AF` | `#6B7280` | 弱文字 · 辅助 / 占位 / 角注 |

### 2. 字体系统

```
--font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
             "Helvetica Neue", Arial, "PingFang SC", "Microsoft YaHei", sans-serif;
--font-mono: "SF Mono", "Fira Code", "Cascadia Code", Consolas, monospace;
```

**字号阶梯（只有这 7 档，禁止出现其他字号）**：

| 字号 | 使用场景 |
|---|---|
| 20px / 800 | 型号主标题（detail-mpn） / 主价格 |
| 18px / 700 | 页面总标题（page-title） / Logo 文字 |
| 15px / 700 | 列表型号号（product-mpn） |
| 14px / 400~600 | **正文基准字号** · 表格内容 / 按钮 / Nav / 筛选项 / 描述 |
| 13px / 400~700 | 次要文字 · sort-tab / 商品描述 / detail-card-title / 返回按钮 |
| 12px / 400~600 | 小字 · badge / sidebar-title / 角注 / 删除价 / freshness |
| 11px / 400~700 | 极小字 · 表格表头 / footer / 说明 / 角标 annotation |

> **关键约束**：字号只能是 11 / 12 / 13 / 14 / 15 / 18 / 20，禁止 16、17、19 等中间值。

### 3. 圆角体系（只有 3 档）

| Token | 值 | 使用场景 |
|---|---|---|
| `--radius` | 6px | 按钮 / 图片 / sort-tab / 小型组件 |
| `--radius-card` | 10px | 卡片（product-card / detail-card / mall-root 根容器） |
| `--radius-full` | 999px | 胶囊 · 搜索框 / badge / expand-btn |

> **禁止出现 4px / 8px / 12px / 16px 等非标准圆角值。**

### 4. 间距系统（4px 栅格）

只能使用：`2 / 3 / 4 / 6 / 8 / 10 / 12 / 14 / 16 / 18 / 20 / 24px`。

常见组合：
- 卡片内边距：12px 14px / 14px 16px
- 按钮内边距：7px 14px（不可拆）
- 表单项 / 列表上下间距：4 / 6 / 8 / 10px
- 区块之间：10 / 16px

---

## 二、组件规范（新页面必须复用，不允许重新发明）

### 1. 按钮 `.btn`

```css
.btn {
  padding: 7px 14px;              /* 固定，不可改 */
  border-radius: var(--radius);   /* 6px */
  font-size: 13px;                /* 固定 */
  font-weight: 600;               /* 固定 semi-bold */
  border: none;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}
```

**两种变体，禁止出现第三种**：

| 变体 | 类名 | 样式 |
|---|---|---|
| 主按钮 | `.btn-primary` | bg=--brand, color=#fff, hover→--brand-dark |
| 描边按钮 | `.btn-outline` | bg=--surface, border=1px solid --border-strong, color=--text, hover→border/color=--brand |

> 特殊：搜索框右侧按钮 `.mall-search-btn` 使用胶囊（--radius-full）+ 紫色填充 + 7px 18px 内边距，仅限 Header 搜索栏使用，不可在别处复用。

### 2. 输入框 Input（**用户重点关注**）

**只有两种形态，禁止出现第三种**：

#### 形态 A：内嵌式输入框（默认，占 90% 场景）

```css
/* 放在搜索条/表单组容器内，容器负责边框 */
.container {
  background: var(--surface-muted);
  border: 2px solid var(--brand);     /* 2px，紫色，聚焦高亮 */
  border-radius: var(--radius-full);  /* 或 --radius，看容器类型 */
  padding: 2px 4px 2px 14px;
}
.container input {
  flex: 1;
  border: none;                       /* input 自身无边框 */
  background: transparent;
  outline: none;
  font-size: 14px;                    /* 固定字号 */
  color: var(--text);
  padding: 6px 0;                     /* 固定高度 */
}
.container input::placeholder { color: var(--text-muted); }
```

**示例**：Header 搜索框就是这种，容器给 2px 紫色边框，input 本身透明无边框。

#### 形态 B：裸复选框（用于 sidebar 筛选）

```css
input[type="checkbox"] {
  accent-color: var(--brand);   /* 选中色跟随品牌紫 */
  width: 14px;                  /* 固定尺寸 */
  height: 14px;                 /* 固定尺寸 */
  cursor: pointer;
}
```

> **绝对禁止**：不要给 input 自己加 1px 灰色边框 + 4px padding 那种"浏览器默认风"的输入框。要么走容器包边（形态 A），要么是 14px 复选框。

### 3. 卡片 `.product-card` / `.detail-card`

```css
/* 基础卡片 */
{
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-card);  /* 10px */
  padding: 12px 14px;
  margin-bottom: 10px;
}

/* hover 态（可点击卡片必须有） */
:hover {
  border-color: var(--brand);
  box-shadow: 0 0 0 1px var(--brand);   /* 1px 紫色外发光，不是真实阴影 */
}
```

> 禁止卡片有 2px 以上的边框或重阴影；禁止卡片用 6px 圆角。

### 4. Badge `.badge`

```css
.badge {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  border-radius: var(--radius-full);   /* 胶囊 */
  padding: 2px 8px;                    /* 固定 */
  font-size: 11px;                     /* 固定 */
  font-weight: 600;
  line-height: 1.4;
}
```

**5 种语义色，和 Tokens 一一对应**：

| 类名 | 背景 | 文字 |
|---|---|---|
| `.badge-brand` | --brand-light | --brand-dark |
| `.badge-accent` | --accent-light | --accent |
| `.badge-warning` | --warning-light | --warning |
| `.badge-danger` | --danger-light | --danger |
| `.badge-neutral` | --surface-alt | --text-secondary |

### 5. 表格

**两种表格规格**：

| 类型 | 类名 | 表头字号 | 内容字号 | 表头下划线 |
|---|---|---|---|---|
| 主表格（大） | `.offer-table` | 11px / 700 | 13px | 2px solid --border |
| 迷你表格（小） | `.mini-offer-table` | 11px / 700 | 12px | 2px solid --border |

共同规则：
- `border-collapse: collapse`
- hover 行背景 `--surface-muted`
- 行之间 `1px solid --border`
- 表头文字颜色 `--text-muted`

### 6. 布局容器

| 容器 | 最大宽度 | 背景 | 边框 |
|---|---|---|---|
| `.mall-root` 页面根 | 1100px，居中 | --surface-muted | 1px solid --border + 极淡阴影 |
| `.detail-grid` 两列卡片栅格 | `1fr 1fr`，<=700px 切单列 | — | gap: 10px |

---

## 三、页面结构标准骨架（每个新页面必须照此套）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>页面标题</title>
<style>
  /* 第一步：粘贴 §一 的完整 Tokens（亮色 + 暗色），不得修改值 */
  /* 第二步：粘贴 §二 的公共组件样式（btn/badge/table/card），不得修改 */
  /* 第三步：追加本页面独有样式，命名必须遵循 mall- / detail- / product- / filter- / sidebar- 前缀 */
</style>
</head>
<body>

  <div class="page-title">页面大标题（18px/700）</div>
  <div class="page-subtitle">副标题 / 说明（13px，muted）</div>

  <div class="mall-root">
    <!-- Header：Logo + 搜索 + Nav（如果需要） -->
    <!-- Body：Sidebar + Main（如果是列表页），或 Main（如果是详情页） -->
    <!-- Footer：小字说明 -->
  </div>

</body>
</html>
```

---

## 四、AI 生成 / Pencil 设计时的硬检查清单（Checklist）

每次 AI 产出页面后，先跑一遍：

- [ ] Tokens 是否与 §一 完全一致？（紫色是不是 #7C3AED，圆角三档是否对）
- [ ] 字号是否在 11/12/13/14/15/18/20 内？
- [ ] 输入框是否为 §2.2 的两种形态？有没有出现 1px 灰边 + 4px padding 的"野生 input"？
- [ ] 按钮 padding 是否为 7px 14px？圆角是否 6px（主按钮）或 999px（搜索按钮）？
- [ ] 卡片圆角是否 10px？hover 是紫色 1px 外发光而不是 box-shadow 重阴影？
- [ ] 新组件是否复用了现有类名体系前缀（mall- / detail- / product- / filter-）？
- [ ] 是否有暗色模式的 `prefers-color-scheme: dark` 覆盖？

---

## 五、如何在 Pencil / pencli 中应用本 Style Kit

### 方案 A：作为 Style Kit 附件（推荐）

1. 在 Pencil 中打开任意一个旧页面（如代购商城原型）
2. 右侧面板 → Style / Variables → 导出为 Style Kit（命名 `LCSC代购·StyleKit`）
3. 新建 Frame 时 → 选择 "Attach Style Kit" → 选择上面这个 Kit
4. Agent 对话里附加一句：**「严格使用本画布附着的 Style Kit 中的颜色、字体、圆角变量，任何组件不得脱离变量直接写死色值」**

### 方案 B：作为 Project Rules 注入（本仓库已同步更新 CLAUDE.md §8）

在任何 AI 对话（Pencil Agent / Claude / Trae）里，让它先读本文件，再生成页面。

### 方案 C：作为 Prompt 前缀（最省事，无需配置工具）

每次生成页面前，粘贴下面模板：

> **模板内容见 `AI生成页面Prompt模板.md`**
