# chris-wgpu

A personal sandbox for experimenting with **WebGPU-style graphics in Rust** using
[`wgpu`](https://wgpu.rs/).

This is a learning / experimentation project rather than a finished library or app.
It exists so I can get hands-on with the modern GPU stack in Rust: instance → adapter →
device → surface → pipeline → render pass. Expect the code to change frequently, and
expect rough edges — it is written to be read and understood, not to be stable.

## Libraries

| Crate | Version | What it does here |
| --- | --- | --- |
| [`winit`](https://docs.rs/winit) | `0.30.*` | Cross-platform window creation and the event loop. The app implements the `ApplicationHandler` trait (`resumed`, `window_event`). |
| [`wgpu`](https://docs.rs/wgpu) | `30.0.*` | The graphics abstraction layer: instances, adapters, devices, queues, surfaces, render pipelines and render passes. |
| [`pollster`](https://docs.rs/pollster) | `1.0.*` | Minimal async executor. `wgpu` setup is async, so `pollster::block_on(...)` bridges it into `winit`'s synchronous callback without pulling in a full async runtime. |
| [`log`](https://docs.rs/log) / [`env_logger`](https://docs.rs/env_logger) | `0.4.*` / `0.11.*` | Declared as dependencies for future diagnostics. Not wired up yet — no `env_logger::init()` call exists, so nothing is currently logged. |

## Requirements

- A Rust toolchain with **edition 2024** support (Rust **1.85+**; developed on 1.98).
- A GPU/driver that can run the OpenGL backend (see [Current state](#current-state)).

## Build and run

```bash
# Debug build + run (opens a window)
cargo run

# Type-check only, without producing a binary
cargo check

# Optimised build
cargo build --release
```

The `wgpu` 30.0 and `winit` 0.30 APIs are in motion, so if you are following along with
older tutorials, expect signature differences (for example `render_pass.draw(0..3, 0..1)`
instead of draw calls taking an explicit vertex buffer, and `queue.present(...)`).

## Project structure

```
.
├── Cargo.toml
├── Cargo.lock                  # committed on purpose: this is an application, not a library
└── src
    ├── main.rs                 # Entry point: creates the EventLoop, sets ControlFlow::Wait, runs the App
    ├── application.rs          # winit ApplicationHandler impl: window creation + event routing
    ├── render_state.rs         # All wgpu setup and the per-frame render() call
    └── shaders
        └── test_shader.wgsl    # WGSL vertex + fragment shader for the triangle
```

The split is deliberate: `application.rs` knows about the **event loop**, `render_state.rs`
knows about the **GPU**. Window events are translated into calls on `RenderState`
(`resize`, `render`) so the two layers stay separable.

## Current state

**A single, static triangle demo.** It is the "hello world" milestone of the project:
it proves out the whole pipeline end to end, and nothing more.

What actually happens right now:

- A window is created with `winit` and a `wgpu` surface is configured against it.
- The instance is created with the **OpenGL backend only**
  (`backends: wgpu::Backends::GL`) — this keeps initialisation predictable while
  learning, rather than the usual Vulkan/DX12/Metal defaults.
- The sRGB surface format, a `RENDER_ATTACHMENT` usage and a frame latency of 2 are
  negotiated from the surface capabilities.
- A single `RenderPipeline` is built from `test_shader.wgsl` with `TriangleList`
  topology, `Ccw` front faces and back-face culling.
- The render pass clears to a dark blue (`0.1, 0.2, 0.3`) and draws **3 vertices**
  with `render_pass.draw(0..3, 0..1)`.
- The triangle is a **red / green / blue** gradient. Its positions and colours are
  hard-coded in the shader and indexed by `@builtin(vertex_index)`, so there is **no
  vertex buffer** and no vertex layout yet.
- `RenderState::new` prints timings for instance, surface, adapter and device creation.

Design notes worth knowing:

- Redraws are **event-driven, not a continuous loop**: the event loop uses
  `ControlFlow::Wait`, and the only explicit `request_redraw()` is on resize. There is
  no animation or frame counter yet.
- `render()` returns `Result<(), &'static str>`, mapping the `CurrentSurfaceTexture`
  variants (`Timeout`, `Occluded`, `Outdated`, `Lost`, `Validation`) to string errors.
  `application.rs` matches on those and ignores the benign lifecycle skips.
- `Outdated`/`Lost`/`Suboptimal` frames trigger an immediate `resize()` reconfigure
  rather than treating the surface as fatal.

## Not done yet

The obvious next steps, roughly in order:

- [ ] Switch from `Backends::GL` back to the default backends once the GL path has served its purpose.
- [ ] Replace the hard-coded shader triangle with a real **vertex buffer** and index buffer.
- [ ] Add a continuous render loop / animation (delta time, e.g. a rotating triangle).
- [ ] Add a depth buffer and a `depth_stencil` state on the pipeline.
- [ ] Add uniforms, bind groups and a camera / MVP transform.
- [ ] Add textures and samplers.
- [ ] Handle multiple windows instead of ignoring `_window_id`.
- [ ] Fix the leftover window title (`"WGPU Clear Color"` from an earlier clear-colour step).
- [ ] Initialise `env_logger` and use `log` for real diagnostics.
- [ ] Replace `.unwrap()`/`.expect()` in setup with meaningful error handling.
- [ ] Add tests and CI.

## Notes

- No licence has been chosen yet.
- This repository is for learning, so commits are likely to be experiments rather than
  finished features.
