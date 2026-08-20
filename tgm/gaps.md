# T-GM Requirements vs Code — Gap Analysis

**Generated**: 2026-08-20
**Spec**: [tgm/spec.md](spec.md)
**Codebases analyzed**: linuxptp-daemon, ptp-operator, cloud-event-proxy

---

## Gap Summary

| # | Severity | Requirement | Gap Description | Affected Repo(s) |
| :--- | :--- | :--- | :--- | :--- |
| G1 | HIGH | §6.2/§6.5 (T6/T9), FUNC-TGM001-R3, FUNC-TGM008-R7 | No distinct Holdover-Out-Of-Spec state. When holdover goes out of spec, the DPLL/composite reports FREERUN lock-state (clockClass advances to 140), so the four-state model collapses to three lock-states. `dpll.go` carries an explicit TODO. | linuxptp-daemon |
| G2 | HIGH | FUNC-TGM003-R2/R3 (Antenna Survey) | The system issues a GNSS survey-in command but does not wait for completion or persist the fixed 3D position after completion; the `SAVE` runs immediately after the survey is *started*. | linuxptp-daemon |
| G3 | HIGH | FUNC-TGM011-R1/R2 (Drift Monitoring) | No position drift monitoring is implemented. The system does not restart the antenna survey when drift exceeds `surveyMinAccuracy`. | linuxptp-daemon |
| G4 | HIGH | FUNC-TGM006-R4 (phc2sys startup) | `phc2sys` startup is delayed but gated on `abs(offset) < 1s` from the reporting process, rather than explicitly waiting for `ts2phc` to report the LOCKED (`s2`) state. | linuxptp-daemon |
| G5 | HIGH | FUNC-TGM007 (processDowntimeThresholds) | `processDowntimeThresholds` exists in the operator CRD but is not referenced by the daemon or cloud-event-proxy. Process restart events fire with no downtime-suppression window. | linuxptp-daemon, ptp-operator |
| G6 | HIGH | §10.2, §14(3), FUNC-TGM010-R3 (Spoof/Jam) | No GNSS spoofing/jamming detection. GNSS monitoring keys only off `gpsFix` and offset; `UBX-NAV-STATUS` spoof/jam flags are not parsed, so an attack that preserves a plausible fix will not force FREERUN. | linuxptp-daemon |
| G7 | MEDIUM | FUNC-TGM005-R4 (GNSS State Event) | The newer in-daemon IPC/cep pipeline defines and maps `event.sync.gnss-status.gnss-state-change`, but no producer emits the `ipc.TypeGNSSState` message. (The legacy cloud-event-proxy log-parse path does still publish GNSS events.) | linuxptp-daemon |
| G8 | MEDIUM | §8.4/§8.5, FUNC-TGM004-R4 (Free-Run frequencyTraceable) | In Free-Run (clockClass 248) the announce advertises `frequencyTraceable=TRUE`; the spec requires FALSE. The flag is set unconditionally for the T-GM and never cleared in the Free-Run branch. | linuxptp-daemon |
| G9 | MEDIUM | PERF-TGM002-R2 (Holdover Duration Metric) | The holdover timer is tracked internally but no Prometheus metric exposes elapsed holdover time. Only offset is exposed. | linuxptp-daemon |
| G10 | MEDIUM | §13.1.2, FUNC-TGM003-R4 (GNSS config) | `antennaCableDelay` and `1ppsPulseWidth` are not user-configurable GNSS parameters. `GNSSInit` exposes only antenna voltage, constellations, survey-in and extra commands. | ptp-operator, linuxptp-daemon |
| G11 | MEDIUM | §13.1.1, FUNC-TGM002-R5 (Profile validation) | Profile-mandated (Tier-1) PTP parameters (`network_transport=L2`, `delay_mechanism`, `time_stamping=hardware`, `dataset_comparison`, `transportSpecific=0x0`, `priority1=128`) are not validated/rejected at admission time against the declared G.8275.1 profile. | ptp-operator |
| G12 | LOW | §11.1.5 (note), FUNC-TGM007-R5 | `process_restart_count` increments on the initial process start and on config-change restarts, and the per-config series is deleted on config change — contrary to the spec's "cumulative since daemon start, excluding config-change restarts" definition. | linuxptp-daemon |

---

## Detailed Findings by Repository

### linuxptp-daemon

| Gap | File | Line(s) | Current Behavior | Required Behavior |
| :--- | :--- | :--- | :--- | :--- |
| G1 | `pkg/dpll/dpll.go` | 678-696 | `DPLL_HOLDOVER` out-of-spec branch sets `state = PTP_FREERUN` with an explicit `// TODO: GNSS holdover currently doesn't offer holdover out of spec` note. | Represent Holdover-Out-Of-Spec as a distinct state that stays HOLDOVER (E1 unchanged) while clockClass advances 7→140; only cross to Free-Run on the holdover-to-freerun threshold (T9). |
| G1 | `pkg/event/event.go` | 445-451 | Composite `updateGMState` reports `state=FREERUN` and `clockClass=140` when `outOfSpec && frequencyTraceable`. | The two holdover sub-states must be differentiable via `clock_state`, not only `clock_class`. |
| G2 | `pkg/hardwareconfig/gnss.go`, `pkg/ublox/ublox.go` | `gnss.go:65-77,106-127`, `ublox.go:157` | Issues `ubxtool -e SURVEYIN` with a 5s wait; appends a single `SAVE` at the end of the init sequence, i.e. right after the survey starts. | Wait for survey completion (hours) and persist the fixed 3D position to NVM after a fixed position is acquired. |
| G3 | `pkg/ublox/*`, `pkg/daemon/gpsd.go` | — | No monitoring of GNSS position drift after fixed-position mode. | Monitor position drift and re-trigger survey when it exceeds `surveyMinAccuracy`. |
| G4 | `pkg/daemon/daemon.go` | 2146-2154 | `HandleDelayedPhc2sysStartup` enables `phc2sys` when `math.Abs(offset) < 1000000000` (sub-second). | Gate `phc2sys` on `ts2phc` reaching the LOCKED (`s2`) servo state, per §9.2 step 5. |
| G5 | `pkg/daemon/*`, `pkg/event/*` | — | `ProcessDowntimeThresholds` is never read by the daemon; state-change/event emission has no suppression window. | Parse `processDowntimeThresholds.*` and suppress state-change events for restarts within the per-process threshold (§9.5). |
| G6 | `pkg/daemon/gpsd.go`, `pkg/ublox/ublox.go` | `gpsd.go:274-310`, `ublox.go:288-306` | `processGNSSLines` derives `sourceLost` solely from `gpsFix>=3` and offset range; `ExtractNavStatus` parses only `gpsFix`. | Parse `UBX-NAV-STATUS` flags/jamming indicators and force GNSS FREERUN on spoof/jam detection. |
| G7 | `pkg/event/event.go`, `pkg/ipc/ipc.go`, `pkg/cep/cep.go` | `ipc.go:16`, `cep.go:34-35` | `TypeGNSSState` is defined and mapped to `GnssStateChange`, but `pkg/event` does not import `pkg/ipc` and no producer constructs a `GNSSStateValue`. | Wire the daemon's GNSS state into the structured IPC producer so the new cep transport emits gnss-state-change. |
| G8 | `pkg/event/event.go` | 1120, 1155-1166 | `case GM:` sets `FrequencyTraceable = true` unconditionally; the `ClockClassFreerun` branch sets `TimeTraceable=false` but never clears `FrequencyTraceable`. | Set `FrequencyTraceable = false` for Free-Run (class 248). |
| G9 | `pkg/daemon/metrics.go`, `pkg/metrics/register.go` | — | No metric tracks elapsed time in holdover; only `offset_ns` is exposed. | Add and publish a holdover-duration gauge/counter for operator compliance checks. |
| G10 | `pkg/hardwareconfig/gnss.go` | 106-127 | `buildGNSSInitCommands` emits antenna-voltage, constellation, survey-in and extra commands only. No cable-delay or 1PPS pulse-width command. | Emit receiver config for `antennaCableDelay` and `1ppsPulseWidth`. |
| G12 | `pkg/daemon/metrics.go` | 555-562, 651-656 | `UpdateProcessStatusMetrics` increments `ProcessRestartCount` on every transition to UP (incl. first start); `deleteProcessStatusMetrics` deletes the series on config change. | Exclude initial start and config-change restarts; keep the counter cumulative since daemon start. |

**Matching requirements (no gap):**
- Core transitions T1/T2/T3/T4/T5/T7 and announce content for classes 6/7/248, including Locked `clockAccuracy=0x21` and `offsetScaledLogVariance=0x4e5d` (`pkg/event/event.go` `updateClockClass`).
- Composite lowest-quality state evaluation across GNSS/DPLL/ts2phc (§6.3).
- Holdover model parameters `MaxInSpecOffset`, `LocalMaxHoldoverOffSet`, `LocalHoldoverTimeout` and timer/slope (`pkg/dpll/dpll.go`).
- Process startup ordering and automatic restart without reordering healthy processes (§9.2/§9.3).
- Prometheus metrics of §11.1 (`clock_state`, `clock_class`, `offset_ns`, `delay_ns`, `frequency_adjustment_ns`, `phase_status`, `frequency_status`, `interface_role`, `process_status`, `process_restart_count`).
- SyncE EEC state and QL/EXT-QL monitoring (§5.3; note S-01 SyncE↔clock-state coupling is FFS in the spec).
- PTP auth key-material (`sa_file`) change detection with process restart (FUNC-TGM010-R2).
- GNSS constellation selection, survey-in start, and custom commands (FUNC-TGM003-R1/R2/R5).

### ptp-operator

| Gap | File | Line(s) | Current Behavior | Required Behavior |
| :--- | :--- | :--- | :--- | :--- |
| G5 | `api/v1/ptpconfig_types.go` | 100-118 | `ProcessDowntimeThresholds` CRD field is defined (default 5s per process) but ignored by the daemon side. | Keep CRD; implement consumption in the daemon (no operator change required beyond the existing field). |
| G10 | `api/v2alpha1/hardwareconfig_types.go` | 167-207 | `GNSSInit` has `AntennaVoltage`, `Constellations`, `SurveyIn`, `ExtraCommands` only. | Add `antennaCableDelay` and `1ppsPulseWidth` fields (or document the delay-compensation-model equivalent for cable delay). |
| G11 | `api/v1/ptpconfig_webhook.go` | 125-283 | `validate()` checks scheduling/priority, allowed `PtpSettings` keys, HA syntax, secret/`sa_file`/`spp` consistency and `masterOnly`, but not profile-mandated ptp4l parameters. | Reject configs whose Tier-1 parameters violate the declared G.8275.1 profile, with an informative admission error naming the parameter. |

**Matching requirements (no gap):**
- `clockType` in `{T-GM, T-BC}` accepted by the webhook.
- `ProcessDowntimeThresholds` schema present with correct defaults/bounds.
- HardwareConfig GNSS/DPLL/pin topology structures present.

### cloud-event-proxy

| Gap | File | Line(s) | Current Behavior | Required Behavior |
| :--- | :--- | :--- | :--- | :--- |
| G5 | `plugins/ptp_operator/metrics/*` | — | No `processDowntimeThresholds` handling; restart-driven events are not suppressed. | Implement threshold-aware suppression in coordination with the daemon. |

**Matching requirements (no gap):**
- Event types E1–E5 and SyncE events are defined in `sdk-go` and published from `plugins/ptp_operator/metrics/` (`PtpStateChange`, `PtpClockClassChange`, `OsClockSyncStateChange`, `SyncStateChange`, `GnssStateChange`, `SynceStateChange`, `SynceClockQualityChange`).
- GNSS state-change events are published via the log-parse path (`plugins/ptp_operator/metrics/logparser.go:656`), so FUNC-TGM005-R4 is functionally satisfied through the legacy pipeline (see G7 for the new IPC transport).
- Clock-class events (E2) are edge-triggered — published only when the class value changes (`ptp4lParse.go`).

---

## Priority Recommendations

### Must fix before next release (HIGH)

1. **G1 (Holdover-Out-Of-Spec state)**: Implement a first-class out-of-spec holdover state so the PTP lock-state stays HOLDOVER while clockClass advances 7→140, mirroring the programmable T-BC transitions referenced in the `dpll.go` TODO.
2. **G5 (processDowntimeThresholds)**: Consume the CRD field in the daemon (and cloud-event-proxy) to suppress transient restart events. CRD is ready.
3. **G4 (phc2sys startup gating)**: Gate `phc2sys` on `ts2phc` reaching `s2`, not on a sub-second offset proxy.
4. **G2/G3 (survey completion & drift/re-survey)**: Wait for survey completion and persist the fixed position; monitor drift and re-survey on `surveyMinAccuracy` breach.
5. **G6 (spoofing/jamming)**: Parse `UBX-NAV-STATUS` spoof/jam flags and force GNSS FREERUN.

### Should fix (MEDIUM)

6. **G8 (Free-Run frequencyTraceable)**: Clear `FrequencyTraceable` in the Free-Run branch of `updateClockClass`.
7. **G7 (GNSS state over new IPC)**: Wire the daemon's GNSS state into the structured IPC producer.
8. **G9 (holdover duration metric)**: Add and publish a holdover-duration metric.
9. **G10 (GNSS cable delay / pulse width)**: Add `antennaCableDelay` and `1ppsPulseWidth` and emit the receiver configuration.
10. **G11 (profile-mandated validation)**: Extend the `PtpConfig` webhook with profile-aware Tier-1 parameter enforcement. (Also resolve the §5.7 vs §13.1.1 `delay_mechanism` P2P/E2E inconsistency in the spec.)

### Nice to have (LOW)

11. **G12 (process_restart_count semantics)**: Exclude initial start and config-change restarts; keep the counter cumulative since daemon start.
