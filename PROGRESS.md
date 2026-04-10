# Progress

## Current Step
Complete. All tasks finished.

## Completed: 21 / 21 tasks

## Last File Written
INTEGRATION.md

## Blocked On
Nothing.

## Deferred
None. All originally-deferred items were moved to active and completed.

## Files Written / Modified This Session
1. ARCHITECTURE.md — render pipeline call chain, buffer table, full API surface,
   integration notes, intentional exclusions
2. EXTENSION_API.md — final confirmed API specification with all three open
   questions answered
3. TASKS.md — 21 atomic tasks across 4 phases, all marked complete
4. PROGRESS.md — this file
5. package/Runtime/GaussianSplatRenderer.cs — code changes (see below)
6. INTEGRATION.md — consumer quick-start for GaussianLOD.Runtime

## Changes to GaussianSplatRenderer.cs

### New backing field (line ~254)
- `int m_ActiveSplatCount = -1;`

### New public API block (after splatCount property, line ~355)
- `public GraphicsBuffer gpuSortKeys => m_GpuSortKeys;`
- `public GraphicsBuffer gpuView => m_GpuView;`
- `public GraphicsBuffer gpuChunks => m_GpuChunks;`
- `public bool gpuChunksValid => m_GpuChunksValid;`
- `public bool skipInternalSort { get; set; }`
- `public event Action<CommandBuffer, Camera> BeforeSort;`
- `public event Action<CommandBuffer, Camera> AfterSort;`
- `public event Action<CommandBuffer, Camera> AfterViewData;`
- `public int activeSplatCount { get => m_ActiveSplatCount; set => m_ActiveSplatCount = value; }`

### Modified SortPoints() (line ~680)
- `BeforeSort?.Invoke(cmd, cam)` fires before CalcDistances
- Existing sort body wrapped in `if (!skipInternalSort) { ... }`
- `AfterSort?.Invoke(cmd, cam)` fires after (or immediately after BeforeSort if skipped)

### Modified CalcViewData() (line ~674)
- `AfterViewData?.Invoke(cmb, cam)` fires after DispatchCompute

### Modified GaussianSplatRenderSystem.SortAndRenderSplats() (line ~164)
- Reads gs.activeSplatCount; if >= 0 and not in DebugChunkBounds mode,
  clamps and applies as instanceCount override
- gs.activeSplatCount = -1 auto-cleared every frame
- DrawProcedural (and its profiler markers) skipped entirely when instanceCount == 0
