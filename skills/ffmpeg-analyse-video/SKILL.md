---
name: ffmpeg-analyse-video
description: Analyse video content by extracting frames with ffmpeg and using AI vision to generate timestamped step-by-step
  summaries. Use when user provides a video file and wants to understand its visual content — screen recordings, tutorials,
  presentations, footage, or animations. Triggers on "analyse this video", "what happens in this video", "summarise this recording",
  or any request involving understanding video file contents.
---

# FFmpeg Video Analysis

Extract frames from video files with ffmpeg. Delegate frame reading to sub-agents to preserve the main context window. Synthesise a structured timestamped summary from text-only sub-agent reports.

## Architecture: Context-Efficient Sub-Agent Pipeline

**Problem**: Reading dozens of images into the main conversation context consumes most of the context window, leaving little room for synthesis and follow-up.

**Solution**: A 3-phase pipeline:

```
Main Agent                          Sub-Agents (disposable context)
──────────                          ──────────────────────────────
1. ffprobe metadata        ───►
2. ffmpeg frame extraction ───►
3. Split frames into batches ──►   4. Read images (vision)
                                      Write text descriptions
                                      to batch_N_analysis.md
5. Read text files only    ◄───    (context discarded)
6. Synthesise final output
```

Images only ever exist inside sub-agent contexts. The main agent only reads lightweight text files. This cuts context usage by ~90%.

## Scene-Aware Vlog Analysis

When a caller supplies `scene_type` or a Vlog scene profile, **use its analysis logic before choosing samples**. The generic duration table below is only the fallback for tasks without a scene profile. Share the selected profile and user focus with every frame reviewer.

- `outdoor-hiking`: read [outdoor profile](../vlog-director/references/scenes/outdoor-hiking.md). Analyse route/environment progression and whole-frame spatial reveals; densify approach, camera turn and revealed landscape, as well as meaningful human events.
- `family-entertainment`: read [family profile](../vlog-director/references/scenes/family-entertainment.md). Analyse faces, actions, dialogue and reaction/response chains even in stationary, continuous shots. Use interval coverage plus dense event review; scene cuts and codec keyframes alone are insufficient.
- User-specified events are search targets across all supplied sources, not only previously selected ranges. Record search coverage and uncertainty instead of equating sampled absence with nonexistence.
- The global 60/80/100-frame caps below must **not discard scene-required coverage**. Batch the sample set and process all batches; a per-batch/context limit is not a whole-video analysis limit. File size above2GB alone does not require a partial-range-only task or a new permission question; for a full-video request proceed in batches and report actual coverage.
- Collect local audio/transcript evidence when needed by the scene. Use available local Whisper directly for authorized analysis, preserving timestamps and marking unreliable text. ASR does not reliably describe nonverbal crying/laughter; audio peaks are only candidate cues. Never claim to have listened when only text or levels were inspected.
- For Vlogs, adapt the reviewer prompt to people, environment, action changes, reactions, interactions, technical quality and uncertainty. Visible text is optional evidence, not the primary subject. Do not require screencast classifications or code extraction. Reviewers must distinguish observation from inferred emotion/cause and report candidate ranges needing denser review.
- Output `analysis.md` with source-specific timestamps, selected scene, time sampling/observed coverage, timeline, audio evidence status, candidate event windows, confidence and limitations. Preserve these reports/manifests for downstream分镜 and revisions; do not delete them during generic cleanup unless they have been saved elsewhere or cleanup was requested.

## 1. Prerequisites

```bash
which ffmpeg && which ffprobe
```

If either is missing, show platform-specific install instructions and STOP:
- **macOS**: `brew install ffmpeg`
- **Ubuntu/Debian**: `sudo apt install ffmpeg`
- **Windows**: `choco install ffmpeg` or `winget install ffmpeg`

## 2. Setup Temp Directory

```bash
# macOS/Linux
VIDEO_ANALYSIS_DIR="/tmp/video-analysis-$(date +%s)"
mkdir -p "$VIDEO_ANALYSIS_DIR"

# Windows (PowerShell)
# $VIDEO_ANALYSIS_DIR = "$env:TEMP\video-analysis-$(Get-Date -UFormat %s)"
# New-Item -ItemType Directory -Path $VIDEO_ANALYSIS_DIR
```

## 3. Extract Video Metadata

```bash
ffprobe -v quiet -print_format json -show_format -show_streams "VIDEO_PATH"
```

Extract and report: duration, resolution (width x height), fps, codec, file size, whether audio is present.

If no video stream is found, report "audio-only file" and STOP.
For large files, use read-only extraction and batch processing. Analyse a restricted range only when requested or when an actual resource limitation requires it; file size alone is not a reason to omit the rest.

## 4. Extract Frames

For generic tasks without a selected Vlog scene, choose an initial overview strategy based on duration. Scene-aware tasks use the selected profile instead:

| Duration | Strategy | Command |
|----------|----------|---------|
| 0-60s | 1 frame every 2s | `ffmpeg -hide_banner -y -i INPUT -vf "fps=1/2,scale='min(1280,iw)':-2" -q:v 5 DIR/frame_%04d.jpg` |
| 1-10min | Scene detection (threshold 0.3) | `ffmpeg -hide_banner -y -i INPUT -vf "select='gt(scene,0.3)',scale='min(1280,iw)':-2" -vsync vfr -q:v 5 DIR/scene_%04d.jpg` |
| 10-30min | Keyframe extraction | `ffmpeg -hide_banner -y -skip_frame nokey -i INPUT -vf "scale='min(1280,iw)':-2" -vsync vfr -q:v 5 DIR/key_%04d.jpg` |
| 30min+ | Thumbnail filter | `ffmpeg -hide_banner -y -i INPUT -vf "thumbnail=SEGMENT_FRAMES,scale='min(1280,iw)':-2" -vsync vfr -q:v 5 DIR/thumb_%04d.jpg` |

For a generic coarse overview, `SEGMENT_FRAMES = total_frames / 60` gives about60 frames. This is an overview budget, not a complete event-search or scene-aware analysis budget.

**Fallbacks:**
- Scene detection yields 0 frames → retry with interval at 1 frame/5s
- Generic coarse overview exceeds100 frames → an80-frame overview can precede targeted review. Do not discard required Vlog samples or evidence of brief events; process them in batches.
- Frame extraction fails → try the next simpler strategy (scene → interval, keyframe → interval)

**Time range analysis:** When user specifies a range, prepend `-ss START -to END` before `-i`.
**Higher detail mode:** If requested, double the fps rate and lower scene threshold to 0.2.

After extraction, list all frame files with their actual source presentation timestamps. Add `showinfo` after the selection filter and preserve `pts_time` from the extraction log in a frame manifest. Scene, keyframe, and thumbnail samples are not uniformly spaced: never derive their timestamps from output sequence numbers. With range analysis, record the source offset and add it to any relative timestamps.

## 5. Delegate Frame Analysis to Sub-Agents

**This is the critical context-saving step.** Do NOT read frame images in the main conversation. Instead, split frames into batches and delegate each batch to a sub-agent.

### 5a. Prepare Batch Manifest

Split the extracted frame file list into batches of 8-10 frames each. For each batch, record:
- Batch number (1, 2, 3, ...)
- Frame file paths (absolute)
- Frame timestamps (actual source presentation timestamps from the manifest)
- Output file path: `VIDEO_ANALYSIS_DIR/batch_N_analysis.md`

### 5b. Spawn Sub-Agents

For each batch, spawn a sub-agent with the prompt below. **Launch all batches in parallel** where the tool supports it — they are fully independent.

#### Sub-Agent Prompt Template

Use the template below for generic analysis. For a scene-aware Vlog task, pass the selected scene profile and user focus, and adapt the description fields and summary to the scene as specified above. Substitute the placeholders:

```
You are analysing frames extracted from a video file.

VIDEO: {filename}
DURATION: {duration}
BATCH: {batch_number} of {total_batches}

Read each frame image listed below using the Read tool (or equivalent file reading tool that supports images). For each frame, write a structured description.

FRAMES:
{for each frame in batch}
- {absolute_path_to_frame} (timestamp: {MM:SS})
{end for}

For each frame, describe:
1. SCENE: What is visible (layout, UI elements, environment)
2. CONTENT: Text, code, labels, menus, or dialogue visible on screen
3. ACTION: What is happening or has changed since the likely previous frame
4. DETAILS: Any notable specifics (error messages, URLs, file names, button states)

After describing all frames, add a BATCH SUMMARY section with:
- Content type (one of: Screencast, Presentation, Tutorial, Footage, Animation)
- Key events in this batch's time range
- Any text/prompts/commands the user typed (quote exactly)

Write the complete analysis to: {VIDEO_ANALYSIS_DIR}/batch_{N}_analysis.md

Format the output file as:

# Batch {N} Analysis ({start_timestamp} - {end_timestamp})

## Frame-by-Frame

### Frame {sequence} ({timestamp})
- **Scene**: ...
- **Content**: ...
- **Action**: ...
- **Details**: ...

(repeat for each frame)

## Batch Summary
- **Content Type**: ...
- **Key Events**: ...
- **Quoted Text/Prompts**: ...
```

#### How to Spawn

Use whatever sub-agent, background task, or independent agent mechanism your tool provides. The requirements are simple — each sub-agent needs to:

1. **Read image files** (the frame JPEGs)
2. **Write a text file** (the batch analysis markdown)

Launch all batches in parallel if your tool supports it — they are fully independent with no shared state.

**If your tool has no sub-agent mechanism**, review directly in small batches (about20 frames per batch) and save text summaries between batches. Preserve the task’s coverage; this is a batch size rather than a whole-video limit.

### 5c. Collect Results

After all sub-agents complete, read the text analysis files. These are lightweight markdown — no images enter the main context.

```bash
ls VIDEO_ANALYSIS_DIR/batch_*_analysis.md
```

Read each `batch_N_analysis.md` file **in order**. These contain only text descriptions — the context cost is minimal compared to reading the original images.

## 6. Synthesise Output

Using only the text from the batch analysis files, perform synthesis in the main context:

1. Merge all frame descriptions into a single chronological timeline
2. Group frames into natural segments (same scene, slide, or screen)
3. Detect the dominant content type across all batches
4. Identify 3-7 key moments
5. Extract all quoted text, prompts, or commands the user typed
6. Write a 2-5 sentence narrative summary

Format the output as:

```markdown
# Video Analysis: [filename]

## Metadata
| Property | Value |
|----------|-------|
| Duration | M:SS |
| Resolution | WxH |
| FPS | N |
| Content Type | [detected] |
| Frames Analysed | N |

## Timeline
### [Segment Title] (M:SS - M:SS)
Description of what happens in this segment.

### [Segment Title] (M:SS - M:SS)
Description of what happens in this segment.

## Key Moments
1. **[M:SS] Title**: Description
2. **[M:SS] Title**: Description
3. **[M:SS] Title**: Description

## Summary
[2-5 sentence narrative paragraph summarising the entire video]
```

## 7. Cleanup

After saving the report and timestamp manifest, remove only disposable extraction files that are no longer needed. For Vlog work, preserve scene-required evidence/reports for the分镜 and revisions; do not run the generic deletion below on their directory. If cleanup is appropriate for a separate disposable directory:

```bash
# macOS/Linux
rm -rf "$VIDEO_ANALYSIS_DIR"

# Windows (PowerShell)
# Remove-Item -Recurse -Force $VIDEO_ANALYSIS_DIR
```

Skip cleanup if the user asks to keep frames.

## Advanced Options

- **Time range**: "Analyse 2:00 to 5:00 of video.mp4" → use `-ss 120 -to 300`
- **Higher detail**: "Analyse in high detail" → double frame rate, lower scene threshold to 0.2
- **Focus area**: "Focus on the code shown" → prioritise text/code extraction in sub-agent prompts
- **Sprite sheet**: For a visual overview, generate a contact sheet:
  ```bash
  ffmpeg -hide_banner -y -i INPUT -vf "select='not(mod(n,EVERY_N))',scale='min(320,iw)':-2,tile=5xROWS" -frames:v 1 DIR/sprite.jpg
  ```

## Error Handling

- ffmpeg not found → install instructions per platform, STOP
- No video stream → report audio-only, STOP
- Scene detection yields 0 frames → fallback to interval
- Large frame sets → separate coarse overview from evidence review; batch required scene/event samples rather than dropping them
- Large files → batch extraction/review; restrict ranges only when the request or an actual limitation calls for it
- Sub-agent fails or times out → read that batch's frames directly as fallback, warn about context usage
- Frame read failure in sub-agent → skip frame, note gap in batch analysis file


## Codex 运行适配

使用 `view_image` 读取抽出的帧，不能用纯文本文件读取代替视觉观察。有可用且获授权的子代理工具时，可按上述批次分析；遵守平台的并发上限，分批调度，不创建用户侧的新聊天。没有可用子代理时采用上述每批约 20 帧的直接观察回退，逐批保存文字摘要，不能把它当作整片检查上限。

抽帧结果是稀疏证据，不能证明未观察到的事件不存在，也不能把样本点当作事件精确边界。剪辑区间需要精度时，在候选区间补充密集抽帧后交给决策技能。交接报告和帧清单保存在任务输出目录；只清理由本任务创建的临时目录。
