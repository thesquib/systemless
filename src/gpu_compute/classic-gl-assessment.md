# GPU classic OpenGL and blitting: assessment

Status: banked. Nothing is implemented. Based on reading the code at `d5f042bc`
and one timing comparison, with no GL title run.

## Today

- Classic OpenGL/AGL and RAVE share a CPU rasteriser, `loader/ppc/classic_gl/raster.rs`.
- QuickDraw and QuickDraw 3D can use `gpu_compute` under `--gpu`. Each GPU result is
  read back to guest memory before the guest resumes.
- Glide reports no board and renders nothing.

## Verdict

| Work | Viable | Why |
|---|---|---|
| Classic GL and RAVE rendering | Yes, opt-in | A frame batches naturally up to swap, so one readback covers many draws |
| 2D blitting | Not as a next step | Each blit is its own round trip, and Ferazel's Wand gained nothing |

## GL design

Keep the CPU front end. `pipeline.rs` already reads guest arrays, transforms, lights
and clips, then hands `WindowVertex` values to `raster::draw_triangle`, `draw_line`
and `draw_point`. `rave.rs` calls the same three functions.

Record at that seam instead of rasterising:

1. Add a renderer trait in the style of `PpcQ3GpuRenderer`, with the backend in
   `gpu_compute` so it does not depend on a host.
2. Append window vertices to a batch, and start a new draw when raster state or a
   texture changes.
3. Upload textures from the decoded copies in `classic_gl/texture.rs`, keyed by a
   generation counter.
4. Use a wgpu render pipeline, not compute: depth, stencil and blending map onto
   fixed-function hardware.
5. Flush at swap and at anything that observes pixels, then write the existing
   `present` output to guest memory.

Flush points to cover: `glReadPixels`, `glFinish`, `glCopyPixels`, `glCopyTexImage`,
`glDrawPixels`, `glBitmap`, single-buffered `glFlush`, and context or surface changes.

Guest bytes never reach the GPU directly, so this needs no client-array byte swapping
or length discovery. Those are the main cost of forwarding calls to host GL.

Use wgpu rather than host OpenGL: the project already targets Metal, Vulkan, DX12
and WebGPU through it, and the browser has no host OpenGL.

## Risks

- Unknown gain: T&L stays on the CPU and the guest stays emulated, so the rasteriser
  must dominate frame time for this to help.
- Fixed-function coverage: WGSL must match texture environments across units, alpha
  test, fog, stipple and polygon modes. Unsupported state should flush and fall back
  to the CPU rasteriser, which needs CPU and GPU buffers kept coherent.
- Pixel differences: GPU output will not match the CPU rasteriser exactly, so it
  cannot replace it where runs are compared exactly.
- Browser: synchronous readback may constrain the WebGPU path. Unchecked.
- Test bed: no GL title has been identified. Find one before building anything.

## Order of work

1. Find a PowerPC title that uses GL or RAVE, and profile it. Stop here if the
   rasteriser is not the main cost.
2. Spike untextured, depth-tested triangles behind `--gpu`, flushing at swap. Compare
   frames and timing with the CPU rasteriser.
3. Add textures and blending, then the remaining state, with CPU fallback throughout.

## Blitting

`--gpu` already covers large colour-converting blits (`quickdraw_prefers_blit`,
16,384 pixels or more). Ferazel's Wand on a 12-core Apple Silicon machine that was
busy with other work, one headless script run twice each way:

| Run | CPU path | `--gpu` |
|---|---|---|
| 1 | 147.5 s | 145.2 s |
| 2 | 146.6 s | 151.3 s |

Screenshots from the first pair matched byte for byte. Whether Ferazel's blits
reached the kernels at all was not checked.

A real gain would need guest framebuffers that live on the GPU and copy to guest
memory only when read. Apps can read that memory at any time, so this means tracking
guest accesses across QuickDraw, which is a far larger and riskier change. Profile
2D-heavy titles first.
