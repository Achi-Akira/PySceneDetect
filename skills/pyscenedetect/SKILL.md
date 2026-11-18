---
name: pyscenedetect
description: Detect scene cuts, fades and transitions in video files with PySceneDetect, then split the video into clips, save frames from each scene, or export scene lists (CSV), HTML reports, chapters, EDL, Final Cut Pro and OpenTimelineIO timelines. Use when the user wants to find where shots change, cut a video into its scenes, make thumbnails per scene, build chapter markers, or call the scenedetect CLI or Python API.
---

# PySceneDetect

PySceneDetect finds where one shot ends and the next begins, using the `scenedetect` command or its Python library.

## Install

```bash
pip install scenedetect --upgrade
```

- Splitting video needs `ffmpeg`, or `mkvmerge` for stream copies. Check with `ffmpeg -version` first.
- On Windows, the MSI installer or portable ZIP from scenedetect.com/download includes everything.
- With Docker: `docker run --rm -v "$(pwd):/files" ghcr.io/breakthrough/pyscenedetect -i /files/video.mp4 split-video -o /files`

## How the CLI works

One input, then a chain of commands that run in order:

```bash
scenedetect -i INPUT [global options] [detect-*] [time] [output commands...]
```

Global options: `-o DIR` output folder, `-s stats.csv` per-frame metrics, `-m` minimum scene length (frames or `0.5s`), `-b` video backend.

If no detector is given, `detect-adaptive` is used.

### Choose a detector

| Command | Finds | Default `-t` | Use when |
|---|---|---|---|
| `detect-adaptive` | Fast cuts, robust to camera motion | 3.0 | Most videos (default) |
| `detect-content` | Fast cuts by colour and intensity change | 27.0 | Clean edits; lower `-t` finds more cuts |
| `detect-threshold` | Fades to or from black | 12.0 | Fade-outs and fade-ins |
| `detect-hist` | Histogram changes | 0.20 | Colour-graded footage |
| `detect-hash` | Perceptual-hash changes | 0.35 | Fast, rough pass |

Tuning: missed cuts → lower `-t`. Too many cuts (flashes, fast motion) → raise `-t`, or raise `-m` so very short scenes merge. Run once with `-s stats.csv` to see the metric values, then pick a threshold from the data.

### Limit the range

`time -s 10s -e 2:00` (start and end), or `time -s 10s -d 30s` (duration).

### Outputs

| Command | Produces |
|---|---|
| `list-scenes` | Scene table printed, plus a CSV (`-n` no file, `-s` skip the cut list) |
| `split-video` | One clip per scene (`-c` stream copy without re-encoding, `-hq` higher quality, `-crf`, `-p` preset, `-m` use mkvmerge, `-f` filename template) |
| `save-images` | Frames per scene (`-n` frames per scene, default 3; `-j` JPEG; `-f` filename template) |
| `save-html` | HTML report with thumbnails |
| `save-chapters` | Chapter file |
| `save-edl` | Edit decision list |
| `save-fcp` | Final Cut Pro XML |
| `save-otio` | OpenTimelineIO timeline |
| `save-keyframes`, `save-qp` | Keyframe or encoder QP lists |
| `load-scenes` | Reuse a saved scene CSV instead of detecting again |

Examples:

```bash
scenedetect -i video.mp4 list-scenes save-images -n 1
scenedetect -i video.mp4 detect-content -t 22 split-video -c -o clips
scenedetect -i video.mp4 detect-threshold list-scenes save-edl
scenedetect -i video.mp4 time -s 10s list-scenes
```

Run `scenedetect help <command>` for the exact options in the installed version before using an unfamiliar flag.

## Python API

```python
from scenedetect import detect, AdaptiveDetector, ContentDetector, split_video_ffmpeg

scenes = detect("video.mp4", AdaptiveDetector())   # list of (start, end) FrameTimecodes
for i, (start, end) in enumerate(scenes, 1):
    print(i, start.get_timecode(), end.get_timecode(), end.frame_num - start.frame_num)

split_video_ffmpeg("video.mp4", scenes)
```

For more control (several detectors, stats files, progress), use `open_video`, `SceneManager` and `add_detector` from `scenedetect`, then `detect_scenes(video, show_progress=True)` and `get_scene_list()`.

## Report results

Give the number of scenes, a short table of the first few with start, end and length, and where output files were written. If no cuts were found, say so: the whole video is then one scene. Suggest a lower threshold or a different detector.
