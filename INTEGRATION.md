# Integration Guide — GaussianLOD.Runtime

How to consume the extension API added to `GaussianSplatRenderer`.

---

## Assembly reference

Add a reference to `GaussianSplatting` (the runtime assembly defined in
`package/Runtime/GaussianSplatting.asmdef`) from your `GaussianLOD.Runtime`
assembly definition.

---

## Subscribing to events

Wire up in `OnEnable`, tear down in `OnDisable`.
**Never** cache buffer references across `OnDisable`/`OnEnable` cycles —
buffers are re-created whenever the assigned asset changes.

```csharp
using GaussianSplatting.Runtime;
using UnityEngine;
using UnityEngine.Rendering;

public class GaussianLODController : MonoBehaviour
{
    GaussianSplatRenderer m_Renderer;

    void OnEnable()
    {
        m_Renderer = GetComponent<GaussianSplatRenderer>();
        m_Renderer.BeforeSort   += OnBeforeSort;
        m_Renderer.AfterSort    += OnAfterSort;
        m_Renderer.AfterViewData += OnAfterViewData;
    }

    void OnDisable()
    {
        if (m_Renderer == null) return;
        m_Renderer.BeforeSort    -= OnBeforeSort;
        m_Renderer.AfterSort     -= OnAfterSort;
        m_Renderer.AfterViewData -= OnAfterViewData;
    }

    void OnBeforeSort(CommandBuffer cmd, Camera cam)   { }
    void OnAfterSort(CommandBuffer cmd, Camera cam)    { }
    void OnAfterViewData(CommandBuffer cmd, Camera cam){ }
}
```

---

## Pattern A — LOD fade via AfterViewData

Dispatch a compute pass after `gpuView` has been written to fade out
distant splats by reducing their opacity in-place.

```csharp
void OnAfterViewData(CommandBuffer cmd, Camera cam)
{
    if (!m_Renderer.HasValidRenderSetup) return;

    cmd.SetComputeBufferParam(m_LODCompute, m_KernelFade, "_SplatViewData", m_Renderer.gpuView);
    cmd.SetComputeBufferParam(m_LODCompute, m_KernelFade, "_OrderBuffer",   m_Renderer.gpuSortKeys);
    cmd.SetComputeFloatParam (m_LODCompute, "_FadeStart", m_FadeStartDistance);
    cmd.SetComputeFloatParam (m_LODCompute, "_FadeEnd",   m_FadeEndDistance);
    cmd.DispatchCompute(m_LODCompute, m_KernelFade, (m_Renderer.splatCount + 63) / 64, 1, 1);
}
```

---

## Pattern B — Custom sort order (replace internal sort)

Take full control of sort order. Set `skipInternalSort = true` and write
into `gpuSortKeys` inside `BeforeSort` or `AfterSort`.

`BeforeSort` fires before `CalcDistances`/`GpuSorting.Dispatch`.
`AfterSort` fires after (or immediately after `BeforeSort` when
`skipInternalSort` is true). Both receive the same `CommandBuffer`.

```csharp
void OnEnable()
{
    m_Renderer = GetComponent<GaussianSplatRenderer>();
    m_Renderer.skipInternalSort = true;   // suppress built-in radix sort
    m_Renderer.AfterSort += OnAfterSort;
}

void OnDisable()
{
    if (m_Renderer != null)
    {
        m_Renderer.skipInternalSort = false;
        m_Renderer.AfterSort -= OnAfterSort;
    }
}

void OnAfterSort(CommandBuffer cmd, Camera cam)
{
    if (!m_Renderer.HasValidRenderSetup) return;

    // Write your pre-sorted indices into gpuSortKeys.
    // The CalcViewData pass reads _OrderBuffer = gpuSortKeys immediately after.
    cmd.SetComputeBufferParam(m_LODCompute, m_KernelSort, "_SplatSortKeys", m_Renderer.gpuSortKeys);
    cmd.DispatchCompute(m_LODCompute, m_KernelSort, (m_Renderer.splatCount + 63) / 64, 1, 1);
}
```

---

## Pattern C — Draw count reduction (hard LOD cutoff)

Limit how many splats are drawn this frame. `activeSplatCount` is
auto-cleared to `-1` each frame, so set it every frame in `AfterSort`
(after the order is finalised) or in `Update`.

```csharp
void OnAfterSort(CommandBuffer cmd, Camera cam)
{
    // Draw only the nearest N splats (they are last in the sorted buffer).
    // gpuSortKeys is sorted back-to-front; CalcViewData processes indices
    // 0..instanceCount-1 in that order.
    float dist = Vector3.Distance(cam.transform.position, transform.position);
    int budget = ComputeLODBudget(dist, m_Renderer.splatCount);
    m_Renderer.activeSplatCount = budget; // 0 = skip draw entirely
}
```

Note: `activeSplatCount` is clamped to `[0, splatCount]` automatically.

---

## Pattern D — Chunk-aware custom compute

If your compute shader needs to dequantize compressed splat positions
(same as the built-in `CalcDistances` kernel), it requires the chunk
bounds buffer. Always guard on `gpuChunksValid`.

```csharp
void OnBeforeSort(CommandBuffer cmd, Camera cam)
{
    if (!m_Renderer.gpuChunksValid) return; // asset uses uncompressed storage

    cmd.SetComputeBufferParam(m_Compute, m_Kernel, "_SplatChunks", m_Renderer.gpuChunks);
    // ... rest of dispatch
}
```

---

## Null safety

All buffer properties (`gpuSortKeys`, `gpuView`, `gpuChunks`) return `null`
when the renderer is disabled or has no valid asset. Always guard:

```csharp
if (!m_Renderer.HasValidRenderSetup) return;
```

`HasValidRenderSetup` is already public on `GaussianSplatRenderer` and
checks that `m_GpuPosData`, `m_GpuOtherData`, and `m_GpuChunks` are
non-null — a sufficient guard for all extension API buffer accesses.

---

## Event timing summary

```
SortPoints() called (only when m_FrameCounter % m_SortNthFrame == 0):
  BeforeSort fires
  [if !skipInternalSort]
    CalcDistances kernel → m_GpuSortDistances
    GpuSorting.Dispatch → m_GpuSortKeys sorted
  AfterSort fires          ← gpuSortKeys is final here

CalcViewData() called (every frame):
  CalcViewData kernel → m_GpuView written
  AfterViewData fires  ← gpuView is final here

DrawProcedural:
  activeSplatCount applied and auto-cleared
  Draw skipped if instanceCount == 0
```
