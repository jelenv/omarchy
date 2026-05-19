# Fix screen recording for external monitors on NVIDIA PRIME laptops

## Summary

- Detect whether the monitor selected from Hyprland is visible to `gpu-screen-recorder`.
- If the selected monitor or region is only visible with NVIDIA PRIME offload, launch that recording with `__NV_PRIME_RENDER_OFFLOAD=1` and `__GLX_VENDOR_LIBRARY_NAME=nvidia`.
- Skip the NVIDIA PRIME probe entirely on systems without an NVIDIA GPU.
- Keep the existing default path for monitors already visible to `gpu-screen-recorder`.

## Why

On hybrid GPU laptops, Hyprland can see both the internal iGPU panel and an external dGPU monitor, while `gpu-screen-recorder` in its default environment may only see the iGPU panel.

Example from an AMD iGPU + NVIDIA dGPU laptop:

```bash
hyprctl monitors -j | jq -r '.[] | .name'
# eDP-2
# DP-2

gpu-screen-recorder --list-monitors
# eDP-2|2560x1600

__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia gpu-screen-recorder --list-monitors
# DP-2|2560x1440
```

Before this change, selecting the external monitor passed `-w DP-2` to `gpu-screen-recorder` without PRIME offload, which failed with:

```text
gsr error: display "DP-2" not found, expected one of:
  "screen"
  "eDP-2"    (2560x1600+0+0)
```

Using the portal backend is not a reliable replacement here; on the same machine it stalled during portal session setup. The KMS path works when `gpu-screen-recorder` is launched with the NVIDIA PRIME environment for the dGPU-connected monitor.

The same environment is also needed for region/window selections that sit fully inside that dGPU-connected monitor.

## Validation

```bash
bash -n bin/omarchy-capture-screenrecording

__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia \
  gpu-screen-recorder --list-monitors
# DP-2|2560x1440
```

Also verified with a short local smoke recording on the external monitor:

```bash
timeout -s INT 3s env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia \
  gpu-screen-recorder -w DP-2 -s 0x0 -k auto -f 30 -fm cfr \
  -fallback-cpu-encoding yes -o /tmp/gsr-dp2-smoke.mp4
```

And with a region on that monitor:

```bash
timeout -s INT 2s env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia \
  gpu-screen-recorder -w 640x480+-2560+0 -k auto -f 15 -fm cfr \
  -fallback-cpu-encoding yes -o /tmp/gsr-dp2-region-smoke.mp4
```
