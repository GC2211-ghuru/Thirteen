# D3D12 Porting Baseline

This document freezes current behavior before backend refactors, so D3D12 -> Metal work can be verified against known semantics.

## Public API Contract (must remain stable)

File: `thirteen.h`

- `uint8* Init(uint32 width = 1024, uint32 height = 768, bool fullscreen = false)`
- `bool Render()`
- `void Shutdown()`
- `void SetVSync(bool enabled)`
- `bool GetVSync()`
- `void SetApplicationName(const char* name)`
- `void SetFullscreen(bool fullscreen)`
- `bool GetFullscreen()`
- `uint32 GetWidth()`
- `uint32 GetHeight()`
- `uint8* SetSize(uint32 width, uint32 height)`
- `double GetDeltaTime()`
- `void GetMousePosition(int& x, int& y)`
- `void GetMousePositionLastFrame(int& x, int& y)`
- `bool GetMouseButton(int button)`
- `bool GetMouseButtonLastFrame(int button)`
- `bool GetKey(int keyCode)`
- `bool GetKeyLastFrame(int keyCode)`

## Behavioral Baseline

- Pixel format is RGBA8 (`4` bytes per pixel), CPU-writable via returned `Pixels` pointer.
- Frame submission model is immediate:
  - user writes to `Pixels`
  - user calls `Render()`
  - `Render()` copies full buffer to current backbuffer and presents.
- `Render()` also:
  - pumps OS messages
  - updates previous-input snapshots
  - updates `GetDeltaTime()`
  - updates window title with app name + FPS text.
- `Render()` returns `false` after close/destroy message.
- VSync can be toggled at runtime through `SetVSync()` / `GetVSync()`.
- Fullscreen is borderless-window style toggle via `SetFullscreen()`.
- `SetSize()` may return a different pixel pointer and must be treated as authoritative.
- Input "last frame" accessors are frame-edge based (read current vs previous to detect presses/releases).

## Compatibility Invariants For Metal

- Keep API names, signatures, and return semantics unchanged.
- Keep `Render()` as the synchronization point visible to user code.
- Preserve full-frame upload semantics (no partial update requirement introduced).
- Preserve input query behavior and frame-edge semantics.
- Preserve fullscreen and vsync runtime toggles at API level (backend-specific implementation allowed).
- Preserve `SetSize()` contract: reallocates backing storage and can change pointer identity.

## Example-Driven Acceptance Criteria

Sources:
- `Examples/Simple/main.cpp`
- `Examples/Mandelbrot/main.cpp`
- `Examples/Minesweeper/main.cpp`

Expected behavior after backend refactor/port:

- `Simple`:
  - gradient animation updates every frame
  - `V` toggles vsync
  - `F` toggles fullscreen
  - `S` writes screenshot file.
- `Mandelbrot`:
  - left click zooms in; right click zooms out
  - `Space` resets view
  - `V` / `F` / `S` behaviors match `Simple`.
- `Minesweeper`:
  - left click reveals, right click flags
  - `Space` restarts board
  - win/lose tinting behavior preserved
  - `V` / `F` / `S` behaviors match `Simple`.

## Manual Validation Checklist

- Launch each example and verify interactive controls above.
- Verify window close exits loop (`Render()` returns `false`).
- Toggle vsync repeatedly; app remains responsive and stable.
- Toggle fullscreen repeatedly; mouse interactions remain aligned with content.
- Run for 5+ minutes; no crashes, hangs, or severe frame pacing issues.
- Resize via `SetSize()` test app path (if present) and verify returned pointer is used.

