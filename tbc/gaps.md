# T-BC Requirements vs Code — Gap Analysis

**Generated**: 2026-06-21
**Spec**: [tbc/spec.md](spec.md)
**Codebases analyzed**: np-linuxptp-daemon, cloud-event-proxy, np-ptp-operator

---

## Gap Summary

| # | Severity | Requirement | Gap Description | Affected Repo(s) |
| :--- | :--- | :--- | :--- | :--- |
| G1 | HIGH | FUNC-TBC015 (Process Startup Order) | Startup order is ts2phc → ptp4l → phc2sys (flat). Spec requires ptp4l[TR] → ts2phc → phc2sys (gated on PTPSourceQualified) → ptp4l[TT] | np-linuxptp-daemon |
| G2 | HIGH | FUNC-TBC015-R3/R4 (phc2sys/TT gating) | phc2sys and ptp4l[TT] start immediately — no gating on PTPSourceQualified condition | np-linuxptp-daemon |
| G3 | HIGH | FUNC-TBC016-R4/R5/R6 (processDowntimeThresholds) | CRD field exists in operator, but neither daemon nor cloud-event-proxy implements it. Process restart events fire immediately with no suppression window | all three |
| G4 | MEDIUM | FUNC-TBC010-R10 (E3 traceability) | Per O-RAN.WG6.O-Cloud Notification API Table 37, E3 (os-clock-sync-state-change) LOCKED requires OS clock synced to **"traceable & valid"** source. Current code forces E3=FREERUN when T-BC is FREERUN — this is **correct**. However, E3 does NOT enter HOLDOVER when E1=HOLDOVER — it either stays LOCKED or jumps to FREERUN. Per O-RAN, E3 should be HOLDOVER when PHC is in holdover (E1=HOLDOVER + phc2sys OK). The process-down path (G5) remains a separate issue | cloud-event-proxy |
| G5 | HIGH | FUNC-TBC016-R2 (ptp4l[TR] crash must NOT emit E3) | `processDownEvent` fires E3 on ANY process down when phc2sys is enabled — no distinction between ptp4l[TR] and phc2sys failures | cloud-event-proxy |
| G6 | HIGH | 6.2 PTPSourceQualified (separate condition) | No separate PTPSourceQualified condition exists. Current code conflates it with InSync by reusing `inSyncConditionThreshold` in a hardcoded 64-sample moving-average filter. Spec requires distinct `ptpSourceQualifiedThreshold`/`ptpSourceDisqualifiedThreshold`, sample counts, or `ptpSourceUseS3` | np-linuxptp-daemon |
| G7 | MEDIUM | FUNC-TBC013-R3 (MaxOffsetThreshold not for T-BC) | MaxOffsetThreshold is absent from the T-BC FSM (correct), but is still used as fallback for `inSyncConditionThreshold` when not explicitly configured, and in `shouldFreeRun()` for phc2sys/ts2phc log parsing | np-linuxptp-daemon |
| G8 | MEDIUM | FUNC-TBC014-R2 (holdover duration from oscillator model) | `LocalHoldoverTimeout` is actively used as a hard timer that forces Free-Run on expiry. Spec says duration should come from the oscillator model, not a user timeout | np-linuxptp-daemon |
| G9 | MEDIUM | FUNC-TBC014-R4 (8-hour holdover module) | Only the linear model ΔT(t) = S·t is implemented. The quadratic aging-dominated model ΔT(t) = T₀ + ½·A·t² for 8-hour modules is not implemented | np-linuxptp-daemon |
| G10 | MEDIUM | 6.2 HoldoverCapable | No `HoldoverCapable` concept exists in code. Source loss always enters holdover (T3) regardless of oscillator readiness. Spec says non-capable systems should go to Free-Run (T4) | np-linuxptp-daemon |
| G11 | MEDIUM | FUNC-TBC016-R1 (ptp4l[TR] crash → holdover) | Holdover entry is driven by log string parsing ("SLAVE to", "ANNOUNCE_RECEIPT_TIMEOUT"). A sudden crash (no log line) may not trigger holdover until the process restarts and re-syncs | np-linuxptp-daemon |
| G12 | MEDIUM | 12.5 SyncE events edge-triggered | SyncE events (`synce-state-change`, `synce-clock-quality-change`) are implemented but fire on every synce4l log line with no change detection. `PortState.LastQLState` is stored but never used to suppress re-publication | cloud-event-proxy |
| G13 | MEDIUM | 11.1.6 E3 threshold model | E3 uses legacy `MaxOffsetThreshold`/`MinOffsetThreshold` from `PtpClockThreshold`. The spec's `SysOffsetThreshold`/`SysOffsetInSyncSamples`/`SysOffsetOutOfSyncSamples` fields do not exist anywhere in the codebase | cloud-event-proxy |
| G14 | MEDIUM | T-TSC clock class in events | cloud-event-proxy does not enforce clock class 255 for T-TSC. E2 (clock class change) fires on any class change from daemon logs. Spec requires constant 255 and E2 suppression | cloud-event-proxy |
| G15 | LOW | T-TSC modeling in operator | No `clockType: "T-TSC"` in CRD or webhook. T-TSC is approximated as OC. Clock class 255 enforcement is in np-linuxptp-daemon (`event_tbc.go:230`) but not in operator or cloud-event-proxy | np-ptp-operator |
| G16 | LOW | Upstream port redundancy (operator) | `upstreamPort` and `leadingInterface` are allowlisted in the webhook but have no validation or reconciliation logic. Dual-upstream T-BC is only a test harness pattern | np-ptp-operator |
| G17 | LOW | SyncE operator logic | SyncE fields (`synce4lOpts`, `synce4lConf`) exist in the CRD but the operator is pure pass-through — no SyncE-specific logic | np-ptp-operator |

---

## Detailed Findings by Repository

### np-linuxptp-daemon

| Gap | File | Line(s) | Current Behavior | Required Behavior |
| :--- | :--- | :--- | :--- | :--- |
| G1 | `pkg/daemon/daemon.go` | 95-101 | `ptpProcesses` order: chronyd → ts2phc → synce4l → ptp4l → phc2sys | ptp4l[TR] first, then ts2phc, then phc2sys gated, then ptp4l[TT] |
| G2 | `pkg/daemon/daemon.go` | 755-788 | All processes start unconditionally via `cmdRun` goroutines | phc2sys and ptp4l[TT] must wait for PTPSourceQualified |
| G3 | `pkg/daemon/daemon.go` | 1360-1368 | `logProcessStatus` emits immediately, no threshold check | Suppress events within `processDowntimeThresholds.{process}` |
| G6 | `pkg/daemon/daemon.go` | 1403-1419, 1092-1100 | `checkOffsetFilterAndTransition` uses 64-sample filter + `inSyncConditionThreshold` (or `MaxOffsetThreshold` fallback) | Separate PTPSourceQualified condition with distinct thresholds (`ptpSourceQualifiedThreshold`/`ptpSourceDisqualifiedThreshold`), sample counts, or `ptpSourceUseS3` |
| G7 | `pkg/daemon/daemon.go` | 1099 | `MaxOffsetThreshold` used as fallback for `offsetThreshold` | Must not be used for T-BC; require explicit `inSyncConditionThreshold` |
| G7 | `pkg/daemon/log_parsing.go` | 187-199 | `shouldFreeRun()` uses `MaxOffsetThreshold` on phc2sys/ts2phc offsets | Should use `SysOffsetThreshold` or be removed for T-BC |
| G8 | `pkg/dpll/dpll.go` | 976-990, 1011-1017 | `LocalHoldoverTimeout` drives a hard timer; expiry → Free-Run | Duration from oscillator model, no hard timeout |
| G9 | `pkg/dpll/dpll.go` | 385-387 | Linear slope only: `slope = LocalMaxHoldoverOffSet / LocalHoldoverTimeout` | Add quadratic model for 8-hour modules |
| G10 | `pkg/event/event_tbc.go` | 146-157 | Source lost → always Holdover (T3); no HoldoverCapable check | Source lost + not capable → Free-Run (T4) |
| G11 | `pkg/daemon/daemon.go` | 1517-1528 | Holdover entry via log string match only | Should also detect process death as equivalent to source loss |

**Matching requirements (no gap):**
- T-TSC clock class 255: enforced at `event_tbc.go:230-232` via `ClockClassSlaveOnly`
- T7 (HO-In-Spec → Free-Run): exists via `freeRunCondition` in HOLDOVER state
- WPC holdover slope 104 ps/s: defaults produce correct value (1500/14400)
- MaxOffsetThreshold absent from T-BC FSM (`event_tbc.go` does not reference it)

### cloud-event-proxy

| Gap | File | Line(s) | Current Behavior | Required Behavior |
| :--- | :--- | :--- | :--- | :--- |
| G3 | `plugins/ptp_operator/config/config.go` | 70-80 | No `processDowntimeThresholds` in config struct | Implement threshold parsing and event suppression |
| G4 | `plugins/ptp_operator/metrics/logparser.go` | 504-521 | T-BC FREERUN forces E3 to FREERUN | Correct per O-RAN (FREERUN → no traceability). No code change needed for FREERUN path |
| G4 | `plugins/ptp_operator/metrics/ptp4lParse.go` | 216-277 | Holdover timer entry drives E3 to FREERUN | Per O-RAN Table 37, E3 should be **HOLDOVER** (not FREERUN) when E1=HOLDOVER and phc2sys is OK |
| G5 | `plugins/ptp_operator/metrics/metrics.go` | 355-387 | `processDownEvent` emits E3 on any process down | Must distinguish ptp4l[TR] from phc2sys; ptp4l[TR] down must NOT emit E3 |
| G12 | `plugins/ptp_operator/metrics/logparser.go` | 762-769 | SyncE events published on every log line | Add change detection before publishing |
| G13 | `plugins/ptp_operator/config/config.go` | 70-80 | Uses `MaxOffsetThreshold`/`MinOffsetThreshold` for phc2sys | Implement `SysOffsetThreshold`/`SysOffsetInSyncSamples`/`SysOffsetOutOfSyncSamples` |
| G14 | `plugins/ptp_operator/metrics/ptp4lParse.go` | 27-55 | E2 fires on any clock class change from daemon logs | Suppress E2 for T-TSC (class is always 255) |

**Matching requirements (no gap):**
- T6: E1 does NOT fire on clock-class-only changes (E2 path is separate)
- E4 worst-of aggregation: correctly implemented in `OverallState()`
- E4 not updated on E2 changes: only on E1/E3
- E2 edge detection: fires only when `ClockClass() != newClass`

### np-ptp-operator

| Gap | File | Line(s) | Current Behavior | Required Behavior |
| :--- | :--- | :--- | :--- | :--- |
| G15 | `api/v1/ptpconfig_webhook.go` | 51 | `clockTypes = ["T-GM", "T-BC"]` | Add `"T-TSC"` or document OC-as-T-TSC pattern |
| G16 | `api/v1/ptpconfig_webhook.go` | 210-215 | `upstreamPort`/`leadingInterface` allowlisted, no validation | Add validation for dual-upstream configurations |
| G17 | — | — | SyncE is pass-through only | No operator-side gap for current scope |

**Matching requirements (no gap):**
- `ProcessDowntimeThresholds` CRD: fully defined with correct defaults on `main`
- `PtpClockThreshold` legacy fields: present, schema correct
- SyncE CRD fields: present (`synce4lOpts`, `synce4lConf`)

---

## Priority Recommendations

### Must fix before next release (HIGH)

1. **G3 (processDowntimeThresholds)**: Implement threshold parsing and event suppression in both np-linuxptp-daemon and cloud-event-proxy. CRD is ready.
2. **G4/G5 (E3 independence)**: Decouple E3 from T-BC state machine in cloud-event-proxy. Remove the T-BC FREERUN → E3 FREERUN forcing. Make `processDownEvent` distinguish ptp4l[TR] from phc2sys.
3. **G1/G2 (process startup order)**: Reorder startup to match spec. Gate phc2sys and ptp4l[TT] on PTPSourceQualified.

### Should fix (MEDIUM)

4. **G6 (PTPSourceQualified)**: Define and implement as a separate condition using `ptpSourceQualifiedThreshold`, `ptpSourceDisqualifiedThreshold`, `ptpSourceQualifiedSamples`, `ptpSourceDisqualifiedSamples`, and `ptpSourceUseS3`. Separate the filter from InSync.
5. **G7 (MaxOffsetThreshold fallback)**: Require explicit `inSyncConditionThreshold` for T-BC profiles. Remove fallback to `MaxOffsetThreshold`.
6. **G8/G9 (holdover model)**: Remove `LocalHoldoverTimeout` as hard timer for T-BC. Implement quadratic model for 8-hour modules.
7. **G10 (HoldoverCapable)**: Implement the concept, even as a simple proxy (e.g., HoldoverDataValid = true → capable). Formal criteria FFS.
8. **G12 (SyncE edge detection)**: Add change guard to SyncE event publishing.
9. **G13 (SysOffsetThreshold)**: Implement dedicated phc2sys threshold fields (`SysOffsetThreshold`, `SysOffsetInSyncSamples`, `SysOffsetOutOfSyncSamples`).
10. **G14 (T-TSC E2 suppression)**: Suppress E2 in cloud-event-proxy for T-TSC profiles.

### Nice to have (LOW)

11. **G15 (T-TSC in operator)**: Add `clockType: "T-TSC"` to webhook or document OC pattern.
12. **G16 (upstream port validation)**: Add operator-side validation for dual-upstream configs.
