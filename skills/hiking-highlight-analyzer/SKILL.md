---
name: hiking-highlight-analyzer
description: 原徒步精彩片段决策器的兼容入口，将已有分析转为选片、分镜和计划。默认户外徒步；明确指定家庭娱乐时由多场景决策器按家庭规则处理，不执行视频剪辑。
metadata:
  display_name: 徒步视频精彩片段决策器
  display_name_en: Hiking Highlight Analyzer
  description_zh: 基于已有的视频时间轴、关键帧描述、事件描述和转录结果，对徒步/户外长视频候选片段进行角色分类、价值评估、相似去重、保留决策和故事完整性检查，最终输出结构化
    edit-plan.yaml。
  description_en: Converts upstream video-analysis results into a structured editing
    decision plan for hiking and outdoor vlogs by classifying roles, evaluating segment
    value, deduplicating similar footage, making keep/drop decisions, and checking
    story integrity. It does not process raw video or execute editing commands.
  version: 2.0.0
  author: Sven
  user-invocable: 'True'
  disable-model-invocation: 'False'
---

# 原徒步决策器兼容入口

读取并执行 [vlog-highlight-analyzer](../vlog-highlight-analyzer/SKILL.md)，本目录旧版规则不再是默认执行依据。

- 显式场景由用户或导演传入，优先于本名称。无其他类型且明确调用原徒步决策器时采用 `outdoor-hiking`；家庭娱乐请求采用 `family-entertainment`。
- 原始视频先调用分析技能；已有证据保留原片位置、人物/声音描述和不确定性。不能自行补出景观、对白或事件。
- 旧计划可继续阅读，`journey` 评分按选定场景解释；新计划使用通用契约的 `progression`，保留完整源区间和选片理由。
- 输出分镜和 edit-plan.yaml，由共用 FFmpeg 技能执行。沿用本次交付授权，不因升级重新索取确认。

旧 references/templates 作为原资料保留，只有复查旧格式时按需读取；不得将其中徒步故事检查项强加到家庭模式。
