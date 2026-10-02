# edit-plan.yaml 输出契约

最终输出必须是合法 YAML，并尽量遵循以下结构。

```yaml
schema_version: "1.0"

project:
  type: hiking_vlog
  source_name: ""
  source_duration: ""
  target_duration_seconds: 480
  style: natural_documentary

input_summary:
  candidate_segment_count: 0
  upstream_evidence:
    keyframes: true
    transcript: false
    scene_descriptions: true
  notes: []

constraints:
  preserve_journey_feel: true
  avoid_repetitive_walking: true
  prefer_natural_sound: true
  no_total_score: true

similarity_groups:
  - id: forest_walk_01
    segment_ids: [S010, S011, S012]
    selected_ids: [S011]
    rationale: ""

segments:
  - id: S001
    source_start: "00:00:00"
    source_end: "00:00:20"
    description: ""
    evidence_notes: []
    roles: [opening]
    scores:
      visual: 3
      event: 4
      journey: 5
      emotion: 2
      story: 5
      uniqueness: 5
      quality: 4
    similarity_group: null
    decision: must_keep
    recommended_start: "00:00:02"
    recommended_end: "00:00:14"
    recommended_duration_seconds: 12
    trim_confidence: high
    reason:
      - ""

story_check:
  opening: covered
  journey_progression: covered
  environment_change: covered
  human_experience: covered
  key_reveal: covered
  climax: covered
  ending: covered
  gaps: []

selection_summary:
  must_keep_count: 0
  keep_count: 0
  optional_count: 0
  drop_count: 0
  estimated_selected_duration_seconds: 0
  target_duration_seconds: 480
  duration_warning: false
  recommended_minimum_duration_seconds: null

editor_handoff:
  ready_for_execution: true
  notes:
    - "下游剪辑器仅使用 recommended_start / recommended_end 执行裁剪。"
    - "本计划不包含转场、BGM、调色等执行命令。"
```

## 枚举约束

### decision

仅允许：

- `must_keep`
- `keep`
- `optional`
- `drop`

### story_check 状态

仅允许：

- `covered`
- `partial`
- `not_available`
- `missing`

### trim_confidence

仅允许：

- `high`
- `medium`
- `low`

## 输出一致性规则

1. `drop` 片段的 `recommended_start/end` 可以为 `null`。
2. 所有非 `drop` 片段都应尽量给出推荐剪辑区间。
3. `recommended_start/end` 必须位于 `source_start/end` 范围内。
4. `recommended_duration_seconds` 应与推荐区间一致；若无法精确计算可为 `null`。
5. `similarity_group` 引用的组必须存在于 `similarity_groups`。
6. `selected_ids` 必须是该组 `segment_ids` 的子集。
7. 不能出现总分或 `overall_score` 字段。
