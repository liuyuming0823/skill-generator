---
# ══════════════════════════════════════════════════════════════════════
#  ▍技能骨架模板 — skill-generator / templates
#
#  用法（两条路，优先第一条）：
#    A. 自动生成：python scripts/new_skill.py <skill-name> --desc "..." \
#                    --desc-zh "..." --desc-en "..." --triggers a,b,c
#    B. 手工复制：把本文件拷成 <技能目录>/SKILL.md，逐项替换占位值后，
#                 删掉本 frontmatter 内所有 # 注释行，再跑体检。
#
#  ⚠️ 本文件首行必须是 ---（frontmatter 起始）。若在文件最上方加 HTML 注释块，
#     体检会直接判 P0「缺 frontmatter」——这是最容易被忽略的坑。
#
#  字段依据技能市场分发规范（YAML frontmatter + Markdown 正文）。
#  [必填] 缺一个就上架失败；[建议] 只影响展示；[可选] 用不到直接删行。
# ══════════════════════════════════════════════════════════════════════

# ── 标识 ────────────────────────────────────────────────────────────
name: your-skill-name                        # [必填] 小写字母+数字+连字符，必须与目录名一致
display_name: 你的技能展示名                  # [建议] 中文展示名，市场列表显示这个
display_name_en: Your Skill Name             # [建议] 英文展示名

# ── 描述三件套（各有各的用处，别只写一个）────────────────────────────
description: >-                              # [必填] 给模型看：做什么 / 何时触发 / 触发词
  一句话说清这个技能解决什么问题。当用户提到「触发词A」「触发词B」「触发词C」，
  或出现 <具体场景> 时使用。
description_zh: >-                           # [必填] 给中文用户看，30 字以内，别照抄 description
  一句话中文介绍
description_en: >-                           # [必填] 给英文用户看，首字母大写、结尾不加句号
  One-line English introduction

# ── 元信息 ──────────────────────────────────────────────────────────
category: development                        # [建议] 分类之一，取值须在平台枚举内（如 writing）
version: 1.0.0                               # [必填] 语义化版本，改动后必须递增
author: 你的署名                              # [建议] 个人或团队/公司名，别留机器默认值

# ── 可选开关（用不到就删）───────────────────────────────────────────
allowed-tools: Read, Write, Bash             # [可选] 工具白名单，逗号分隔
disable-model-invocation: false              # [可选] true = 只能用户手动调用，AI 不自动触发
user-invocable: true                         # [可选] false = 隐藏菜单，仅供 AI 内部调用

# ── WorkBuddy 本机扩展 ──────────────────────────────────────────────
agent_created: true                          # 让 Agent 能用 skill_manage 后续修改本技能
trigger:                                     # 结构化触发词，比 description 里的散文更稳
  - 触发词A
  - 触发词B
---

# 技能展示名 (your-skill-name)

一句话说明它替用户省掉了什么。

## 何时使用

- 用户提到「触发词A」时
- 场景二

## 运行前提

- 依赖：列出第三方包 / 外部可执行文件；有第三方包必须配 `scripts/setup.py`
- 配置：真实值放 `~/.workbuddy/<name>_config.json`，对外只留 `config/settings.example.json` 空模板
- 首次配置：`python scripts/setup.py`

## 执行步骤

当用户需要 <做什么> 时，按以下步骤执行：

1. **<步骤名>** — 具体命令与判断条件。
   ```bash
   python scripts/<脚本>.py --参数 <值>
   ```
2. **<步骤名>** — 说明输入、输出、失败时的行为（缺依赖 / 缺配置 / 无网络）。
3. **<步骤名>** — 产出交付给用户，并说明输出文件放在哪。

## 目录说明

- `scripts/` — 执行逻辑
- `references/` — 按需加载的参考文档
- `templates/` — 可复制的骨架 / 报告模板
- `config/` — 只放 `*.example.*` 空模板

> 只保留真实存在的目录条目；写了不存在的文件会被体检判成「引用缺失」。

## 参考资料

- `references/<文档>.md` —— 什么时候需要读它（按需加载，不要默认全读）

## 常见坑

1. <踩过的坑 + 规避方式> —— 这一节是技能最值钱的部分，务必写

<!--
  写完后跑：
    python scripts/audit_skill.py "<技能目录>" --market     # 体检
    python scripts/pack_skill.py  "<技能目录>" --install    # 打包 + 装到本机

  注意：本注释块是文件末尾的说明，不要挪到文件开头。
-->
