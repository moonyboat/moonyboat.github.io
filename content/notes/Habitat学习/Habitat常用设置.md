---
title: Habitat 常用设置
date: 2026-03-17
draft: false
tags:
  - Habitat
summary: "  "
---
```python
	os.environ["MAGNUM_LOG"] = "quiet"
	os.environ["HABITAT_SIM_LOG"] = "quiet"
```

```bash
export DISPLAY=:0
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json
export LD_PRELOAD=/lib/x86_64-linux-gnu/libGLX_nvidia.so.0:/lib/x86_64-linux-gnu/libEGL_nvidia.so.0
export SDL_VIDEODRIVER=x11
export SDL_VIDEO_X11_FORCE_EGL=1
export SDL_RENDER_DRIVER=software

```

```bash
sh ~/start.sh
```

```yaml
# 原版
motion_file: ""
motion_mode: seq_pose

```