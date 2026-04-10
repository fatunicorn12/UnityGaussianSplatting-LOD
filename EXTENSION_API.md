# Extension API — Final Specification

## Summary

All public API lives on `GaussianSplatRenderer` (already `public class`).
`GaussianSplatRenderSystem` stays `internal`. No new classes.

---

## Correction: m_GpuIndexBuffer is not the order buffer

`m_GpuIndexBuffer` is a static 36-element `ushort` geometric index buffer
for cube face triangles. It is written once at asset load and passed as the
first argument to `DrawProcedural` to define quad/box geometry. It has no
per-splat ordering data and is not part of the extension API.

The per-splat order buffer is `m_GpuSortKeys`.

---

## Read-only buffer properties

```csharp
public GraphicsBuffer gpuSortKeys  => m_GpuSortKeys;
public GraphicsBuffer gpuView      => m_GpuView;
public GraphicsBuffer gpuChunks    => m_GpuChunks;
public bool           gpuChunksValid => m_GpuChunksValid;
```

- All return `null` (or `false`) before `OnEnable` and after `OnDisable`.
- `gpuChunks` may be a 1-element dummy buffer when `gpuChunksValid` is false.
  Always check `gpuChunksValid` before dispatching compute that reads chunk data.
- Do not cache references across `OnDisable`/`OnEnable` cycles — buffers are
  re-created when the asset changes.

---

## Sort control

### `skipInternalSort`

```csharp
public bool skipInternalSort { get; set; }
```

- Default: `false`
- Not auto-cleared. Consumer is responsible for managing this flag.
- When `true`: `BeforeSort` fires, then `SortPoints` returns immediately
  without running `CalcDistances` or `GpuSorting.Dispatch`.
  `AfterSort` still fires regardless.

### Events

```csharp
public event Action<CommandBuffer, Camera> BeforeSort;
public event Action<CommandBuffer, Camera> AfterSort;
public event Action<CommandBuffer, Camera> AfterViewData;
```

#### `BeforeSort`
- Fires at the start of `SortPoints()`, before `CalcDistances` and before
  the GPU radix sort.
- Not fired on frames where sort is skipped by `m_SortNthFrame` (because
  `SortPoints` is not called at all on those frames).

#### `AfterSort`
- Fires at the end of `SortPoints()`, after `GpuSorting.Dispatch` completes
  (or immediately after `BeforeSort` if `skipInternalSort` is true).
- `m_GpuSortKeys` contains the final sorted order at this point.
- Fires regardless of `skipInternalSort` value.

#### `AfterViewData`
- Fires at the end of `CalcViewData()`, after the `CalcViewData` compute
  kernel has written `m_GpuView`.
- Always fires when `CalcViewData` runs (every frame, not gated by
  `m_SortNthFrame`).

---

## Draw count control

```csharp
public int activeSplatCount { get; set; }
```

- Default: `-1` (use full `splatCount`).
- When `>= 0`: overrides `instanceCount` in `DrawProcedural` for the
  normal Splats render mode. Clamped to `[0, splatCount]`.
- When `0`: `DrawProcedural` is skipped entirely for this renderer this frame.
- **Auto-cleared to `-1` after each `DrawProcedural` opportunity** (whether
  the draw happened or was skipped due to count = 0).
- Not applied in `DebugChunkBounds` mode (that mode uses chunk count).

---

## Invocation sites in source

| Event / property | Location |
|---|---|
| `BeforeSort` fires | `SortPoints()` — first line after Preview guard |
| `skipInternalSort` checked | `SortPoints()` — after `BeforeSort` fires |
| `AfterSort` fires | `SortPoints()` — final line (inside and outside skip branch) |
| `AfterViewData` fires | `CalcViewData()` — after `DispatchCompute` |
| `activeSplatCount` read + cleared | `GaussianSplatRenderSystem.SortAndRenderSplats()` — before `DrawProcedural` |

---

## What was NOT exposed and why

| Item | Reason |
|---|---|
| `GaussianSplatRenderSystem` | Stays `internal`. All API is on `GaussianSplatRenderer`. |
| `m_GpuSortDistances` | Internal sort key buffer, encoding is an implementation detail. |
| `m_GpuPosData`, `m_GpuOtherData`, `m_GpuSHData`, `m_GpuColorData` | Raw compressed asset data; format-version-dependent. |
| All `m_GpuEdit*` buffers | Editor-only, lazily allocated, not relevant to runtime LOD. |
| `m_GpuIndexBuffer` | Static cube geometry index buffer; no per-splat data. |
| `GaussianSplatURPFeature` | Internal URP integration. Events fire from `GaussianSplatRenderer` directly. |
