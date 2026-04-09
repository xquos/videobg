# Changelog — videobg (mpvpaper fork)

All notable changes to videobg (xquos/videobg branch `fix/improvements`).

## [Unreleased] — NVIDIA EGL Memory Leak: What We Tried

### What's Working
- **Self-restart workaround** at 800MB RSS — process restarts transparently, video position preserved
- **egl-wayland2** (dmabuf platform) — ~30% leak reduction, installed via `pacman -S egl-wayland2`
- **mpv_render_context_report_swap()** — tells libmpv when a frame was displayed
- **glFlush()** before swap + after mpv_render_context_update() — drains GL pipeline
- **Event queue draining** — drains all MPV events per tick to prevent backlog
- **swapchain-depth=1** + **opengl-pbo=no** — minimize VRAM/buffer accumulation
- Frame timing: A-V: 0.000, no drops, 24fps

### What Didn't Work
| Attempt | Result | Why |
|---------|--------|-----|
| gSlapper (GStreamer path) | Worse | Same EGL path, 2x baseline memory |
| FBO throttle (reduce swap to 6fps) | Minimal | Per-render leak component exists independently |
| EGL surface recycle (destroy + recreate) | CRASHES | Compositor holds buffer refs — can't safely tear down |
| Lower fps alone | No change | Leak rate is per-render, not per-fps |
| hwdec=no | No change | Not hardware decode buffers |
| VSync alone | No change | Not a sync timing issue |

### Current Leak Rate
| Config | Leak Rate | Restart Interval |
|--------|-----------|-----------------|
| Stock mpvpaper | ~14MB/min | ~30min |
| + egl-wayland2 | ~10MB/min | ~45min |
| **+ report_swap + glFlush + Daniel fixes** | **~7MB/min** | **~60+ min** |

### Still Needs Fixing
The ~7MB/min leak is inside NVIDIA's proprietary EGL driver (`libnvidia-eglcore.so`).
Root cause confirmed by Valgrind: zero leaks in our C code — all errors from NVIDIA blob.

### Real Fix Options (not yet built)
1. **wl_shm wallpaper daemon** — bypasses EGL entirely, zero leak, ~150MB steady, ~3-5 day build
2. **Report to NVIDIA** — file bug at https://github.com/NVIDIA/egl-wayland
3. **Accept + wait** — current workaround is functional

---

## [1.8.1] — 2026-04-09

### Added
- `src/main.c`: videobg fork from mpvpaper with all improvements

### Key Fork Changes
- FBO render + blit path (working, video visible, correct orientation)
- EGL swap order fix (callback before swap, not after — eliminates frame drops)
- glFlush() calls around EGL operations
- Memory-capped demuxer settings (50MiB/25MiB)
- 24fps poll timeout (halves GPU usage vs 60fps)
- Event thread at 10Hz (was 100Hz)
- Self-restart at 800MB RSS threshold
- Video position save/restore via -Z flag
- Support for per-output process spawning

### Removed
- Broken FBO throttle attempt (didn't materially reduce leak)
