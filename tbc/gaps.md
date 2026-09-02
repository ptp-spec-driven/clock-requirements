# T-BC Requirements vs Code — Gap Analysis

**Generated**: 2026-09-02 (revisited from 2026-06-21)
**Spec**: [tbc/spec.md](spec.md)
**Codebases analyzed**: np-linuxptp-daemon, cloud-event-proxy, np-ptp-operator

**Branches analyzed (working tree):**
- np-linuxptp-daemon: `fix/phc2sys-metrics-cleanup` (clean)
- cloud-event-proxy: `014-deprecate-min-offset-threshold` (clean)
- np-ptp-operator: `018-sysoffset-threshold-e3` (clean; 2 commits ahead of `main`)

> Note: some fixes exist only on feature branches and are not yet on `main`. Where a status depends on a non-`main` branch it is called out explicitly. The T-BC FSM moved from `pkg/event/event_tbc.go` to `pkg/clock/tbc.go` since the previous revision; stale file references have been updated.

---

## Gap Summary

Status legend: **FIXED** (implemented, no remaining gap) · **PARTIAL** (partially implemented / landed on a branch but not complete) · **OPEN** (still a gap).

| # | Severity | Status | Requirement | Gap Description | Affected Repo(s) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ~~G1~~ | ~~HIGH~~ | ~~PARTIAL~~ | ~~FUNC-TBC015 (Process Startup Order)~~ | ts2phc startup is now gated on offset qualification, but the process registry order is still flat (`chronyd → ts2phc → synce4l → ptp4l → phc2sys`) and ptp4l[TT] still starts immediately. Spec requires explicit ptp4l[TR] → phc2sys (gated) → ts2phc (gated) → ptp4l[TT] (gated) <br>**Will be addressed by<br>https://redhat.atlassian.net/browse/CNF-26381**| np-linuxptp-daemon |
| ~~G2~~ | ~~HIGH~~ | ~~PARTIAL~~ | ~~FUNC-TBC015-R3/R4 (phc2sys/TT gating)~~ | phc2sys is coarsely gated on `abs(offset) < 1e9` ns (not PTPSourceQualified); ptp4l[TT] is NOT gated at all — starts immediately<br>**For ptp4l(TT) start will be addressed by https://redhat.atlassian.net/browse/CNF-26381.<br>for phc2sys start will be addressed by G4 of [T-GM gaps](../tgm/gaps.md)**| np-linuxptp-daemon |
| ~~G3~~ | ~~HIGH~~ | ~~PARTIAL~~ | ~~FUNC-TBC016-R4/R5/R6 (processDowntimeThresholds)~~ | `ProcessDowntimeThresholds` CRD fully defined with correct defaults in the operator. Neither np-linuxptp-daemon nor cloud-event-proxy implements parsing/suppression — process restart events fire immediately<br>**will be addressed by <br>https://redhat.atlassian.net/browse/CNF-26381** | all three |
| ~~G4~~ | ~~MEDIUM~~ | ~~FIXED~~ | ~~FUNC-TBC010-R10 (E3 traceability)~~ | E3 is now derived as worst-of(phc2sys, E1). E1=HOLDOVER + phc2sys OK → E3=HOLDOVER (was FREERUN). Traceable-source constraints enforced via phc2sys selection-window machine in `os_clock_discipline.go` | cloud-event-proxy |
| ~~G5~~ | ~~HIGH~~ | ~~OPEN~~ | ~~FUNC-TBC016-R2 (ptp4l[TR] crash must NOT emit E3)~~ | `processDownEvent` still emits E3 (os-clock-sync-state-change) on ANY process down when phc2sys enabled — no distinction between ptp4l[TR] and phc2sys failures <br>**Will be addressed by<br>https://redhat.atlassian.net/browse/CNF-26381**| ~~cloud-event-proxy~~ |
| G6 | LOW | PARTIAL | 6.2 PTPSourceQualified (separate condition) | A source-qualified gate now exists for ts2phc startup (`NotifyTs2phcSourceQualified`), but it is not a distinct FSM condition. InSync still uses `inSyncConditionThreshold` in a hardcoded 64-sample filter. No `ptpSourceQualifiedThreshold`/`ptpSourceDisqualifiedThreshold`/`ptpSourceUseS3` | np-linuxptp-daemon |
| G7 | MEDIUM | OPEN | FUNC-TBC013-R3 (MaxOffsetThreshold not for T-BC) | `MaxOffsetThreshold` still used as fallback for `inSyncConditionThreshold` and in `shouldFreeRun()` for phc2sys/ts2phc log parsing | np-linuxptp-daemon |
| G8 | MEDIUM | OPEN | FUNC-TBC014-R2 (holdover duration from oscillator model) | `LocalHoldoverTimeout` still drives a hard `time.After` timer that forces Free-Run on expiry; offset grows as `slope·t`. No oscillator-model-derived duration | np-linuxptp-daemon |
| G9 | LOW | OPEN | FUNC-TBC014-R4 (8-hour holdover module) | Only the linear model ΔT(t) = T₀ + S·t is implemented. The quadratic aging-dominated model ΔT(t) = T₀ + ½·A·t² for 8-hour modules is not implemented<br>**Setting as low for 5.1 - holdover research is still ongoing**| np-linuxptp-daemon |
| G10 | MEDIUM | OPEN | 6.2 HoldoverCapable | No `HoldoverCapable` concept. Source loss always enters holdover (T3) regardless of oscillator readiness; spec says non-capable systems go to Free-Run (T4) | np-linuxptp-daemon |
| G11 | MEDIUM | OPEN | FUNC-TBC016-R1 (ptp4l[TR] crash → holdover) | Holdover entry still driven by ptp4l log string parsing ("SLAVE to", ANNOUNCE_RECEIPT_TIMEOUT). Process death is not treated as source loss <br>**Will be addressed by<br>https://redhat.atlassian.net/browse/CNF-26381 and solution of G6**| np-linuxptp-daemon |
| G12 | LOW | OPEN | 12.5 SyncE events edge-triggered | SyncE events fire on every synce4l log line with no change detection. `PortState.LastQLState` is stored but never used to suppress re-publication <br>**Not in scope for 5.1**| cloud-event-proxy (obsolete) |
| G13 | MEDIUM | PARTIAL | 11.1.6 E3 threshold model | Operator branch `018-sysoffset-threshold-e3` adds `sysOffsetInSyncThreshold`/`sysOffsetOutOfSyncThreshold`/`sysOffsetSamples` as `ptpSettings` (webhook-validated, configmap passthrough). cloud-event-proxy does NOT consume them yet — E3 still uses shared `MaxOffsetThreshold`. Not on `main` | np-ptp-operator, cloud-event-proxy |
| G14 | LOW | OPEN | T-TSC clock class in events | cloud-event-proxy still does not suppress E2 for T-TSC (clock class always 255). E2 fires on any class change from daemon logs **T-TSC is not in scope for 5.1**| cloud-event-proxy (obsolete)|
| G15 | LOW | PARTIAL | T-TSC modeling in operator | Daemon enforces clock class 255 for T-TSC (`pkg/clock/tbc.go`). Operator still has no `clockType: "T-TSC"` (allowlist `["T-GM","T-BC"]`); T-TSC modeled as T-BC-without-controlled-ports <br>**T-TSC is not in scope for 5.1**| np-ptp-operator |
| G16 | LOW | OPEN | Upstream port redundancy (operator) | `upstreamPort`/`leadingInterface` still allowlisted in webhook with no validation or reconciliation logic; no dual-upstream validation | np-ptp-operator |
| G17 | LOW | OPEN | SyncE operator logic | SyncE fields (`synce4lOpts`, `synce4lConf`) still pure pass-through — no SyncE-specific operator logic <br>**Not in scope for 5.1**| np-ptp-operator |

---

## Epic Coverage Check (CNF-25637)

Release planning note: for the next release, `cloud-event-proxy` is not the delivery target. Eventing scope moves to CEP v2 in `np-linuxptp-daemon/pkg/cep`.

### Remaining MEDIUM gaps vs current epic coverage

| Gap | Current status in `gaps.md` | Existing child ticket(s) under CNF-25637 | Coverage assessment |
| :--- | :--- | :--- | :--- |
| G7 | OPEN | `CNF-26630` | **Partial**: generic `minOffsetThreshold` deprecation exists, but no explicit T-BC ticket for removing `MaxOffsetThreshold` fallback from T-BC in-sync/free-run paths |
| G8 | OPEN | `CNF-23950`, `CNF-26620` | **Partial**: quadratic model and oscillator data tickets exist; explicit removal of `LocalHoldoverTimeout` hard cutoff is not clearly tracked as a deliverable |
| G10 | OPEN | `CNF-26673` | **Covered**: placeholder exists for Holdover-capable conditions/observability |
| G11 | OPEN | _none_ (related `CNF-26381` is under epic `CNF-26597`, not `CNF-25637`) | **Missing in this epic**: no dedicated child to treat ptp4l[TR] process death as source loss/holdover trigger |
| G13 | PARTIAL | `CNF-26775`, `CNF-26631` | **Partial**: E3 conditioning tickets exist for ptp-operator, but need the same for np-linuxptp-daemon |

### Potential Jira stories to add under CNF-25637 (do not create yet)

1. **[T-BC][5.1][daemon] Remove MaxOffsetThreshold fallback from T-BC qualification paths (G7)**  
   Short description: Remove `MaxOffsetThreshold` fallback usage for T-BC in-sync/free-run decisions; require explicit T-BC thresholds and keep behavior aligned with `inSyncConditionThreshold` + PTPSourceQualified semantics.

2. **[T-BC][5.1][daemon] Implement OS CLOCK state conditioning filter (G13)**  
   Short description: Wire `sysOffsetInSyncThreshold`, `sysOffsetOutOfSyncThreshold`, and `sysOffsetSamples` into OS-clock state evaluation and event emission.

3. **[T-BC][5.1] Trigger Holdover on ptp4l[TR] process death (G11)**  
   Short description: Treat ptp4l[TR] process crash/kill as source-loss input to the T-BC state machine so holdover transitions occur even without log-based state strings. **Blocked on https://redhat.atlassian.net/browse/CNF-26381**

4. **[T-BC][5.1] Replace LocalHoldoverTimeout hard cutoff with oscillator-derived holdover window (G8 follow-up)**  
   Short description: Use oscillator model outputs to determine holdover validity duration and remove fixed `LocalHoldoverTimeout` forced free-run cutoff from runtime behavior.
