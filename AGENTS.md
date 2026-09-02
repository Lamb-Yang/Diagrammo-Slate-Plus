# AGENTS.md

面向在本仓库工作的 AI 编码代理的说明。人吃的介绍在 `README.md`,这里只写代理需要的。

## 项目概述

Diagrammo Slate Plus 是一个 Obsidian 主题:纯 CSS、无构建、无依赖、无 JS,基于
Demian Neidetcher 的 Diagrammo Slate,目标版本 Obsidian 1.13+(`manifest.json`
的 `minAppVersion`)。全部样式都在根目录的 `theme.css` 单文件中(约 3800 行,
内含面向 Style Settings 插件的 `/* @settings ... */` YAML 块)。

## 仓库结构

- `theme.css` — 全部主题样式;`@settings` 块是 Style Settings 设置定义
- `manifest.json` — 主题元数据;`version` 字段是发布的唯一事实来源
- `versions.json` — 版本 → minAppVersion 映射,由 CI 自动维护,**不要手改**
- `.github/workflows/release.yml` — Release 流水线(见下文「验证与发布」)
- `docs/` — 截图与 `preview-note.md`(人工目测用的测试笔记)
- `README.md` — 面向用户的介绍

## 验证与发布

无构建系统,也没有 linter。改动 `theme.css` 后跑以下静态检查(最低门槛):

```bash
# 1. 括号 / 圆括号配对
python3 -c "s=open('theme.css').read(); assert s.count('{')==s.count('}') and s.count('(')==s.count(')')"

# 2. @settings YAML 可解析(需要 ruby)
ruby -ryaml -e 'YAML.safe_load(File.read("theme.css")[/\/\*\s*@settings(.*?)\*\//m, 1]); puts "yaml ok"'

# 3. 扫一遍所有 calc(var(--ds-*)):带 format: px/rem 的设置项必须直接消费,
#    不得出现 calc(var(--ds-x, 10) * 1px) 这类单位叠加(见下方规则 2)
grep -n "calc(var(--ds-" theme.css
```

发布 = 把 `manifest.json` 的 `version` 改成新值并 push 到 `main`。CI 会自动维护
`versions.json`、打同名 tag、创建带 `theme.css` / `manifest.json` / `versions.json`
三个资产的 GitHub Release。**不要**手动建 tag 或 Release。版本号没变时流水线
直接跳过,所以任何提交都安全。

Git 约定:提交信息用中文一行式(如「修复表格不展示控制手柄的问题」),直接提交
`main`。注意本仓库有两个远程:`origin`(本仓库)与 `upstream`(上游原主题仓库)。
`gh` 命令务必带 `-R Lamb-Yang/Diagrammo-Slate-Plus`,否则会解析到 upstream 报 404。

## theme.css 的既有约定

动手前先读文件头注释和各分区注释。核心规则:

1. **变量分四层,不要跨层乱用**
   - `--ds-*`:Style Settings 的输入,在 `@settings` 块定义;
   - `--sl-*`:内部调色板,在 `.theme-light` / `.theme-dark` 中以 **RGB 三元组**
     定义,`body` 里派生完整颜色;
   - `--slate-*`:语义令牌(深度色阶 ladder、按首字母的 rotation、各处 alpha);
   - Obsidian 核心变量(`--text-*`、`--tag-*`、`--table-*` 等):主题只做赋值层。

2. **Style Settings 单位陷阱**:带 `format: px/rem` 的数字设置,插件写出变量时
   自带单位。消费时直接 `var(--ds-x, 10px)`,严禁 `calc(var(--ds-x, 10) * 1px)`
   ——用户拖过滑杆后变量带单位,calc 变成 px*px,声明在计算值阶段静默失效。

3. **半透明一律 color-mix 或 rgba(三元组变量)**;alpha 值集中定义为令牌
   (`--sl-wash`、`--slate-tint-N`、`--slate-quote-wash` 等),不要在使用点写死。

4. **主色/强调色走 HSL 管道**:`--sl-primary` / `--sl-accent` 由色相滑杆
   (`--ds-primary-hue` / `--ds-accent-hue`)派生。新颜色引用挂到这两个变量或
   色板色,不要引入新的硬编码色。

5. **黄色不作小号文字墨色**(浅色地面下对比度低于 3:1),只用于填充类场景
   (高亮、Canvas、插件色槽)。

6. **选择器纪律**:核心变量名与类名以官方文档为准
   (docs.obsidian.md → Reference → CSS variables,源文件在
   `obsidianmd/obsidian-developer-docs` 仓库);查不到的变量名要么实测要么不写。
   `!important` 只允许出现在 Mermaid 分区(Mermaid 往 SVG 注入 id 选择器样式,
   别无他法)。注意优先级:`body {}` 压不过 `.theme-light` / `.theme-dark`
   (0,1,0),覆盖它们时要重述主题类。

7. **双视图都要顾**:阅读视图用 `.markdown-rendered`,实时预览用
   `.markdown-source-view.mod-cm6`。行级类名(`HyperMD-header-N`、
   `HyperMD-quote-N`、`HyperMD-list-line-N` 等)只出现在 DOM 不在 app.css,
   不要因为 app.css 里搜不到就断定不可用。只适用于单一视图的规则必须明确
   限定作用域(参考 HyperMD-quote 圆角框只限定 `.is-live-preview` 的先例)。

8. **可访问性既定决策,配色体系改动时保持**:标签按首字母 8 桶配色、
   `slate-cvd-ladder` 色盲友好开关、`prefers-reduced-motion` 处理、
   标题/文件夹/列表/引用共用同一套深度色阶。

9. **`@media print` 分区**保持「去 wash、保墨色」的意图;改色带/光晕相关令牌时
   同步检查该分区(覆盖 `.theme-light`/`.theme-dark` 的值要用
   `body.theme-light, body.theme-dark`)。

10. **Callout 刻意不定制**(留给 Obsidian 默认),不要顺手加样式。

## 已确认的事实(避免重复调研)

- Canvas 卡片颜色:1.13 起核心以**完整颜色**消费 `--canvas-color-N`,旧的
  RGB 三元组写法会让边框和底色失效(文件头注释第 1 条)。
- `--embed-border-start` 是核心的嵌入侧边框变量,`--embed-border-left` 是旧名,
  两个都写(Minimal 主题同样双写)。
- `--color-accent-rgb` 核心不会从 HSL 三元组派生,需按当前主色手动同步
  (见 `.theme-light` / `.theme-dark`)。
- `--font-adaptive-normal` 是 Minimal 主题的变量,核心不消费;核心正文字号
  变量是 `--font-text-size`。
- Style Settings 只写用户改动过的设置;`title.zh` / `description.zh` 是其
  多语言键;源码在 `mgmeyers/obsidian-style-settings`。
- 文件内分区编号有错序(21 Mermaid 在 20 Print 之前),历史遗留,按注释
  标题找内容即可。
