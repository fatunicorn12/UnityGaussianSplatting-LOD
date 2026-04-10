# Implementation Tasks

Each task is atomic. Mark [x] immediately when complete.
Update PROGRESS.md after every file touched.

---

## Phase 0 — Decisions

- [x] 0.1 Open questions answered by owner (2026-04-09)
- [x] 0.2 Write EXTENSION_API.md with final confirmed decisions
- [x] 0.3 Write ARCHITECTURE.md (render pipeline call chain, buffer table, exclusions)
- [x] 0.4 Write TASKS.md + PROGRESS.md

---

## Phase 1 — Modify GaussianSplatRenderer.cs

- [x] 1.1 Add backing field: `int m_ActiveSplatCount = -1;`
- [x] 1.2 Add public property: `public GraphicsBuffer gpuSortKeys => m_GpuSortKeys;`
- [x] 1.3 Add public property: `public GraphicsBuffer gpuView => m_GpuView;`
- [x] 1.4 Add public property: `public GraphicsBuffer gpuChunks => m_GpuChunks;`
- [x] 1.5 Add public property: `public bool gpuChunksValid => m_GpuChunksValid;`
- [x] 1.6 Add public property: `activeSplatCount` backed by `m_ActiveSplatCount`
- [x] 1.7 Add public property: `public bool skipInternalSort { get; set; }`
- [x] 1.8 Add event: `public event Action<CommandBuffer, Camera> BeforeSort;`
- [x] 1.9 Add event: `public event Action<CommandBuffer, Camera> AfterSort;`
- [x] 1.10 Add event: `public event Action<CommandBuffer, Camera> AfterViewData;`
- [x] 1.11 Wire BeforeSort + skipInternalSort + AfterSort into `SortPoints()`
- [x] 1.12 Wire AfterViewData into `CalcViewData()`

---

## Phase 2 — Modify GaussianSplatRenderSystem (SortAndRenderSplats)

- [x] 2.1 Read activeSplatCount, clamp, apply as instanceCount override
          (skip DebugChunkBounds mode)
- [x] 2.2 Auto-clear activeSplatCount = -1 after each DrawProcedural opportunity
- [x] 2.3 Skip DrawProcedural entirely when instanceCount == 0 after override

---

## Phase 3 — Correctness verification

- [x] 3.1 Zero-subscriber: ?.Invoke with no subscribers — null-conditional operator
          guarantees no NullReferenceException and no delegate call overhead
- [x] 3.2 Sort-skip path: BeforeSort fires inside SortPoints; SortPoints is only
          called when m_FrameCounter % m_SortNthFrame == 0 (caller, line ~121)
          — BeforeSort correctly absent on skipped frames
- [x] 3.3 Null-safe properties: backing fields are null before OnEnable / after
          OnDisable; => properties return null naturally, no throw
- [x] 3.4 activeSplatCount clamp: Mathf.Clamp(overrideCount, 0, gs.splatCount)
          — cannot exceed loaded count regardless of consumer input
- [x] 3.5 Read modified GaussianSplatRenderer.cs around each change site — verified
          above: draw block, property block, SortPoints, CalcViewData all correct

---

## Phase 4 — Documentation

- [x] 4.1 Write INTEGRATION.md (consumer quick-start for GaussianLOD.Runtime)
- [x] 4.2 Final review: re-read ARCHITECTURE.md, EXTENSION_API.md, INTEGRATION.md,
          and the modified GaussianSplatRenderer.cs for consistency

---

## Deferred (not in this PR)

- None. All originally-deferred items (AfterSort, SkipInternalSort, activeSplatCount)
  have been moved into the active task list per owner instruction 2026-04-09.
