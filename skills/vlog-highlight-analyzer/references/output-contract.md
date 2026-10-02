# 分析、分镜与执行交接

## 分析报告：analysis.md

由上游分析技能生成，包含源片清单、所选 `scene_type`、逐源时间轴、事件/声音证据、采样策略、实际已观察覆盖、密查位置及不确定项。包含所有已检查素材的概览，不仅列最终选中的镜头。说明是否实际试听、是否只有转写、哪些转写未经核实。每个源文件记录 `duration_seconds`、`sampling_interval_seconds`（非均匀时说明）、`last_observed_pts`、`tail_gap_seconds`、`dense_review_ranges`、`crop_visibility_limitations` 与 `audio_evidence_status`；这些字段应从实际清单/观察取得，未获取用null，不填猜测数值。裁切观察过的脸部不能算成人互动已全部可见。

## 编辑分镜：storyboard.md 和 storyboard.csv

两者内容一致，按预计成片顺序排列；每行对应一个计划使用的连续源窗口，场景复杂时同一行说明内部动作阶段，不编造新的拍摄视角。

字段：`shot_id`、`scene_type`、`source_file`、`source_in`、`source_out`、`estimated_timeline_in`、`estimated_timeline_out`、`duration_seconds`、`observed_content`、`story_role`、`selection_reason`、`framing_suggestion`、`source_audio`、`music_suggestion`、`transition_suggestion`、`evidence_confidence`、`open_questions`。

- 源起止是各原视频自身时间，使用 `HH:MM:SS.mmm`；精度不足可使用整秒并说明。预计成片起止标明“预计”，不能当作原片时间。
- `observed_content` 仅写观察到的内容；构图、配乐和转场字段写建议。未听音频用“未试听”，不能填入猜测对白。未选曲可写音乐氛围建议，不能虚构曲名。
- 时长预算计入转场重叠，各分镜预计区间因此可以在过渡范围重叠；转场尚未决定时说明预算假设。
- 重要但缺证据的事件放入待核实项，不作为确定事实渲染。Markdown 中可加“故事结构、删减理由、声音安排、待复查位置”的简短说明。

## 可执行计划：edit-plan.yaml

骨架见 [模板](../templates/edit-plan.yaml)。`scene_type` 只能取支持值或逐段标明的 `mixed`；混合计划每个 segment 必须带自身支持的场景值。模板是空骨架，产出前填充实际事实，不留占位符。

每段需要：源文件、已分析候选起止、实际描述、证据及置信度、角色、七维说明评分、相似组、保留决策、推荐起止和理由。`drop` 可无推荐区间，其余尽量填写；范围未能确认时低置信并提出复查。

一致性要求：

1. `recommended_start/end` 位于该候选的 `source_start/end` 之内，也位于对应原片时长内；结束严格大于开始。
2. `recommended_duration_seconds` 与推荐区间一致，时间精度误差要说明。各 `source_file` 明确，不依赖编号猜文件。
3. `selected_order` 只列实际用于分镜/执行的非 `drop` 段，顺序与分镜一致；`optional` 不等于自动全部渲染。记录未选 optional 的理由。
4. 相似组 `selected_ids` 是 `segment_ids` 子集；只有真的语义相似才入同组。
5. 七维说明评分不求总分，禁止 `overall_score/total_score`。旧输入 `journey` 不直接当成家庭需要路线的规则。
6. story_check 使用该场景要求的项；每项建议记录 `{status, segment_ids, reason}`，状态限 `covered / partial / not_available / missing`。不得用家庭没有山景/登顶判失败。
7. `selection_summary` 的保留源秒数是 selected_order 段时长之和；预计成片秒数是该值减转场重叠，再加实际计划的独立片头/片尾等新增时长。目标未指定时可以 `null`，记录建议及假设。
8. `editor_handoff.ready_for_execution` 表示证据、区间和顺序足够执行，不代表新增授权。用户要求成片时继续执行，只要求分镜时止于计划。分别记录视觉事件、声音核实和切点的就绪情况及原因；用户授权范围另列，不因“仅分析”就把证据就绪判为false。没有试听是否影响执行取决于选段需求：依赖声音确认事件或完整对白时应补查，纯画面事件且原声原样保留时可标明限制后执行，不自动阻塞全部剪辑。全片覆盖缺口与已确认选段的切点问题分别记录。
9. 计划传给共用执行技能转换为其实际支持的项目格式；不能把未知字段直接传给 render.py。原片保留，成片实际探测后更新实际时长及检查结论。

## 单片段字段示例（结构示例，不是已观察素材）

```yaml
- id: S001
  scene_type: family-entertainment
  source_file: source.mp4
  source_start: "00:00:10.000"
  source_end: "00:00:25.000"
  description: "按实际证据填写行为"
  evidence_notes: []
  evidence_confidence: medium
  audio_evidence_status: not_auditioned
  roles: [interaction]
  scores:
    visual: 3
    event: 4
    progression: 3
    emotion: null
    story: 4
    uniqueness: 4
    quality: 3
  similarity_group: null
  decision: keep
  recommended_start: "00:00:11.000"
  recommended_end: "00:00:23.000"
  recommended_duration_seconds: 12
  trim_confidence: medium
  reason: ["根据真实输入填写保留理由"]
```

`evidence_confidence` 和 `trim_confidence` 使用 high/medium/low；情绪未确认与切点未确认分别说明，不能相互替代。`audio_evidence_status` 应区分实际试听、仅转写、仅音量线索和不可获取的情况。
