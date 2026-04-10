# Architecture: GaussianSplatting Extension API

## Confirmed Render Pipeline Call Chain

### Built-in Render Pipeline (BiRP)

```
Camera.onPreCull
  └─ GaussianSplatRenderSystem.OnPreCullCamera(cam)
       ├─ GatherSplatsForCamera(cam)
       │    └─ collects active GaussianSplatRenderer instances, sorts by m_RenderOrder then depth
       ├─ InitialClearCmdBuffer(cam)
       │    └─ attaches m_CommandBuffer to CameraEvent.BeforeForwardAlpha (once per camera)
       ├─ m_CommandBuffer: GetTemporaryRT(_GaussianSplatRT), SetRenderTarget, ClearRenderTarget
       ├─ SortAndRenderSplats(cam, m_CommandBuffer)    ← see shared section below
       ├─ Compose: DrawProcedural(Matrix4x4.identity, matComposite, ...)
       └─ ReleaseTemporaryRT(_GaussianSplatRT)
```

### URP

```
GaussianSplatURPFeature.OnCameraPreCull(renderer, cameraData)
  └─ GaussianSplatRenderSystem.instance.GatherSplatsForCamera(cam)

GaussianSplatURPFeature.AddRenderPasses
  └─ enqueues GSRenderPass (RenderPassEvent.BeforeRenderingTransparents)

GSRenderPass.RecordRenderGraph → SetRenderFunc (UnsafePass):
  ├─ SetRenderTarget(GaussianSplatRT, SourceDepth, ClearFlag.Color)
  ├─ GaussianSplatRenderSystem.instance.SortAndRenderSplats(cam, commandBuffer)
  └─ Blitter.BlitCameraTexture(GaussianSplatRT → SourceTexture, matComposite, pass 0)
```

### HDRP

```
GaussianSplatHDRPPass.Execute(ctx)
  ├─ GaussianSplatRenderSystem.instance.GatherSplatsForCamera(cam)
  ├─ CoreUtils.SetRenderTarget(ctx.cmd, m_RenderTarget, ctx.cameraDepthBuffer, ClearFlag.Color)
  ├─ GaussianSplatRenderSystem.instance.SortAndRenderSplats(cam, ctx.cmd)
  └─ CoreUtils.DrawFullScreen(ctx.cmd, matComposite, ...)
```

### Shared per-splat loop — `SortAndRenderSplats(cam, cmb)` (GaussianSplatRenderer.cs:108)

Called identically by all three pipeline paths. Iterates `m_ActiveSplats` in sorted order:

```
for each (GaussianSplatRenderer gs, MaterialPropertyBlock mpb) in m_ActiveSplats:

  gs.EnsureMaterials()

  [if gs.m_FrameCounter % gs.m_SortNthFrame == 0]
    gs.SortPoints(cmb, cam, matrix)                          ← :612
      ├─ SetComputeBufferParam: SplatSortDistances → m_GpuSortDistances
      ├─ SetComputeBufferParam: SplatSortKeys      → m_GpuSortKeys
      ├─ SetComputeBufferParam: SplatChunks        → m_GpuChunks
      ├─ SetComputeBufferParam: SplatPos           → m_GpuPosData
      ├─ DispatchCompute: KernelIndices.CalcDistances
      │    writes m_GpuSortDistances (view-space depth per splat)
      │    reads  m_GpuSortKeys as initial index identity
      └─ GpuSorting.Dispatch(cmd, m_SorterArgs)
           sorts m_GpuSortDistances (keys) + m_GpuSortKeys (payload) together
           result: m_GpuSortKeys[0] = furthest splat index, [N-1] = nearest

  ++gs.m_FrameCounter

  gs.CalcViewData(cmb, cam)                                  ← :579
      ├─ SetAssetDataOnCS → binds all data buffers including m_GpuSortKeys as _OrderBuffer
      ├─ DispatchCompute: KernelIndices.CalcViewData
      │    reads  m_GpuSortKeys (_OrderBuffer), m_GpuPosData, m_GpuOtherData,
      │           m_GpuSHData, m_GpuColorData, m_GpuChunks
      └─   writes m_GpuView (40 bytes/splat: 2D covariance, projected pos, opacity)

  mpb.SetBuffer(OrderBuffer,    m_GpuSortKeys)               ← :142
  mpb.SetBuffer(SplatViewData,  m_GpuView)                   ← :140
  mpb.SetBuffer(SplatChunks,    m_GpuChunks)                 ← :139
  [+ remaining asset data buffers]

  cmb.DrawProcedural(gs.m_GpuIndexBuffer, matrix, displayMat, 0, topology, indexCount, instanceCount, mpb)
      ← m_GpuIndexBuffer is a static 36-element ushort cube geometry index buffer
        created once at asset load, never modified
```

---

## Buffer Roles and Access Modifiers

| Field | Type | Current Access | Role | API decision |
|---|---|---|---|---|
| `m_GpuSortKeys` | `GraphicsBuffer` (Structured, uint×N) | `internal` | Per-splat draw order — payload of GPU radix sort. Initialized 0..N-1, sorted back-to-front each frame. Bound as `_OrderBuffer` in both compute (CalcViewData) and draw shader. **This is the LOD injection point.** | Expose as `public GraphicsBuffer gpuSortKeys` (read-only property) |
| `m_GpuSortDistances` | `GraphicsBuffer` (Structured, float×N) | `private` | Sort keys — view-space depth per splat. Written by CalcDistances kernel, consumed only by GpuSorting. | Stay private. Not needed by consumers. |
| `m_GpuView` | `GraphicsBuffer` (Structured, 40 bytes×N) | `internal` | Per-splat projected data: 2D screen covariance, projected position, opacity. Written by CalcViewData, read by draw shader. | Expose as `public GraphicsBuffer gpuView` (read-only property) |
| `m_GpuChunks` | `GraphicsBuffer` (Structured, ChunkInfo×C) | `internal` | Compressed chunk bounds for dequantization. Read by CalcDistances and CalcViewData. C = ceil(N/256). | Expose as `public GraphicsBuffer gpuChunks` (read-only property) — **pending final decision, see Deferred** |
| `m_GpuPosData` | `GraphicsBuffer` (Raw) | `private` | Raw compressed position data. | Stay private. |
| `m_GpuOtherData` | `GraphicsBuffer` (Raw) | `private` | Raw compressed scale/rotation data. | Stay private. |
| `m_GpuSHData` | `GraphicsBuffer` (Raw) | `private` | Raw compressed spherical harmonics. | Stay private. |
| `m_GpuColorData` | `Texture` | `private` | Base color texture (various compressed formats). | Stay private. |
| `m_GpuIndexBuffer` | `GraphicsBuffer` (Index, ushort×36) | `internal` | Static cube geometry index buffer. **Not a per-splat order buffer.** Never sorted, never updated. Used only as first arg to DrawProcedural. | No change. Not part of the extension API. |
| `m_GpuEditCutouts` | `GraphicsBuffer` | `private` | Lazily-created cutout shape data. Editor-only. | Stay private. |
| `m_GpuEditSelected` | `GraphicsBuffer` | `private` | Bitfield of selected splats. Editor-only. | Stay private. |
| `m_GpuEditDeleted` | `GraphicsBuffer` | `private` | Bitfield of deleted splats. Editor-only. | Stay private. |
| `m_GpuEditCountsBounds` | `GraphicsBuffer` | `private` | Selection statistics (counts + AABB). Editor-only. | Stay private. |
| `m_GpuEditSelectedMouseDown` | `GraphicsBuffer` | `private` | Selection snapshot at drag start. Editor-only. | Stay private. |
| `m_GpuEditPosMouseDown` | `GraphicsBuffer` | `private` | Position snapshot at drag start. Editor-only. | Stay private. |
| `m_GpuEditOtherMouseDown` | `GraphicsBuffer` | `private` | Scale/rot snapshot at drag start. Editor-only. | Stay private. |

---

## Complete Final API Surface (on `GaussianSplatRenderer`)

All additions are on the existing `public class GaussianSplatRenderer : MonoBehaviour`. No new classes. No changes to `GaussianSplatRenderSystem`.

### Read-only buffer properties

```csharp
/// <summary>
/// Per-splat draw order buffer. Contains N uint indices sorted back-to-front by
/// view-space depth. Index 0 is the furthest splat; index N-1 is the nearest.
/// Bound as _OrderBuffer in both the CalcViewData compute shader and the draw shader.
/// Valid only while the renderer is enabled and has a valid asset.
/// </summary>
public GraphicsBuffer gpuSortKeys => m_GpuSortKeys;

/// <summary>
/// Per-splat view data buffer. 40 bytes per splat: projected screen position,
/// 2D covariance, and opacity. Written by CalcViewData each frame.
/// Valid only while the renderer is enabled and has a valid asset.
/// </summary>
public GraphicsBuffer gpuView => m_GpuView;

/// <summary>
/// Chunk bounds buffer for dequantizing compressed splat data.
/// Contains one ChunkInfo per 256 splats. May be a dummy 1-element buffer
/// when asset data is stored without chunking (m_GpuChunksValid == false).
/// </summary>
public GraphicsBuffer gpuChunks => m_GpuChunks;   // pending final decision
```

### Events

```csharp
/// <summary>
/// Fired at the start of SortPoints(), before CalcDistances and before the GPU
/// radix sort runs. m_GpuSortDistances and m_GpuSortKeys contain values from
/// the previous sort frame.
/// Not fired on frames where sort is skipped (m_FrameCounter % m_SortNthFrame != 0).
/// Subscribers may enqueue compute work on the provided CommandBuffer.
/// </summary>
public event Action<CommandBuffer, Camera> BeforeSort;

/// <summary>
/// Fired at the end of CalcViewData(), after the CalcViewData compute kernel
/// has written m_GpuView. m_GpuSortKeys contains the current sorted order.
/// Subscribers may enqueue compute work on the provided CommandBuffer.
/// </summary>
public event Action<CommandBuffer, Camera> AfterViewData;
```

### Invocation sites

- `BeforeSort` fires inside `SortPoints()` at line ~613, before the CalcDistances dispatch.
- `AfterViewData` fires inside `CalcViewData()` at line ~609, after the CalcViewData dispatch.
- Both methods are `internal` but called by `GaussianSplatRenderSystem.SortAndRenderSplats()`.
- No changes needed in `GaussianSplatURPFeature`, `GaussianSplatHDRPPass`, or `GaussianSplatRenderSystem`.

---

## Integration Notes for GaussianLOD.Runtime

GaussianLOD.Runtime is an external assembly that will reference the GaussianSplatting runtime package. It should consume the API as follows:

```csharp
// Attach to a GaussianSplatRenderer component and wire up LOD logic
var renderer = GetComponent<GaussianSplatRenderer>();
renderer.BeforeSort += OnBeforeSort;
renderer.AfterViewData += OnAfterViewData;

void OnBeforeSort(CommandBuffer cmd, Camera cam)
{
    // Option A: dispatch a compute pass that writes into renderer.gpuSortKeys
    // to mask out distant splats before the sort runs.
    // Note: the internal GPU radix sort will still run after this and will
    // re-sort whatever is in gpuSortDistances / gpuSortKeys.
    // Use AfterSort (if added) to fully replace sort output.
}

void OnAfterViewData(CommandBuffer cmd, Camera cam)
{
    // Option B: dispatch a compute pass that reads renderer.gpuView and/or
    // renderer.gpuSortKeys to post-process projected splat data,
    // e.g. fade out splats in LOD transition bands.
}
```

**Key consumer constraints:**
- Consumers must not cache the buffer references across `OnDisable`/`OnEnable` cycles — buffers are re-created when the asset changes.
- `gpuSortKeys` and `gpuView` are null when `HasValidRenderSetup` is false.
- `gpuChunks` may be a 1-element dummy buffer when `m_GpuChunksValid` is false; consumers must check `m_GpuChunksValid` — **expose `public bool gpuChunksValid => m_GpuChunksValid`** to allow this check without reflection.

---

## What Was Intentionally NOT Exposed and Why

| Item | Reason not exposed |
|---|---|
| `GaussianSplatRenderSystem` access modifier | Stays `internal` per project constraint. All pipeline coordination is an implementation detail; consumers have no reason to touch it. |
| `m_GpuSortDistances` | Purely internal to the sort pass. Exposing it would invite consumers to write distances directly and rely on internal sort key encoding, creating fragile coupling. |
| `m_GpuPosData`, `m_GpuOtherData`, `m_GpuSHData`, `m_GpuColorData` | Raw compressed asset data. Format changes with asset version. Consumers doing custom compute on this data should use the existing `SetAssetDataOnMaterial` / `SetAssetDataOnCS` paths. |
| All `m_GpuEdit*` buffers | Editor-only, lazily created, lifecycle managed by editor tools. Not relevant to runtime LOD. |
| `m_GpuIndexBuffer` | Static cube geometry, not a per-splat data buffer. Exposing it would cause confusion (it was initially mistaken for the order buffer). |
| `GaussianSplatURPFeature` | Internal URP integration detail. Events fire from `GaussianSplatRenderer` directly, so URP-specific code does not need to be involved. |
| A writable `activeSplatCount` | Deferred. Requires changing the `instanceCount` argument in `DrawProcedural`, which is inside `GaussianSplatRenderSystem`. Changing instance count without changing access modifiers of the system is non-trivial. Punted to a future PR. |
