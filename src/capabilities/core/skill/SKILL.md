---
name: qwen-mm-plugins-core
description: >-
  Read and visualize any file — images, video, documents, code, data, 3D models,
  and more — with local MCP tools, and run visual/audio understanding on local
  Unsloth (no cloud APIs). Use when the agent needs to inspect file contents or
  media metadata, crop an image, draw bounding boxes, save document pages and
  video frames, or ask a local VL/LLM about media without uploading.
---

# Local File Inspection

You have `qwen-mm-plugins-core` MCP tools available. Use them to read and visualize supported local files, inspect media metadata, crop/annotate images, and save document pages or video frames. Prefer these MCP tools over manual scripting.

In the default native mode, these tools return text and images for the agent to inspect without a model API call. Keep `QWEN_MM_NATIVE_MODE=1`. Do **not** set `QWEN_MM_NATIVE_MODE=0` — that caption fallback sends image results to the configured VL endpoint.

Check the `qwen-mm-plugins-core` tools in your tool list for full schemas and parameters.

## Local Unsloth only (no cloud APIs)

Do not call cloud model, search, storage, or upload APIs for media understanding. That includes DashScope, OpenRouter, OrcaRouter, Cheaper Inference, MiniMax, Serper/Tavily/Exa/Serply, hosted `qwen-mm-plugins-api` (`vision_chat`, `ocr`, `grounding`, Omni, DashScope `transcribe_audio`), and any remote endpoint that would receive local files or conversation content.

For visual questions about images/frames, call the local Unsloth OpenAI-compatible server (do not send pixels to a hosted VL API):

```bash
unsloth run \
  --model unsloth/Qwen3-VL-2B-Instruct-GGUF:UD-Q2_K_XL \
  --disable-tools \
  --reasoning off \
  --api-only \
  -H 127.0.0.1 \
  -p 8888
```

- Client: `http://127.0.0.1:8888/v1` (`/v1/chat/completions`)
- Model id: `unsloth/Qwen3-VL-2B-Instruct-GGUF`
- Quant file: `Qwen3-VL-2B-Instruct-UD-Q2_K_XL.gguf`
- Auth: Unsloth local `sk-unsloth-…` key stored as `DASHSCOPE_API_KEY` in `~/.qwen-mm-plugins/config` (never a cloud key). `DASHSCOPE_BASE_URL` should be `http://127.0.0.1:8888/v1`.
- Bind **127.0.0.1** only. `-H 0.0.0.0` can expose the port on the public internet. `--api-only` keeps the Studio UI from unloading this GGUF. Keep `--disable-tools` so the server cannot web-search.

For audio/text tasks on the small local model (port **8889**, does not collide with vision):

```bash
unsloth run \
  --model unsloth/Qwen3-0.6B-GGUF:UD-Q4_K_XL \
  --disable-tools \
  --reasoning off \
  -H 127.0.0.1 \
  -p 8889
```

Client: `http://127.0.0.1:8889/v1`. Do not use DashScope ASR. If a server is down, start the matching `unsloth run` command locally; do not fail over to a hosted API.

## When to Use Which Tool

Reading and inspection:
- **Inspect metadata FIRST** for any video/audio (duration, resolution, fps, codecs, bitrate, audio/video/subtitle tracks, rotation, chapters) → `media_info`. Run it before `read_video` and before any clip/edit — see *Metadata first* below.
- **See a file** (PDF, Office, CSV, code, notebook, 3D, ...) → `visualize`
- **Read an image** with dynamic resolution → `read_image`
- **Read a video** (extract frames) → `read_video`
- **Save specific frame(s)** of a video to file → `save_view` (pass `times=[...]`)
- **Save document page(s)** as images → `save_view` (pass `pages="..."`)

Producing / annotating (writes an image file):
- **Crop a rectangular region** from an image → `crop`
- **Photo coordinates**: `read_image`, `crop`, and `draw_bbox` apply EXIF orientation. Use 0–1000 coordinates in the displayed image, including boxes returned by grounding tools when available locally.
- **Draw bounding boxes** on an image → `draw_bbox`

## Visualize — Supported Formats

| Category | Extensions | Notes |
|----------|-----------|-------|
| Documents | `.pdf`, `.svg` | Built-in (pypdfium2 + resvg) |
| Office | `.docx`, `.pptx`, `.vsdx` | Needs `libreoffice` |
| Data | `.csv`, `.xlsx` | Text table + chart image |
| Code | `.js`, `.ts`, `.py`, `.go`, `.rs`, `.md`, ... | Returns text (markdown code block) |
| Plain text | `.txt`, `.text`, `.log` | Returns text (fenced block) |
| Web pages | `.html`, `.htm`, `.mhtml` | Screenshot; needs `playwright` |
| Diagrams | `.drawio` | XML → SVG rendering |
| Subtitles | `.srt`, `.vtt` | Returns text |
| 3D Models | `.obj`, `.stl`, `.glb`, `.gltf`, `.fbx`, `.ply`, `.step`, `.stp` | Built-in; `blender` for best quality |
| GIS/Geo | `.geojson`, `.kml`, `.shp` | Built-in |
| Notebooks | `.ipynb` | Text cells + embedded images |
| LaTeX | `.tex` | Compiles to PDF; falls back to source on failure |
| Images/Videos | `.jpg`, `.mp4`, ... | Delegates to `read_image`/`read_video` |

The table lists supported formats; files with unknown extensions return an unsupported-type error.

Use `pages` for page ranges, `budget` for resolution, `max_pages` to cap output.

## Metadata First

Always run `media_info` on a video/audio file **before** `read_video` and before any clip/trim/concat/mux — it reads only the header, so it is fast even on huge files, and it tells you how to treat the source. `read_video` also samples smarter once you know the fps and duration.

When juggling heterogeneous assets (action-cam clips, VFX/stock footage, voiceover, SFX), `media_info` is how you catch the traps that silently corrupt an edit — check for:
- **VFR** (variable frame rate — common on phones/action cams): `media_info` flags it as `possibly VFR`. Frame-accurate cuts need a constant-fps conform first (a video-edit step) — never assume a fixed fps on VFR sources.
- **Rotation** (`rotation 90°/180°`): the display orientation differs from stored pixels; bake it in before cropping or overlaying.
- **Mismatched fps / resolution / DAR** across clips → pick one project timebase and conform the outliers.
- **Audio sample rate & channels** (voiceover vs music vs SFX often differ, e.g. 44.1k vs 48k, mono vs stereo) → resample to one project rate before mixing.
- **No audio track** / **odd container start_time** → `media_info` calls these out; both shift or break naive timestamp math.

## Tips

**Resolution budgets**: `small` for preview, `normal` (~1024) default, `large` for fine detail.

**Video strategy**: `fps=0` auto-selects. Skim with `fps=1, budget="normal"` in 5-min chunks, then detail with `fps=2, budget="large"` on interesting segments. Use `start_time`/`end_time` for windowing.

**Model questions about pixels**: use Core to materialize frames (`read_image` / `save_view` / `visualize`), then ask the local Unsloth VL endpoint above. Prefer summarizing answers in text rather than re-uploading large images into the chat when the host agent is cloud-backed.

## Relationship to Other Capabilities

Core handles file reading, rendering, and basic image operations. Under the local-only policy, do **not** install or call hosted `qwen-mm-plugins-api` / `qwen-mm-plugins-search` for media that must stay on-device. Use local Unsloth for VL/audio understanding instead. Annotate boxes with `draw_bbox` here after any local grounding step.
