# AI 生成页面 Prompt 模板（复制粘贴即用）

> **用途**：把下面整段粘贴到任何 AI（Pencil Agent / Trae / Claude / v0 / bolt.new）提示词的最开头，后面再接你的具体页面需求描述。

---

```
【设计约束 · 强制执行】

项目：海外代购业绩提升
参考基线文件：09-业务项目/海外代购业绩提升/原型-代购商城.html
完整设计规范：09-业务项目/海外代购业绩提升/设计系统-StyleKit.md
请先完整读取以上两个文件后再开始生成。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
一、Design Tokens（值不可改，必须用 CSS 变量 --xxx 引用）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

■ 颜色（亮色模式 / 暗色模式成对出现）：
  --brand: #7C3AED / #A78BFA          品牌紫
  --brand-light: #EDE9FE / #2E1065    品牌浅
  --brand-dark: #5B21B6 / #C4B5FD     品牌深
  --accent: #059669 / #34D399         成功绿
  --accent-light: #D1FAE5 / #064E3B
  --warning: #D97706 / #FBBF24        警告黄
  --warning-light: #FEF3C7 / #451A03
  --danger: #DC2626 / #F87171         错误红
  --danger-light: #FEE2E2 / #450A0A
  --surface: #FFFFFF / #1F2937        一级表面（卡片）
  --surface-muted: #F9FAFB / #111827  二级表面（页面背景）
  --surface-alt: #F3F4F6 / #374151    三级表面（占位/badge中性）
  --border: #E5E7EB / #374151         默认边框
  --border-strong: #D1D5DB / #4B5563  强边框
  --text: #111827 / #F9FAFB           主文字
  --text-secondary: #4B5563 / #D1D5DB 次文字
  --text-muted: #9CA3AF / #6B7280     弱文字

■ 字体：
  --font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
               "Helvetica Neue", Arial, "PingFang SC", "Microsoft YaHei", sans-serif;
  --font-mono: "SF Mono", "Fira Code", "Cascadia Code", Consolas, monospace;

■ 字号阶梯（只有这 7 档，禁止出现其他值）：
  20(800) / 18(700) / 15(700) / 14(400~600) / 13(400~700) / 12(400~600) / 11(400~700)
  禁止 16、17、19 等中间值。

■ 圆角（只有这 3 档）：
  --radius: 6px        按钮 / 图片 / 小组件
  --radius-card: 10px  卡片 / 大容器
  --radius-full: 999px 胶囊 / badge / 搜索按钮
  禁止 4 / 8 / 12 / 16 等非标准值。

■ 间距（4px 栅格）：只能用 2/3/4/6/8/10/12/14/16/18/20/24px。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
二、组件形态（禁止出现第四种 input！）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

■ 输入框（只有两种，没有第三种）：
  形态A-内嵌式：用容器包 2px solid var(--brand) 紫色边框，input 本身 border:none;
         background:transparent; padding:6px 0; font-size:14px;。
  形态B-复选框：14×14px，accent-color: var(--brand)。
  ⚠️ 禁止：1px 灰色独立边框 + 4px padding 那种 AI 最爱的"野生 input"。

■ 按钮（只有两种）：
  padding: 7px 14px（固定，不可改）
  border-radius: var(--radius) 即 6px
  font-size: 13px; font-weight: 600
  主按钮 .btn-primary: bg=--brand, color=#fff
  描边 .btn-outline: bg=--surface, border=1px --border-strong, hover→紫边紫字
  例外：Header 搜索按钮用胶囊（--radius-full），仅限此处。

■ 卡片（只有一种）：
  bg=--surface, border=1px --border, border-radius=10px, padding=12~14px
  hover: border-color=--brand; box-shadow: 0 0 0 1px var(--brand)（1px紫色外发光，不是重阴影）

■ Badge：圆角 999px，padding: 2px 8px，字号 11px/600，5 种语义色对应各 token-light。

■ 表格：表头 11px/700/muted，行间 1px --border，hover 行 --surface-muted。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
三、页面骨架（必须遵循）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

<head> 内 CSS 结构：
  1. :root { /* 全部 Tokens 亮色 */ }
  2. @media (prefers-color-scheme: dark) { :root { /* Tokens 暗色覆盖 */ } }
  3. reset（box-sizing/margin/padding）
  4. body 默认样式
  5. 通用组件样式（btn / card / badge / table / input）
  6. 本页面独有样式

<body> 内 DOM 结构：
  <div class="page-title">页面大标题（18px/700）</div>
  <div class="page-subtitle">副标题（13px/muted）</div>
  <div class="mall-root">  <!-- 最大 1100px 居中，圆角 10px -->
    <!-- 内容：Header + Body(可选Sidebar+Main) + Footer -->
  </div>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
四、生成后自检（回答以下问题，答错重做）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 紫色品牌色值是多少？全页面是否都走 --brand 变量而非写死？
2. 所有字号是否在 11/12/13/14/15/18/20 内？有没有 16/17/19？
3. 所有圆角是否在 6/10/999 三档内？有没有 4/8/12？
4. Input 是哪两种形态？有没有第三种未授权的独立灰边 input？
5. 按钮 padding 是否精确 7px 14px？字号 13？字重 600？
6. 卡片圆角是否 10px？hover 是否为 1px 紫色外发光？
7. 是否包含 @media (prefers-color-scheme: dark) 暗色模式覆盖？

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【具体页面需求】：
（请在这里粘贴你这次要做什么，比如"做一个登录页，包含账号密码输入框、记住我、登录按钮"）
```
