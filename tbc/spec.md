# Feature Specification: T-BC Testable Requirements

**Module**: PTP Operator Stack (np-ptp-operator, np-linuxptp-daemon, cloud-event-proxy)
**Author**: Vitaly Grinberg
**Created**: 2026-06-16
**Last Updated**: 2026-07-07
**Status**: Approved

---

## 1. Overview

Testable requirement specification for the Telecom Boundary Clock (T-BC)
implementation. Defines desired behavior
across all operating states, performance metrics, state machines, and event
contracts. Describes WHAT the system must do and WHY — not HOW.
Any delta between current code and this specification is a bug or future backlog item.

---

## 2. Goals

- Establish a single, traceable source of truth for T-BC behavior and performance
- Enable agentic SDLC tooling and automated gap analysis
- Define all behavioral outcomes independent of current code limitations

## 3. Non-Goals

- Duplicating upstream standards (IEEE 1588, ITU-T G.8273.2, G.8275)
- Prescribing implementation details (languages, frameworks, internal APIs)
- Defining T-GM (Grandmaster) behavior (covered separately)
- Defining T-TSC behavior (see [T-TSC spec](../ttsc/spec.md))

---

## 4. Glossary and Standards References

4.1 Abbreviations and Acronyms
- FFS - For Further Study
- T-BC - Telecom Boundary Clock
- T-TSC - Telecom Time Synchronous Clock
- TR - Time Receiver
- TT - Time Transmitter
- DPLL - Digital Phase-Locked Loop
- SyncE - Synchronous Ethernet
- WPC - Westport Channel
- PHC - PTP Hardware Clock
- PMC - PTP Management Client
- (A-)BMCA - (Alternate) Best Master Clock Algorithm
- NIC - Network Interface Card
- JBOD - Just a Bunch of Devices
- OCXO - Oven-Controlled Crystal Oscillator
- cTE - Constant Time Error
- dTE - Dynamic Time Error
- maxTE - Maximum Time Error
- MTIE - Maximum Time Interval Error
- TDEV - Time Deviation

4.2 Normative References
- IEEE 1588-2019 (PTP v2.1)
- ITU-T G.8273.2 — T-BC/T-TSC timing characteristics
- ITU-T G.8275.1 — PTP telecom profile for phase/time (with full timing support)
- ITU-T G.8275 (2024) Amd.1 — Clock modes and state definitions

4.3 Informative References
- O-RAN O-Cloud Notification API v2
- CloudEvents v1.0 specification

---

## Clarifications

### Session 2026-06-30
- Q: Should ts2phc and phc2sys maintain a specific relative ordering now that both require PTPSourceQualified, or start concurrently? → A: phc2sys starts first (step 2), then ts2phc (step 3) — ts2phc depends on phc2sys for ToD
- Q: Should ts2phc restart after a crash also require PTPSourceQualified, or bypass the precondition? → A: Restart bypasses the precondition (restart immediately, same as current bypass rule)
- Q: How should the spec structurally accommodate multiple clock-type startup orders? → A: Each clock-type spec defines its own startup order; T-TSC spec links to T-BC section 9.2
- Q: Should the Guard Condition column in 6.4 include inline descriptions, or is reordering sections (diagram → definitions → conditions → table) sufficient? → A: Reorder only — proximity is sufficient
- Q: Should upstream port switchover and ptp4l[TR] crash have aligned event suppression behavior? → A: Yes — both use `processDowntimeThresholds.ptp4l` to suppress holdover events if resolved within threshold
- Q: Should the behavioral contract be a standalone section or distributed across config/observability? → A: Standalone section (§13 "User Contract") with Inputs/Outputs tables referencing detailed sections. Tiered configuration model (Tier 1/2/3) rejected in favor of three concerns: user intent, hardware config, K8s status
- Q: Should the user intent section define the full cross-clock-type model or T-BC only? → A: Full intent model (clock type, compliance class, PTP profile) defined here; other specs reference it
- Q: Which K8s observability elements to adopt from T-GM? → A: PtpConfig.status and Kubernetes events; not HardwarePtpDevice or health probes

### Session 2026-07-07
- Q: Should PTPSourceQualified be promoted from FFS to a documented baseline, and if so, kept conflated with InSync or separated? → A: Promote as-is (conflated with InSync). Document current moving-average implementation as baseline. Note that this gate may become obsolete with ptp4l S3 servo state integration.
- Q: Should HoldoverCapable follow the same FFS-to-baseline promotion pattern as PTPSourceQualified? → A: Yes — promote fallback (HoldoverCapable ≡ HoldoverDataValid) as baseline. Future refinement with FFO-based syntonization criteria is a must for general availability.
- Q: Does T4 (Locked→Free-Run) fire on both paths (offset spike OR source-lost-and-not-holdover-capable), or offset spike only? → A: Both paths as written. 

---

## 5. System Scope and Context

5.1 Clock Types in Scope
- T-BC (Telecom Boundary Clock): receives synchronization upstream, distributes downstream

For T-TSC (Telecom Time Synchronous Clock) requirements, see [T-TSC spec](../ttsc/spec.md).

5.2 Supported Topologies
- Single-NIC T-BC (one NIC with TR + TT ports)
- Multi-NIC T-BC (2-NIC and 3-NIC fan-out configurations)
- Phase / frequency (optional) transfer between NICs in multi-NIC T-BC configurations

5.3 SyncE (Synchronous Ethernet) Role

SyncE provides physical layer frequency assistance to the T-BC. When available, SyncE improves holdover performance by providing a frequency reference traceable to a Primary Reference Clock (PRC), enabling the DPLL to maintain better frequency accuracy during PTP source loss.

- SyncE is **optional** — the T-BC must operate correctly without SyncE (unassisted holdover)
- When SyncE is configured, the DPLL locks to both PTP (phase) and SyncE (frequency) sources
- SyncE state (EEC Locked/Holdover/Freerun) influences the `frequencyTraceable` flag in announce messages
- SyncE lock state is reported via the `event.sync.synce-status.synce-state-change` event (O-RAN 7.2.3.9)
- SyncE clock quality changes are reported via `event.sync.synce-status.synce-clock-quality-change` (O-RAN 7.2.3.11)

5.4 Platform Context
- Kubernetes-managed, operator-driven deployment
- DaemonSet-based per-node daemon lifecycle
- HTTP server for event subscription and metrics access

### 5.5 Synchronization Chain

The T-BC sits between an upstream PTP source (Grandmaster or another T-BC) and downstream PTP consumers (other T-BCs or end applications). Its job is to recover phase/frequency from the upstream source and redistribute it downstream with minimal added error.

**Signal flow in Locked state:**

```
Upstream GM                
    │                         
    │                         
    ▼                         
┌──────────┐    disciplines   ┌──────┐    phase         ┌──────────┐
│ ptp4l 1  │───────────────►  │ PHC1 │ ──────────────►  │   DPLL   │
│   (TR)   │                  │      │                  │ (OCXO)   │
└──────────┘                  └──┬───┘                  └──────────┘
                                 │                           │
                          ┌──────┤                           │ timestamps
                          │      │                           │
                          ▼      ▼                           ▼
                   ┌──────────┐ ┌────────────┐   ToD    ┌──────────┐
                   │ ptp4l 2  │ │  phc2sys   │─────────►│  ts2phc  │
             ┌─────│  (TT)    │ │ CLOCK_REAL │          │          │
             │     └──────────┘ └────────────┘          └──────────┘
             │            ▲                                  │
             ▼            │   ┌──────┐       disciplines     │
        Downstream        └───│ PHC2 │◄──────────────────────┘
          nodes               │      │                    
                              └──────┘
```

**Key components and their roles:**

| Component | What it does | Disciplines | Disciplined by |
| :--- | :--- | :--- | :--- |
| **ptp4l[TR]** | Runs PTP protocol on upstream (TR) port(s). Recovers PTP time from the upstream GM via Sync/DelayReq exchange | PHC1 | Upstream GM (via PTP) |
| **PHC1** | PTP Hardware Clock on NIC1 (upstream NIC). | DPLL (via 1PPS/freq output pins) | ptp4l[TR] (via servo) |
| **DPLL** | Digital PLL with OCXO. Locks to PHC1 signals in normal operation. Provides holdover stability when PHC1 is no longer disciplined | — | PHC1 (via input pins) |
| **ts2phc** | Disciplines PHC2 using DPLL timestamps and ToD from the system clock (via phc2sys) | PHC2 | DPLL (timestamps) + system clock (ToD) |
| **PHC2** | PTP Hardware Clock on NIC2 (downstream NIC). Disciplined by ts2phc from DPLL phase transfer | ptp4l[TT] | ts2phc |
| **phc2sys** | Synchronizes the OS system clock (`CLOCK_REALTIME`) to PHC1. Also provides ToD to ts2phc | System clock, ts2phc (ToD) | PHC1 |
| **ptp4l[TT]** | Runs PTP protocol on downstream (TT) port(s) in JBOD mode. Distributes Sync/Announce to downstream nodes. Uses PHC1 and PHC2 as clock sources | — | PHC1, PHC2 |
| **pmc** | PTP Management Client. Updates Announce message content (clock class, GM identity, time properties) on ptp4l[TT] via PMC commands during state transitions | ptp4l[TT] data sets | Supervision software |

**Signal flow in Holdover state:**

When the upstream source is lost, the sync direction between PHC and DPLL reverses:

```
Upstream GM                
    │                         
    │                         
    ▼                         
┌──────────┐                  ┌──────┐                        ┌──────────┐
│ ptp4l 1  │───────X          │ PHC1 │     disciplines        │   DPLL   │
│   (TR)   │                  │      │◄───────────────────┐   │ (OCXO)   │
└──────────┘                  └──┬───┘                    │   └──────────┘
                                 │                        │      │
                          ┌──────┤                        │      │ timestamps
                          │      │                        │      │
                          ▼      ▼                        │      ▼
                   ┌──────────┐ ┌────────────┐   ToD    ┌──────────┐
                   │ ptp4l 2  │ │  phc2sys   │─────────►│  ts2phc  │
             ┌─────│  (TT)    │ │ CLOCK_REAL │          │          │
             │     └──────────┘ └────────────┘          └──────────┘
             │            ▲                                  │
             ▼            │   ┌──────┐       disciplines     │
        Downstream        └───│ PHC2 │◄──────────────────────┘
          nodes               │      │                    
                              └──────┘
```

- The DPLL's OCXO free-runs on its last locked frequency, providing stability
- ts2phc disciplines both PHC1 and PHC2 using timestamps and ToD from the system clock
- ptp4l[TT] continues announcing to downstream, but with local GM identity and holdover clock class (135 or 165)
- ptp4l[TR] listening on the upstream port

### 5.6 Upstream Port Redundancy

A T-BC may be configured with multiple upstream (TR) ports on a single NIC for redundancy. This enables continuity of synchronization when one upstream link fails while another remains available.

**Behavioral requirements for redundant upstream ports:**

- The ptp4l[TR] instance operates on all configured upstream ports simultaneously
- The Alternate Best Master Clock Algorithm (A-BMCA) per G.8275.1 selects the active upstream source
- `ValidSourceAvailable` is TRUE when **at least one** upstream port ptp4l servo is in the S2/S3 (SLAVE) state
- `NoValidSourceAvailable` is TRUE only when **all** upstream ports have lost SLAVE state
- During a switchover (one port loses SLAVE, A-BMCA promotes another), there is a brief gap where no port is SLAVE. Entering holdover during this gap is correct because the DPLL is not being disciplined by PTP, and the ptp4l servo is not in the S2/S3 (SLAVE) state. When the new port reaches SLAVE, the PTPSourceQualified filter confirms stability before exiting holdover
- If the switchover resolves (a new port reaches SLAVE) within `processDowntimeThresholds.ptp4l` (default 5 s), holdover state-change events must be suppressed to avoid brief HOLDOVER→LOCKED toggles. This aligns switchover behavior with ptp4l[TR] process failure handling (see section 9.5)
- The `leadingInterface` setting determines which upstream port's offset is used for state machine decisions when multiple ports are in SLAVE state

5.7 Actors
- Cluster Administrator: configures T-BC profiles and thresholds
- Downstream PTP nodes: consume Announce/Sync from TT ports
- Upstream PTP nodes: provide Announce/Sync to TR ports
- Monitoring systems: subscribe to clock state events

---

## 6. T-BC State Machine

### 6.1 State Transition Diagram

``` 
                                   T2                           
  Init ──T1──►  ┌────────────────┐ ────► ┌────────────────┐
                │   Free-Run     │       │    Locked      │
                │  (class 248)   │ ◄──── │(class from GM) │◄──────────┐
                └────────────────┘  T4   └────────────────┘           │
                   ▲          ▲                 │      ▲              │
                   │          │                 │ T3   │  T5          │
                   │          │                 ▼      │              │
                   │          │     ┌─────────────────────┐           │
                   │    T7    │     │ Holdover-In-Spec    │           │
                   │          └──── │    (class 135)      │           │
                   │                └─────────────────────┘           │
                   │                           │                      │
                   │                           │ T6                   │
                   │                           ▼                      │
                   │                ┌─────────────────────┐           │
                   │       T9       │Holdover-Out-Of-Spec │           │
                   └─────────────── │    (class 165)      │──────T8───┘
                                    └─────────────────────┘
```

### 6.2 State Definitions

The T-BC supports four clock states for unassisted holdover. State names and semantics are derived from ITU-T G.8275 (2024) Amd.1, Section VIII.2.

**Free-Run** — The PTP clock has never been synchronized to a time source, or is in the process of synchronizing but has not yet reached acceptable accuracy. This state combines two modes from G.8275 Amd.1:
- *"The PTP clock has never been synchronized to a time source and is not in the process of synchronizing to a time source"*
- *"The PTP clock is in the process of synchronizing to a time source. The duration and functionality of this mode is implementation specific."*

In the Free-Run state the system is trying to acquire frequency and phase from the reference (if available) and to reach the acceptable accuracy defined by the InSync condition parameters. Clock class: **248**.

**Locked** — The PTP clock is synchronized to a time source and is within internal acceptable accuracy. Per G.8275 Amd.1: *"The PTP clock is synchronized to a time source and is within some internal acceptable accuracy."* While in the Locked state, the TR port must be continuously monitored for Announce timer expiry and phase offset threshold violations. Clock class: **forwarded from upstream GM**.

**Holdover-In-Spec** — The PTP clock has lost its synchronization source but is maintaining performance within the desired specification using information obtained while previously synchronized. Per G.8275 Amd.1: *"The PTP clock is no longer synchronized to a time source and is using information obtained while it was previously synchronized [...] in order to maintain performance within the desired specification. The node may be relying solely on its own facilities for holdover or may use something like a frequency input from the network to achieve a holdover of time and/or phase."* Clock class: **135**.

**Holdover-Out-Of-Spec** — The PTP clock has lost its synchronization source and can no longer maintain performance within the desired specification. Per G.8275 Amd.1: *"The PTP clock is no longer synchronized to a time source and [...] it is unable to maintain performance within the desired specification."* Clock class: **165**.

### 6.3 State Transition Conditions

The following conditions govern transitions between states. Each condition is evaluated continuously by the supervision software.

| Condition Name | Definition | Parameters |
| :--- | :--- | :--- |
| **ValidSourceAvailable** | The ptp4l servo on at least one of the configured upstream (TR) ports has reached the S2 (SLAVE) state as defined in IEEE 1588. S3 is also accepted where applicable. This is the logical inverse of NoValidSourceAvailable | — |
| **NoValidSourceAvailable** | No upstream (TR) port ptp4l servo is in the S2/S3 (SLAVE) state. This occurs on Announce timer expiry or port state change away from SLAVE | — |
| **PTPSourceQualified** | The upstream PTP source has been validated as stable enough to enable hardware sync direction (DPLL inputs) and to gate process startup (phc2sys, ts2phc — see §9.2). Currently equivalent to the InSync condition: a moving-average filter (window size 64, ~4 seconds) of ptp4l offsets compared against `inSyncConditionThreshold`. This conflation is intentional for the current release — the same filter and threshold that declare InSync also qualify the source for hardware activation and process startup. **Note**: this gate may become obsolete if ptp4l S3 servo state integration provides a native convergence signal, removing the need for a separate qualification filter | `inSyncConditionThreshold` |
| **InSync** | ValidSourceAvailable AND worst-case phase offset (across ptp4l, DPLL, ts2phc measurement points) ≤ inSyncConditionThreshold for inSyncConditionTimes consecutive samples AND all PPS DPLLs in "Locked Holdover Acquired" | `inSyncConditionThreshold`, `inSyncConditionTimes` |
| **HoldoverDataValid** | All PPS DPLLs have "Locked Holdover Acquired" as their current state | — |
| **HoldoverCapable** | The system can satisfy holdover requirements. Current baseline: equivalent to HoldoverDataValid (all PPS DPLLs in "Locked Holdover Acquired"). **GA REQUIREMENT**: this baseline is insufficient for general availability. A production-grade definition must incorporate syntonization criteria — e.g., fractional frequency offset (FFO) within bounds specified by the oscillator manufacturer, sustained for a minimum qualifying period — to confirm the OCXO has converged before certifying holdover capability. Without FFO-based qualification, the system may enter holdover with an uncalibrated oscillator, yielding drift far exceeding the modeled ΔT(t) | — |
| **Locked-To-FreeRun** | (**NoValidSourceAvailable** AND the system is not **HoldoverCapable**) OR any of the monitored phase offsets exceeds `LocalMaxHoldoverOffSet`. The first path (NOT HoldoverCapable) is expected only during early operation before the DPLL reaches LHAQ — once LHAQ is achieved, source loss routes through T3 (holdover) instead. The second path (offset spike) can fire at any time while Locked |`LocalMaxHoldoverOffSet` |
| **Offset-Above-InSpecOffset** | Any of the estimated or reported DPLL phase offsets exceeds MaxInSpecOffset. | `MaxInSpecOffset` |
| **Holdover-To-FreeRun** | Any of the estimated phase offsets exceeds LocalMaxHoldoverOffSet | `LocalMaxHoldoverOffSet` |

### 6.4 State Transition Table

| # | From State | To State | Guard Condition | Actions |
| :--- | :--- | :--- | :--- | :--- |
| T1 | **Init** | Free-Run | Always (initialization complete) | Determine clock type from configuration; start processes |
| T2 | **Free-Run** | Locked | InSync condition met | Forward upstream GM data via IWF to TT ports |
| T3 | **Locked** | Holdover-In-Spec | NoValidSourceAvailable AND HoldoverDataValid AND HoldoverCapable| Reverse hardware sync direction (DPLL→PHC); restart ptp4l[TR] in monitor mode; start ts2phc ToD transfer; update announce to local (class 135) |
| T4 | **Locked** | Free-Run | Locked-To-FreeRun condition | Update announce to local (class 248); configurable — can be disabled |
| T5 | **Holdover-In-Spec** | Locked | InSync condition met | Reverse hardware sync direction back (PHC→DPLL); restore upstream GM forwarding |
| T6 | **Holdover-In-Spec** | Holdover-Out-Of-Spec | Offset-Above-InSpecOffset AND NoValidSourceAvailable | Update announce (class 165, timeTraceable=FALSE) |
| T7 | **Holdover-In-Spec** | Free-Run | Holdover-To-FreeRun condition | Reverse hardware sync direction; update announce (class 248). Configurable — set LocalMaxHoldoverOffSet ≤ MaxInSpecOffset to skip Out-Of-Spec |
| T8 | **Holdover-Out-Of-Spec** | Locked | InSync condition met | Reverse hardware sync direction back (PHC→DPLL); restore upstream GM forwarding |
| T9 | **Holdover-Out-Of-Spec** | Free-Run | Holdover-To-FreeRun condition | Reinitialize hardware (disable all inputs and outputs); update announce (class 248) |

### 6.5 Initialization Behavior

The initialization transition occurs when the supervision software applies the node configuration. During this transition:

1. The current PTP clock type is determined from the configuration resource.
2. Processes are started in the required order with generated configurations.
3. DPLL pins are configured to the initial state for the determined clock type.
4. The system enters the **Free-Run** state unconditionally.

Initialization always ends in the Free-Run state regardless of whether an upstream source is already available. The system must progress through the InSync condition to reach Locked.

### 6.6 Configuration Use Cases

All the states and transitions in the state machine above are enabled by default. The behavior can be customized through threshold configuration. 

| User Intent | Configuration |
| :--- | :--- |
| "Never go from Locked to Free-Run if Holdover data is valid" | Set `LocalMaxHoldoverOffSet` to infinity (or a very large value). This essentially gives the control over the transition to Free-Run to the DPLL device: when it goes to Free-Run, the system will also go to Free-Run. |
| "Always go from Holdover-In-Spec directly to Free-Run (skip Out-Of-Spec)" | Set `LocalMaxHoldoverOffSet` ≤ `MaxInSpecOffset` |
| "Go to Free-Run when not holdover capable" | Locked-To-FreeRun triggers when source is lost and the system is not HoldoverCapable |
| "Go to Free-Run from any state on offset spike" | `LocalMaxHoldoverOffSet` controls the Locked-To-FreeRun transition |


### 6.7 General Timing Constraints

| Constraint | Value / Source |
| :--- | :--- |
| State transition event latency | Events must be published within **1 second** of the state transition |
| Announce message update latency | Announce content must reflect the new state within **1 second** after transition. Note: IEEE1588-2019, Section 9.6 permits some lack of coherence between announce messages on different TT ports: "Unless otherwise stated in the standard or the applicable PTP Profile, PTP messages transmitted by different PTP Ports of a PTP Instance may reflect the updated dataset values at different times." The **1 second** interval is FFS until a stricter requirement  is determined |
| Convergence time for re-lock | FFS — depends on servo algorithm and offset at re-acquisition |
| InSync filter window | `inSyncConditionTimes` consecutive samples below `inSyncConditionThreshold` |

### 6.8 Holdover Model and Configuration Parameters

#### 6.8.1 The industry model for clock time error
Note: This model is not used for T-BC, but is brought here for reference.
The industry accepted model for a clocktime error ΔT(t) as a function of time is expressed as:
ΔT(t) = T₀ + (Δf/f)·t + ½·A·t² + ε(t), where

- T₀ : Initial Time Offset: The phase error present at the exact moment the reference signal was lost. If the synchronization was perfect, this is ideally zero.

- Δf/f: Initial Frequency Offset: The difference between the oscillator's actual frequency and the target frequency at the start of holdover. This component causes a linear increase in time error.

- ½·A·t²: Frequency Drift / Aging: The rate at which the frequency changes over time due to the physical changes in the oscillator crystal. This component causes a quadratic (squared) increase in time error.

- ε(t): Phase Noise / Random Walk: Stochastic (random) noise that is unpredictable but contributes to the overall jitter and wander. This component is unpredictable due to its stochastic nature. 

#### 6.8.2 The simplified holdover model - FFS
For the practical application, where the stochastic part and the initial frequency offset part are hard to estimate, a simpler holdover model is sufficient. It relies on the manufacturer provided value for the maximum holdover time and the maximum offset drifted during this time:

ΔT(t) = S·t + ½·A·t²

The maximum values for ΔT and t are provided by the manufacturer as maximum holdover time and maximum offset drifted during this time. For example there are "8hours holdover module" and "4 hours holdover module", while the ΔT reached during this time is 1500ns. The S value is the slope of the linear part of the model, which is the maximum offset drift rate. The A value is oscillator aging component of the model, which is becoming significant at a longer holdover times

The model above can be sufficient for the practical application assuming the clocks are **fairly well syntonized**, so the initial frequency offset is within the bounds assumed by the manufacturer when specifying the maximum holdover time for the oscillator. This syntonization criteria, when fulfilled, makes the system **holdover capable** at the "source lost" event. Current baseline: HoldoverCapable is equivalent to HoldoverDataValid (all PPS DPLLs in LHAQ). **GA REQUIREMENT**: for general availability, this must be refined to include FFO-based syntonization criteria — verifying that the OCXO frequency offset is within manufacturer-specified bounds for a qualifying period before certifying holdover capability (see §6.3 HoldoverCapable).

#### 6.8.3 Holdover time calculation example
Assuming we have a "8hours holdover module" with 1500ns maximum offset drifted during this time, and the oscillator aging is 0.3 ppb/day, we can calculate the linear drift component S as follows:

S = (ΔT(max t) - ½·A·(max t)²) / (max t) 
Assuming T₀ = 0, the aging component is (in days) 0.5 * 0.3E-9 * (1/3) * (1/3) = 0.15 E-9 / 9 = 0.0166E-9 days, or 0.01666 E-9* 86400s = 1439ns, **which almost entirely consumes the budget of 1500ns for the holdover time**.
S = (1500ns - 1439ns ) / 8hours = 61ns / 28800s = 0.0000021ns/s - a negligible value. Hence the recommendation for the holdover offset calculation is to usse the simplified model with the linear component ignored

#### 6.8.4 Oscillator on-time requirement
Oscillator performance is guaranteed only after a sufficient power on time. The required power-on time is from 10 to 100 hours, depending on the preceding power off time. Reference: [OX-2211-EAE-5000-10M0000000.pdf](https://ww1.microchip.com/downloads/aemDocuments/documents/VOP/ProductDocuments/DataSheets/OX-2211-EAE-5000-10M0000000.pdf)

#### 6.8.5 Configuration Parameters
 The user API for configuring the holdover model consists of three parameters: `LocalMaxHoldoverOffSet`, `LocalHoldoverTimeout` and `MaxInSpecOffset`. These define the holdover-to-freerun boundary, the holdover duration and the in-spec/out-of-spec boundary. The holdover duration is derived from the oscillator model (see 6.8.1/6.8.2) and / or from a user-configurable timeout.

 For GNR-D systems, the oscillator model can be read from the hardware, while for WPC systems, the oscillator model is considered to be constant "4 hours holdover module" with 1500ns maximum offset drifted during this time.

#### 6.8.6 Offset estimation during holdover
##### 6.8.6.1 8-hour holdover module
Upon holdover entry, the system should estimate the offset as follows:
- If the system is **HoldoverCapable**, the offset is estimated as ΔT(t) = T₀ + ½·A·t² (See 6.8.1 for more details)
- If the system is not **HoldoverCapable**, the offset can't be estimated and the system should enter the Free-Run state.

##### 6.8.6.2 4-hour holdover module, or the WPC default holdover module
Upon holdover entry, the system should estimate the offset as follows:
- If the system is **HoldoverCapable**, the offset is estimated as ΔT(t) = T₀ + S·t with a linear component S = 104 ps/s
- If the system is not **HoldoverCapable**, the offset can't be estimated and the system should enter the Free-Run state.
---

## 7. Synchronization Direction and Hardware Reconfiguration

During normal (Locked) operation, the PHC disciplines the DPLL via 1PPS or a Ref-Sync signal pair through the DPLL input pins. During holdover, the direction reverses: the DPLL's internal oscillator free-runs (or disciplines the PHC), and PHC-to-DPLL input pins are disabled.

The pin state changes are triggered by T-BC state transitions (section 6.4). Pin names are platform-specific board labels that map to the DPLL and PHC subsystems.

### 7.1 WPC (E810) Pin States

WPC uses four DPLL-side pins (CVL-SDP20 through CVL-SDP23) and PHC-side SMA/SDP pins. The key sync-direction pins are CVL-SDP22 (PPS input to DPLL) and CVL-SDP23 (PPS output from DPLL).

**7.1.1 Initialization / Free-Run state**

| Pin | Target | State | Purpose |
| :--- | :--- | :--- | :--- |
| CVL-SDP20 | DPLL | Disabled (priority 255) | 10MHz input — not used at init |
| CVL-SDP21 | DPLL | Disconnected | 10MHz output — not used at init |
| CVL-SDP22 | DPLL | Disabled (priority 255) | PPS input from PHC — disabled until lock |
| CVL-SDP23 | DPLL | Disconnected | PPS output to PHC — disabled until holdover |
| SMA2/SDP22 | PHC | TX output, ch2, 1PPS period | PHC outputs 1PPS on SMA2 for external use |
| GNSS-1PPS | DPLL | Disabled (priority 255) | GNSS input — disabled for T-BC |

**7.1.2 Locked state (PTPSourceQualified — see 6.3)**

PHC disciplines DPLL: the E810 PHC 1PPS output (SMA2/SDP22) is already active from initialization. The only change is enabling the DPLL PPS input to receive it. This transition fires when the PTPSourceQualified condition is met (see section 6.3).

| Pin | Target | Change from Init | Purpose |
| :--- | :--- | :--- | :--- |
| SMA2/SDP22 | PHC | No change (already TX output with 1PPS period from init) | E810 PHC 1PPS output — set once at init, persists |
| CVL-SDP22 | DPLL | PPS priority → **0** (enabled), EEC is disabled | DPLL accepts 1PPS input from PHC |
| CVL-SDP23 | DPLL | Remains **disconnected** | DPLL output not needed — PHC is master |

**7.1.3 Holdover (PTP source lost)**

DPLL disciplines PHC (through ts2phc): the PPS input is disabled and the DPLL output is enabled. The DPLL holds frequency on its internal OCXO.

| Pin | Target | Change from Locked | Purpose |
| :--- | :--- | :--- | :--- |
| CVL-SDP22 | DPLL | PPS priority → **255** (disabled) | Stop PHC from disciplining DPLL |
| CVL-SDP23 | DPLL | **Connected** (output enabled) | DPLL 1PPS output disciplines PHC |

**7.1.4 Return to Locked (leaving holdover)**

Reverse of holdover entry — restores PHC→DPLL direction:

| Pin | Target | Change from Holdover | Purpose |
| :--- | :--- | :--- | :--- |
| CVL-SDP22 | DPLL | PPS priority → **0** (enabled) | Resume PHC disciplining DPLL |
| CVL-SDP23 | DPLL | **Disconnected** (output disabled) | Stop DPLL output to PHC |

### 7.2 GNR-D (E825) Pin States

GNR-D uses NAC PHC-side pins (SDP0, SDP2) to output 1PPS and 1kHz signals to the DPLL, and DPLL-side pins (REF0P, REF0N) as phase/frequency references. Pin board labels are platform-specific (Dell: `ETH01_SDP_TIMESYNC_0/2`, HPE: `1PPS_IN0/1`).

**7.2.1 Initialization / Free-Run state**

| Pin | Target | State | Purpose |
| :--- | :--- | :--- | :--- |
| SDP0 | PHC (NAC) | Disabled (func=0) | 1PPS output — disabled until lock |
| SDP2 | PHC (NAC) | Disabled (func=0) | 1kHz output — disabled until lock |
| REF0P | DPLL | PPS: **selectable** | DPLL ready to accept PPS input when available |
| REF0N | DPLL | PPS: **selectable** | DPLL ready to accept PPS input when available |
| GNSS inputs | DPLL | Disabled | GNSS not used for T-BC |

**7.2.2 Locked state (PTPSourceQualified — see 6.3)**

PHC outputs 1PPS and 1kHz to DPLL via SDP0/SDP2. DPLL locks on these signals. This transition fires when the PTPSourceQualified condition is met (see section 6.3).

| Pin | Target | Change from Init | Purpose |
| :--- | :--- | :--- | :--- |
| SDP0 | PHC (NAC) | TX output, ch1, **1PPS** period | PHC 1PPS output drives DPLL phase lock |
| SDP2 | PHC (NAC) | TX output, ch2, **1kHz** period | PHC 1kHz output drives DPLL frequency lock |
| REF0P/REF0N | DPLL | Remain **selectable** | DPLL accepts PHC signals |

**7.2.3 Holdover (PTP source lost)**

PHC outputs are disabled. DPLL holds frequency on its internal oscillator (OCXO). Unlike WPC, there is no explicit DPLL→PHC output pin; the DPLL free-runs and ToD is maintained by ts2phc from the system clock.

| Pin | Target | Change from Locked | Purpose |
| :--- | :--- | :--- | :--- |
| SDP0 | PHC (NAC) | **Disabled** (func=0) | Stop PHC 1PPS output to DPLL |
| SDP2 | PHC (NAC) | **Disabled** (func=0) | Stop PHC 1kHz output to DPLL |
| REF0P/REF0N | DPLL | Remain **selectable** (unchanged) | Ready to resume when PHC outputs restart |

**7.2.4 Return to Locked (leaving holdover)**

PHC outputs are re-enabled with 1PPS and 1kHz periods. DPLL re-locks on PHC signals.

| Pin | Target | Change from Holdover | Purpose |
| :--- | :--- | :--- | :--- |
| SDP0 | PHC (NAC) | TX output, ch1, **1PPS** period | Resume PHC 1PPS to DPLL |
| SDP2 | PHC (NAC) | TX output, ch2, **1kHz** period | Resume PHC 1kHz to DPLL |

---

## 8. Announce Message Behavior

The T-BC operates two separate ptp4l instances (TR and TT). Announce messages are transmitted by the TT instance on all downstream (masterOnly) ports. The content of these messages is managed via PMC commands and must reflect the current T-BC state. In the Locked state, the T-BC acts as an inter-working function (IWF), forwarding upstream GM data. In all other states, the T-BC announces itself as the grandmaster with local clock parameters.

### 8.1 Free-Run State Announce Content

| Information Element | Content |
| :--- | :--- |
| sourcePortIdentity | Local clockId of the T-BC + PortNumber |
| leap61 | FALSE |
| leap59 | FALSE |
| currentUtcOffsetValid | FALSE |
| ptpTimescale | TRUE |
| timeTraceable | FALSE |
| frequencyTraceable | TRUE/FALSE based on traceability to Cat 1 frequency source |
| currentUtcOffset | Current UTC offset (from leap second file) |
| grandmasterPriority1 | Configured priority1 (from ptp4lConf) |
| grandmasterClockQuality.clockClass | **248** |
| grandmasterClockQuality.clockAccuracy | Unknown (0xFE) |
| grandmasterClockQuality.offsetScaledLogVariance | 0xFFFF |
| grandmasterPriority2 | Configured priority2 (from ptp4lConf) |
| grandmasterIdentity | Local clockId |
| stepsRemoved | 0 |
| timeSource | INT_OSC (0xA0) |
| synchronizationUncertain | Not supported in this version |

### 8.2 Locked State Announce Content

In the Locked state, the T-BC forwards upstream GM announce data via an inter-working function (IWF). The only locally modified field is `stepsRemoved` (incremented by 1).

| Information Element | Content |
| :--- | :--- |
| sourcePortIdentity | Local clockId of the T-BC + PortNumber |
| leap61 | Value from upstream GM announce |
| leap59 | Value from upstream GM announce |
| currentUtcOffsetValid | Value from upstream GM announce |
| ptpTimescale | Value from upstream GM announce |
| timeTraceable | Value from upstream GM announce |
| frequencyTraceable | Value from upstream GM announce |
| currentUtcOffset | Value from upstream GM announce |
| grandmasterPriority1 | Value from upstream GM announce |
| grandmasterClockQuality.clockClass | Value from upstream GM announce |
| grandmasterClockQuality.clockAccuracy | Value from upstream GM announce |
| grandmasterClockQuality.offsetScaledLogVariance | Value from upstream GM announce |
| grandmasterPriority2 | Value from upstream GM announce |
| grandmasterIdentity | Value from upstream GM announce |
| stepsRemoved | Received stepsRemoved **+ 1** |
| timeSource | Value from upstream GM announce |
| synchronizationUncertain | Not supported in this version |

### 8.3 Holdover-In-Spec State Announce Content

The T-BC announces itself as the grandmaster with holdover parameters. Time remains traceable (the clock is still within specification). Clock accuracy is calculated from the oscillator drift ramp model.

| Information Element | Content |
| :--- | :--- |
| sourcePortIdentity | Local clockId of the T-BC + PortNumber |
| leap61 | FALSE, or advertise last known leap second event if expected |
| leap59 | FALSE, or advertise last known leap second event if expected |
| currentUtcOffsetValid | TRUE |
| ptpTimescale | TRUE |
| timeTraceable | **TRUE** |
| frequencyTraceable | TRUE/FALSE based on traceability to Cat 1 frequency source |
| currentUtcOffset | Current UTC offset (from leap second file) |
| grandmasterPriority1 | Configured priority1 (from ptp4lConf) |
| grandmasterClockQuality.clockClass | **135** |
| grandmasterClockQuality.clockAccuracy | **Calculated from oscillator ramp model** |
| grandmasterClockQuality.offsetScaledLogVariance | 0xFFFF |
| grandmasterPriority2 | Configured priority2 (from ptp4lConf) |
| grandmasterIdentity | Local clockId |
| stepsRemoved | 0 |
| timeSource | INT_OSC (0xA0) |
| synchronizationUncertain | Not supported in this version |

### 8.4 Holdover-Out-Of-Spec State Announce Content

The T-BC announces degraded holdover. Time is no longer traceable. Clock accuracy is unknown since the offset has exceeded MaxInSpecOffset.

| Information Element | Content |
| :--- | :--- |
| sourcePortIdentity | Local clockId of the T-BC + PortNumber |
| leap61 | FALSE, or advertise last known leap second event if expected |
| leap59 | FALSE, or advertise last known leap second event if expected |
| currentUtcOffsetValid | TRUE |
| ptpTimescale | TRUE |
| timeTraceable | **FALSE** |
| frequencyTraceable | TRUE/FALSE based on traceability to Cat 1 frequency source |
| currentUtcOffset | Current UTC offset (from leap second file) |
| grandmasterPriority1 | Configured priority1 (from ptp4lConf) |
| grandmasterClockQuality.clockClass | **165** |
| grandmasterClockQuality.clockAccuracy | Unknown (0xFE) |
| grandmasterClockQuality.offsetScaledLogVariance | 0xFFFF |
| grandmasterPriority2 | Configured priority2 (from ptp4lConf) |
| grandmasterIdentity | Local clockId |
| stepsRemoved | 0 |
| timeSource | INT_OSC (0xA0) |
| synchronizationUncertain | Not supported in this version |

### 8.5 Key Differences Across States

| Field | Free-Run | Locked | HO-In-Spec | HO-Out-Of-Spec |
| :--- | :--- | :--- | :--- | :--- |
| clockClass | 248 | from GM | 135 | 165 |
| clockAccuracy | Unknown | from GM | Calculated | Unknown |
| timeTraceable | FALSE | from GM | TRUE | FALSE |
| frequencyTraceable | per Cat 1 | from GM | per Cat 1 | per Cat 1 |
| grandmasterIdentity | local | from GM | local | local |
| stepsRemoved | 0 | received+1 | 0 | 0 |
| timeSource | INT_OSC | from GM | INT_OSC | INT_OSC |
| currentUtcOffsetValid | FALSE | from GM | TRUE | TRUE |

### 8.6 Clock Accuracy Estimation in Holdover

- The clock accuracy in Holdover-In-Spec is derived from the oscillator drift ramp model (See 6.8.1 for more details)
- The corresponding IEEE 1588 `clockAccuracy` enumeration value is selected based on the estimated offset magnitude
- Programmability: users define oscillator parameters (drift slope, holdover timeout) to match their hardware
- GNR-D platforms may discover frequency source accuracy from DPLL hardware capabilities (FFS)

### 8.7 Announce Update Timing

- Announce content must be updated via PMC as soon as possible after a state transition
- During transitions from Holdover back to Locked, the IWF must fetch fresh upstream GM data from the ParentDS before updating downstream announces — stale holdover parameters must not persist
- The two-instance ptp4l architecture (TR + TT) prevents false announcements: TT ports never directly observe upstream state changes; all updates flow through explicit PMC commands

---

## 9. Process Orchestration

### 9.1 Process Inventory and Roles

| Process | Role | Instances per T-BC |
| :--- | :--- | :--- |
| ptp4l[TR] | PTP protocol engine on Time Receiver port(s) — phase/frequency recovery from upstream GM | 1 |
| ptp4l[TT] | PTP protocol engine on Time Transmitter port(s) — PHC discipline, Announce/Sync distribution (JBOD mode) | 1 |
| ts2phc | PHC discipline from external 1PPS timestamp; Time-of-day (ToD) provision from system clock during holdover | 1 |
| phc2sys | System clock (`CLOCK_REALTIME`) synchronization from the leading PHC | 1 |
| pmc | PTP Management Client — runtime monitoring of ParentDS, TimePropertiesDS; GMSettings/ClockClass updates on state transitions | Invoked on demand |

### 9.2 T-BC Process Startup Order

The process startup order is clock-type-specific. This section defines the order for T-BC. For T-GM startup order, see the T-GM specification.

For T-BC, processes must be started in the following order. Each step has a precondition that must be satisfied before starting the next process.

| Step | Process | Precondition | Rationale |
| :--- | :--- | :--- | :--- |
| 1 | **ptp4l[TR]** | none | ptp4l[TR] must start first to begin phase/frequency recovery from the upstream source. No other process depends on being started before it |
| 2 | **phc2sys** | PTPSourceQualified condition met (same condition as for enabling DPLL inputs — see section 6.3) | phc2sys must not start disciplining `CLOCK_REALTIME` from the PHC until the PHC is being meaningfully disciplined. Starting phc2sys before ptp4l convergence would cause the system clock to track a free-running PHC |
| 3 | **ts2phc** | PTPSourceQualified condition met (same condition as for enabling DPLL inputs — see section 6.3) | ts2phc disciplines downstream PHCs from DPLL timestamps and ToD from the system clock. It must not start until the upstream PTP source is fully qualified, otherwise it would discipline downstream PHCs from a free-running DPLL. Additionally, ts2phc depends on phc2sys for ToD, so phc2sys must be running first |
| 4 | **ptp4l[TT]** |T-BC transitions to S2 state | ptp4l[TT] distributes Announce/Sync to downstream nodes. It must not begin announcing until the clock quality is known and announce content can be set correctly via PMC |

### 9.3 Process Lifecycle

- **Configuration generation**: each process config is rendered from the PtpConfig profile and written to a temporary file before process startup
- **Scheduling policy**: all timing-critical processes (ptp4l, ts2phc, phc2sys) must run under `SCHED_FIFO` with configurable real-time priority
- **Graceful shutdown**: on configuration change or node shutdown, processes must be stopped (in any order).
- **Automatic restart**: if a process exits unexpectedly, it must be restarted automatically. The restart may bypass the startup order and preconditions (do not kill and restart other processes to adhere to the startup order). In particular, ts2phc and phc2sys restarts do not need to re-check PTPSourceQualified — the system was already qualified when they were originally started

### 9.4 Process State Monitoring

- Process state is extracted by parsing stdout/stderr output lines for known patterns (ptp4l port state changes, offset values, servo state codes s0/s1/s2/s3 where applicable)
- Each process is continuously monitored for health; failure detection triggers automatic restart (see 9.3)

### 9.5 T-BC Behavior on Process Failure

Every killed or crashed process is automatically restarted. The system behavior during the outage depends on which process failed and its impact on the synchronization chain. A configurable `processDowntimeThresholds` structure (in seconds, per process) defines the maximum acceptable downtime before state-change events are emitted.

**Configuration API** (defined in `PtpClockThreshold` within the PtpConfig CRD):

```yaml
ptpClockThreshold:
  processDowntimeThresholds:
    ptp4l: 5        # seconds, default 5
    phc2sys: 5      # seconds, default 5
    ts2phc: 5       # seconds, default 5
    synce4l: 5      # seconds, default 5
    chronyd: 5      # seconds, default 5
    gpsd: 1         # seconds, default 1
    gpspipe: 1      # seconds, default 1
```

| Field | Type | Range | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `ptp4l` | *int | 0–86400 | 5 | Acceptable downtime for ptp4l process |
| `phc2sys` | *int | 0–86400 | 5 | Acceptable downtime for phc2sys process |
| `ts2phc` | *int | 0–86400 | 5 | Acceptable downtime for ts2phc process |
| `synce4l` | *int | 0–86400 | 5 | Acceptable downtime for synce4l process |
| `chronyd` | *int | 0–86400 | 5 | Acceptable downtime for chronyd process |
| `gpsd` | *int | 0–86400 | 1 | Acceptable downtime for gpsd process |
| `gpspipe` | *int | 0–86400 | 1 | Acceptable downtime for gpspipe process |

If the process restarts within its configured threshold, state-change events are suppressed. If the downtime exceeds the threshold, the appropriate state-change events must be emitted.

| Process Failed | Impact on Synchronization | Desired Behavior | Event Behavior |
| :--- | :--- | :--- | :--- |
| **ptp4l[TR]** | NAC PHC is no longer disciplined, but still feeds the DPLL, which remains in LHAQ throughout the outage | Must enter holdover and flip the sync direction (DPLL→PHC) for the duration of the outage (same as TR link down). The holdover mechanism handles detection and recovery | If ptp4l[TR] restarts and re-acquires SLAVE within `processDowntimeThresholds.ptp4l`, holdover state-change events must be suppressed. If downtime exceeds the threshold, clock class change events must be emitted. Must NOT emit E3 (os-clock-sync-state-change) — phc2sys is unaffected. This same threshold applies to upstream port switchovers (see section 5.6) |
| **ptp4l[TT]** | PTP protocol timeout on downstream nodes | Must transition TT ports to Free-Run and back to Locked after ptp4l[TT] restarts and ports return to MASTER state (consider changing TT profile clock class to 248, so the ports will start free-running automatically)| Must emit E1 (ptp-state-change) toggle: FREERUN then LOCKED |
| **ts2phc** | None — back in ~1 s, no TE jumps | Restart silently. Since there is no impact on synchronization, event emission should be suppressed if downtime is within `processDowntimeThreshold` | No events if within threshold |
| **phc2sys** | None — restarted and functional in ~1 s | Restart silently. Since there is no impact on synchronization, event emission should be suppressed if downtime is within `processDowntimeThreshold` | No E3 toggle if within threshold. Current behavior (LOCKED→FREERUN→LOCKED toggle) is undesirable |


---

## 10. Clock components monitoring

10.1 Offset Measurement Points
- ptp4l master offset (TR port phase offset from upstream)
- DPLL phase offset
- DPLL fractional frequency offset (FFS)
- ts2phc offset per NIC
- phc2sys offset (system clock vs PHC)

10.2 Holdover Offset Estimation and use in the state transitions
- Clock time error model and simplifications
- Estimated time error accumulation during holdover
- Maximum in-spec holdover duration calculation

---
## 11. Observability and Diagnostics

### 11.1 Prometheus Metrics

The following metrics must be exposed for T-BC monitoring. All metrics use the `openshift_ptp_` prefix.

#### 11.1.1 Clock State and Class
The clock state for PHC and DPLL clocks is aliased by the related network interface name prefix, for example `enox`, `enp108s0fx`, etc. This isdone for the sake of readability and to avoid listing clock IDs. The overall clock state is represented by a virtual process "T-BC"

| Metric | Type | Labels | Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `openshift_ptp_clock_state` | gauge | `iface`, `node`, `process` | 0=FREERUN, 1=LOCKED, 2=HOLDOVER | Clock state per interface and process. Process values: `T-BC` (aggregate), `ptp4l`, `ts2phc`, `dpll`, `phc2sys` |
| `openshift_ptp_clock_class` | gauge | `config`, `node`, `process` | 6, 7, 135, 165, 248, 255 | Current clock class. 6=Locked, 7=PRC unlocked in-spec, 135=T-BC holdover in-spec, 165=T-BC holdover out-of-spec, 248=Free-Run, 255=Slave Only |

#### 11.1.2 Offset and Delay

| Metric | Type | Labels | Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `openshift_ptp_offset_ns` | gauge | `from`, `iface`, `node`, `process` | nanoseconds | Current phase offset. `from` values: `master` (ptp4l/ts2phc), `phc` (phc2sys), `dpll`, `T-BC` (aggregate) |
| `openshift_ptp_delay_ns` | gauge | `from`, `iface`, `node`, `process` | nanoseconds | Path delay measurement |
| `openshift_ptp_frequency_adjustment_ns` | gauge | `from`, `iface`, `node`, `process` | ppb | Frequency adjustment applied by servo to the clock frequency, in ppb |
| `openshift_ptp_ffo_ppt` | gauge | `from`, `iface`, `node`, `process` | ppt | Fractional frequency offset reported by the DPLL active pin relatively to the DPLL clock island frequency, in parts per trillion |

Note 1: openshift_ptp_max_offset_ns should not be used and will be deprecated in the future.


#### 11.1.3 DPLL Status

| Metric | Type | Labels | Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `openshift_ptp_phase_status` | gauge | `from`, `iface`, `node`, `process` | -1=UNKNOWN, 0=INVALID, 1=FREERUN, 2=LOCKED, 3=LOCKED_HO_ACQ, 4=HOLDOVER | DPLL phase lock status |
| `openshift_ptp_frequency_status` | gauge | `from`, `iface`, `node`, `process` | -1=UNKNOWN, 0=INVALID, 1=FREERUN, 2=LOCKED, 3=LOCKED_HO_ACQ, 4=HOLDOVER | DPLL frequency lock status |

#### 11.1.4 Interface Role

| Metric | Type | Labels | Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `openshift_ptp_interface_role` | gauge | `iface`, `node`, `process` | 0=PASSIVE, 1=SLAVE, 2=MASTER, 3=FAULTY, 4=UNKNOWN, 5=LISTENING | PTP port state per interface (from ptp4l) |

#### 11.1.5 Process Health

| Metric | Type | Labels | Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `openshift_ptp_process_status` | gauge | `config`, `node`, `process` | 0=DOWN, 1=UP | Current process liveness |
| `openshift_ptp_process_restart_count` | counter | `config`, `node`, `process` | monotonic count | Cumulative process restart count since daemon start. The count must not include restarts due to configuration changes. |

#### 11.1.6 Configuration Thresholds

| Metric | Type | Labels | Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `openshift_ptp_threshold` | gauge | `node`, `profile`, `threshold` | ns or s (per threshold) | Active threshold values per profile |


Threshold names exposed for T-BC:

| `threshold` label value | Unit | Source (section) | Description |
| :--- | :--- | :--- | :--- |
| `MaxInSpecOffset` | ns | 6.3 Offset-Above-InSpecOffset | Holdover in-spec / out-of-spec boundary |
| `LocalMaxHoldoverOffSet` | ns | 6.3 Holdover-To-FreeRun | Holdover-to-freerun boundary |
| `LocalHoldoverTimeout` | s | 6.8.5 Configuration Parameters | Holdover timeout |
| `inSyncConditionThreshold` | ns | 6.3 InSync | Offset threshold for declaring clock locked and enabling DPLL inputs |
| `inSyncConditionTimes` | count | 6.3 InSync | Consecutive samples below `inSyncConditionThreshold` required for lock |
| `SysOffsetThreshold` | ns | 12.5 E3 | phc2sys system clock lock threshold — offset above this triggers E3 FREERUN |
| `SysOffsetSamples` | count | 12.5 E3 | Consecutive phc2sys samples below `SysOffsetThreshold` required for E3 LOCKED |

Legacy thresholds (not used in T-BC state machine):

| `threshold` label value | Status | Notes |
| :--- | :--- | :--- |
| `MaxOffsetThreshold` | Not used for T-BC | Legacy parameter from `ptpClockThreshold`. Must be ignored by T-BC state machine |
| `MinOffsetThreshold` | Not used for T-BC | Deprecated. Must be ignored |
| `HoldOverTimeout` | Not used for T-BC | Legacy parameter from `ptpClockThreshold`. Must be ignored by T-BC state machine. Holdover duration is derived from the oscillator model (see 6.8) |

### 11.2 Kubernetes Object Status and Events

#### 11.2.1 PtpConfig Status

The `PtpConfig` custom resource status must report any unrecoverable error detected when applying the configuration. The error information must include the node name, the error condition, and if possible additional diagnostic information. Synchronization state and quality are reported via Prometheus metrics (§11.1) and must not be duplicated in the CRD status.

| Status Field | Description |
| :--- | :--- |
| `matchList` | Node-to-profile matching results |
| `error.nodeName` | Node on which the error occurred |
| `error.condition` | Error condition (e.g., hardware config failed, process failed to start, invalid pin configuration) |
| `error.message` | Additional diagnostic information when available |

#### 11.2.2 NodePtpDevice Status

The `NodePtpDevice` custom resource status must report PTP-capable hardware discovered on the node and the result of hardware configuration.

| Status Field | Description |
| :--- | :--- |
| `devices[].name` | PTP device name (e.g., `/dev/ptp0`) |
| `devices[].hardwareInfo` | Vendor ID, device ID, PCI address, driver |


#### 11.2.3 HardwareConfig Status

The `HardwareConfig` custom resource status must report clock chain health, enabling operators to verify that the declared hardware topology is operational.

| Status Field | Description |
| :--- | :--- |
| `matchedNodes` | Nodes matched to this hardware config |
| `clockChainState` | Per-subsystem DPLL lock state and source condition (FFS) |
| `appliedCondition` | Name of the currently active behavior condition (FFS) |

#### 11.2.4 Kubernetes Events

The system must emit Kubernetes Events on the `PtpConfig` resource for operational anomalies and configuration lifecycle events. Clock synchronization state is reported via metrics (§11.1) and CloudEvents (§12) and must not be duplicated here. Events provide an audit trail visible via `kubectl describe` and are forwarded to cluster-level event sinks.

| Event Reason | Type | Condition |
| :--- | :--- | :--- |
| `ConfigurationApplyFailed` | Warning | PtpConfig or HardwareConfig could not be applied to a node |
| `HardwareConfigFailed` | Warning | Hardware configuration failed on a device (pin setup, DPLL init) |
| `ProcessStartFailed` | Warning | A managed process failed to start or repeatedly crashes |
| `TimeDiscontinuity` | Warning | Significant time irregularity detected (e.g., time jumped backwards, PHC step beyond threshold) |
| `LeapSecondUpdate` | Normal | Leap second data updated (new UTC offset or pending leap event) |

### 11.3 Logging Requirements
- State transition log entries
- Offset threshold crossing warnings
- PMC update confirmations
- Hardware reconfiguration audit trail
- DPLL device and pin reports information

---

## 12. CloudEvents and Notification Behavior

12.1 Event Types
  - PTP Lock State Change: LOCKED / FREERUN / HOLDOVER with offset and upstream interface
  - PTP Clock Class Change: clock class value transitions (6, 7, 135, 140, 165, 248)
  - OS Clock Sync State: system clock synchronization status
  - GNSS State Change: GPS receiver status (when applicable)
  - Overall Sync State: aggregated synchronization assessment

12.2 Event Generation Rules
  - Edge-triggered: events published only on state transitions, not periodically
  - Required data payload per event type (offset, state, interface, clock class)
  - Event ordering and causality guarantees

12.3 Event Delivery Contract
  - CloudEvents v1.0 envelope format
  - O-RAN O-Cloud Notification API v2 compliance
  - Resource address hierarchy (/sync/ptp-status/lock-state, etc.)
  - Subscription model (endpoint URI + resource address filter)

12.4 Event Data Models
  - PTP Lock State event payload structure
  - Clock Class event payload structure
  - OS Clock Sync event payload structure
  - Overall Sync State event payload structure

### 12.5 Events Per State Transition

The following matrix defines which O-RAN O-Cloud Notification API v4.00 events must be generated for each T-BC state transition. Event types and resource addresses are per O-RAN.WG6.O-Cloud Notification API v04.00, section 7.2.3.

**O-RAN event types relevant to T-BC:**

| # | Event Type | Resource Address | value_type | O-RAN Section |
| :--- | :--- | :--- | :--- | :--- |
| E1 | `event.sync.ptp-status.ptp-state-change` | `/sync/ptp-status/lock-state` | enumeration (LOCKED/HOLDOVER/FREERUN) | 7.2.3.3 |
| E2 | `event.sync.ptp-status.ptp-clock-class-change` | `/sync/ptp-status/clock-class` | metric (Uint8) | 7.2.3.10 |
| E3 | `event.sync.sync-status.os-clock-sync-state-change` | `/sync/sync-status/os-clock-sync-state` | enumeration (LOCKED/HOLDOVER/FREERUN) | 7.2.3.8 |
| E4 | `event.sync.sync-status.synchronization-state-change` | `/sync/sync-status/sync-state` | enumeration (LOCKED/HOLDOVER/FREERUN) | 7.2.3.1 |

**Event generation matrix per T-BC state transition:**

| Transition (from §6.4) | T-BC State Change | E1: PTP Lock State | E2: Clock Class | E4: Overall Sync State |
| :--- | :--- | :--- | :--- | :--- |
| T1: Init → Free-Run | Initial state | FREERUN | 248 | FREERUN |
| T2: Free-Run → Locked | Lock acquired | LOCKED | from GM (e.g. 6) | worst-of(LOCKED, E3) |
| T3: Locked → Holdover-In-Spec | Source lost | HOLDOVER | 135 | worst-of(HOLDOVER, E3) |
| T4: Locked → Free-Run | Offset spike | FREERUN | 248 | FREERUN |
| T5: HO-In-Spec → Locked | Re-lock | LOCKED | from GM (e.g. 6) | worst-of(LOCKED, E3) |
| T6: HO-In-Spec → HO-Out-Of-Spec | Offset > MaxInSpec | — (remains HOLDOVER) | 165 | — (unchanged) |
| T7: HO-In-Spec → Free-Run | Holdover-To-FreeRun | FREERUN | 248 | FREERUN |
| T8: HO-Out-Of-Spec → Locked | Re-lock | LOCKED | from GM (e.g. 6) | worst-of(LOCKED, E3) |
| T9: HO-Out-Of-Spec → Free-Run | Holdover-To-FreeRun | FREERUN | 248 | FREERUN |

**Notes:**
- "—" means the event is not generated for this transition (no state change for that event type)
- E2 (Clock Class) fires on every clock class value change, including the sub-state transition T6 (135→165) where E1 does not fire (PTP state remains HOLDOVER)
- All events are edge-triggered: published only when the value changes, not periodically
- GNSS events (`event.sync.gnss-status.gnss-state-change`) are not applicable to T-BC (GNSS is T-GM)
- SyncE events (`event.sync.synce-status.synce-state-change` and `event.sync.synce-status.synce-clock-quality-change`) are applicable when SyncE is configured (see section 5.3)

**E3: OS Clock Sync State**

Per O-RAN O-Cloud Notification API v04.00, Table 37, E3 reports the synchronization state of the Operating System real-time clock (`CLOCK_REALTIME`). The E3 LOCKED state explicitly requires synchronization to a **"traceable & valid time/phase source"** — not just phc2sys operational success.

E3 is determined by two factors:

1. **phc2sys operational state**: is phc2sys successfully disciplining CLOCK_REALTIME to the PHC? (offset within `SysOffsetThreshold`)
2. **Upstream traceability**: is the PHC itself traceable? This is derived from the E1 (PTP Lock State)

**E3 state derivation:**

| E1 (PTP State) | phc2sys status | E3 (OS Clock) | O-RAN Rationale |
| :--- | :--- | :--- | :--- |
| LOCKED | offset < threshold | **LOCKED** | OS clock synchronized to traceable & valid source |
| HOLDOVER | offset < threshold | **HOLDOVER** | OS clock in holdover — tracking a holdover-mode PHC |
| FREERUN | offset < threshold | **FREERUN** | PHC not traceable; OS clock locked to invalid source |
| any | offset > threshold | **FREERUN** | OS clock not locked to any reference |
| any | phc2sys process down | **FREERUN** | OS clock not disciplined |

**E3 trigger conditions:**

| E3 Trigger | E3 Value Emitted | Condition |
| :--- | :--- | :--- |
| phc2sys offset converges AND E1 = LOCKED | LOCKED | phc2sys offset < `SysOffsetThreshold` for `SysOffsetSamples` consecutive samples, and PHC is traceable |
| E1 transitions to HOLDOVER (phc2sys still OK) | HOLDOVER | PHC enters holdover — OS clock follows into holdover by extension |
| E1 transitions to FREERUN | FREERUN | PHC has no valid upstream reference — OS clock is free-running regardless of phc2sys offset |
| phc2sys offset exceeds threshold | FREERUN | phc2sys offset > `SysOffsetThreshold` |
| phc2sys process stops or PHC becomes unavailable | FREERUN | phc2sys unable to discipline system clock |
| E1 returns to LOCKED + phc2sys offset recovers | LOCKED | Both traceability and phc2sys operation restored |

Key behavioral notes:
- E3 = LOCKED while E1 = HOLDOVER is **NOT valid** — O-RAN defines HOLDOVER as a distinct state; if PHC is in holdover, OS clock is in holdover too
- E3 = LOCKED while E1 = FREERUN is **NOT valid** — OS clock is locked to a free-running clock with no traceable reference
- E3 = HOLDOVER while E1 = HOLDOVER is the **correct** state — phc2sys continues disciplining from a holdover-mode PHC
- E3 must NOT fire due to ptp4l[TR] process failure alone (see G5 in gaps.md) — the holdover mechanism should handle this as NoValidSourceAvailable, leading to E1 = HOLDOVER → E3 = HOLDOVER

**E4: Overall Sync State — aggregation event**

E4 aggregates E1 (PTP Lock State) and E3 (OS Clock Sync State) into a single overall node synchronization health indicator. E4 uses a **worst-of** rule with the following priority: FREERUN > HOLDOVER > LOCKED.

| E1 (PTP Lock State) | E3 (OS Clock Sync) | E4 (Overall Sync State) |
| :--- | :--- | :--- |
| LOCKED | LOCKED | **LOCKED** |
| LOCKED | HOLDOVER | **HOLDOVER** |
| LOCKED | FREERUN | **FREERUN** |
| HOLDOVER | HOLDOVER | **HOLDOVER** |
| HOLDOVER | FREERUN | **FREERUN** |
| FREERUN | FREERUN | **FREERUN** |

Note: some E1/E3 combinations are not reachable in practice (e.g., E1=LOCKED + E3=HOLDOVER requires phc2sys to be in holdover while PTP is locked — unlikely but possible during convergence). E1=FREERUN + E3=LOCKED/HOLDOVER is not valid per the E3 derivation table above (E3 follows E1 into FREERUN). E1=HOLDOVER + E3=LOCKED is not valid (E3 follows E1 into HOLDOVER).

E4 is re-evaluated and (if changed) emitted whenever either E1 or E3 changes. The T-BC transition matrix above shows E4 values assuming E3 remains stable — if E3 is in a degraded state at the time of a T-BC transition, the actual E4 value may differ from what the matrix shows.

---

## 13. User Contract

This section defines the behavioral contract between the system and its users: what information the system requires from the user (inputs), and what information the system provides back (outputs).

### 13.1 User Inputs — Declaring Intent

The user configures the system through two Kubernetes custom resources: the `PtpConfig` (PTP software configuration and clock behavior) and the `HardwareConfig` (clock chain hardware topology). The intent declaration spans three layers.

#### 13.1.1 Clock Type and IEEE 1588 Profile

The user declares the desired clock role and the IEEE 1588 PTP profile. The system derives process topology, startup order, announce behavior, and state machine semantics from this declaration.

| Parameter | Values | Effect |
| :--- | :--- | :--- |
| `clockType` | `T-GM`, `T-BC`, `T-TSC` | Determines the set of processes started, port roles (TR/TT), announce behavior, and which state machine specification applies |
| `ptpProfile` | IEEE 1588 profile identifier (see table below) | Determines transport, delay mechanism, BMCA variant, domain number range, and which PTP parameters are profile-mandated |

**IEEE 1588 PTP Profile taxonomy:**

IEEE 1588 defines a profile mechanism (§20.3) allowing industry organizations to specify parameter selections and optional features for specific applications. Profiles are grouped by industry:

| Category | Profile | Standard | Transport | Delay | Accuracy | Clock Types |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Telecom** | Full timing support | ITU-T G.8275.1 | L2 multicast | E2E | ns-level | T-GM, T-BC, T-TSC |
| **Telecom** | Partial timing support | ITU-T G.8275.2 | UDP unicast | E2E | ns-level | T-GM, T-BC-P, T-TSC-P |
| **Telecom** | Frequency only | ITU-T G.8265.1 | UDP unicast | E2E | — | OC (freq) |
| **Telecom** | APTS | ITU-T G.8275.1 + GNSS fallback | L2 multicast | E2E | ns-level | T-GM with GNSS + PTP backup |
| **Power** | Substation automation | IEC/IEEE 61850-9-3 | L2 multicast | P2P | µs-level | OC, BC, TC |
| **Power** | Power system relay | IEEE C37.238-2017 | L2 multicast | P2P | µs-level | OC, BC, TC |
| **Media** | Broadcast / IP video | SMPTE ST 2059-2 | L2/UDP | P2P | sub-ms | OC, BC |
| **Media** | Audio-over-IP | AES67 | L2/UDP | — | sub-ms | OC, BC |
| **TSN** | Time-Sensitive Networking | IEEE 802.1AS-2025 (gPTP) | L2 | P2P | µs-level | GM, bridge |
| **Enterprise** | Enterprise mixed | IETF RFC 9760 | UDP | E2E | sub-ms | OC, BC |
| **Default** | IEEE 1588 default E2E | IEEE 1588 Annex J | L2/UDP | E2E | varies | OC, BC, TC |
| **Default** | IEEE 1588 default P2P | IEEE 1588 Annex J | L2/UDP | P2P | varies | OC, BC, TC |

When a profile is designated, certain PTP parameters are fixed by the profile and must not be overridden by the user. The system must reject invalid combinations at admission time with an informative error identifying the violating parameter.

**Example: G.8275.1 profile-mandated parameters:**

| Parameter | Mandated value | Rationale |
| :--- | :--- | :--- |
| `network_transport` | `L2` | G.8275.1 requires Layer 2 multicast transport |
| `delay_mechanism` | `E2E` | G.8275.1 requires end-to-end delay measurement |
| `time_stamping` | `hardware` | Software timestamping is insufficient for telecom accuracy |
| `dataset_comparison` | `G.8275.x` | Required for G.8275.1 alternate BMCA behavior |
| `priority1` | `128` | G.8275.1 alternate BMCA ignores priority1; value must remain at 128 |

**T-BC clock-type-specific mandated parameters (regardless of profile):**

| Parameter | Default | Range | Rationale |
| :--- | :--- | :--- | :--- |
| `clock_type` | `T-BC` | `T-BC` | Operate as Telecom Boundary Clock |
| `MaxInSpecOffset` | — | positive integer (ns) | Holdover in-spec / out-of-spec boundary (§6.3 Offset-Above-InSpecOffset) |
| `LocalMaxHoldoverOffSet` | — | positive integer (ns) | Holdover-to-freerun boundary (§6.3 Holdover-To-FreeRun) |
| `LocalHoldoverTimeout` | — | positive integer (s) | Maximum holdover duration (§6.8.5) |
| `inSyncConditionThreshold` | — | positive integer (ns) | Offset threshold to declare clock locked and enable DPLL inputs (§6.3 InSync) |
| `inSyncConditionTimes` | — | positive integer | Consecutive samples below `inSyncConditionThreshold` required for lock (§6.3 InSync) |
| `SysOffsetThreshold` | — | positive integer (ns) | phc2sys system clock lock threshold — offset above this triggers E3 FREERUN (§12.5) |
| `SysOffsetSamples` | — | positive integer | Consecutive phc2sys samples below `SysOffsetThreshold` required for E3 LOCKED (§12.5) |
| `processDowntimeThresholds.ptp4l` | `5` | 0–86400 (s) | Acceptable downtime before holdover events are emitted (§9.5) |
| `processDowntimeThresholds.phc2sys` | `5` | 0–86400 (s) | Acceptable downtime before E3 events are emitted (§9.5) |
| `processDowntimeThresholds.ts2phc` | `5` | 0–86400 (s) | Acceptable downtime before events are emitted (§9.5) |

Profiles not carrying a designation are treated as unconstrained and may set any `ptp4l` parameter to any value. The system must never silently restrict an undesignated profile.

#### 13.1.2 Compliance Class

The user declares the target ITU-T compliance class. This affects performance thresholds, holdover behavior, and noise transfer characteristics.

| Parameter | Values | Effect |
| :--- | :--- | :--- |
| `complianceClass` | `C`, `D` | Determines applicable G.8273.2 performance limits (noise generation, noise tolerance, noise transfer bandwidth). Class D imposes stricter low-pass filtered time error limits (see §15) |

The compliance class is informational for the state machine but constraining for performance validation. The system must expose the configured compliance class in metrics and status for external validation tools.

#### 13.1.3 Hardware Configuration (Clock Chain)

The user declares the hardware clock chain topology through the `HardwareConfig` custom resource. This resource describes the physical synchronization path: DPLL complexes, pins, connectors, phase/frequency inputs and outputs, delay compensations, and behavioral conditions for dynamic reconfiguration.

The HardwareConfig is composed of two parts:

| Part | Description |
| :--- | :--- |
| **Structure** | Static declaration of subsystems (DPLLs, Ethernet ports, pins, connectors), including phase/frequency inputs and outputs with delay compensation values. Each subsystem is associated with a hardware plugin |
| **Behavior** | Declaration of synchronization sources, conditions (locked/lost), and the desired pin states for each condition. The system dynamically switches between condition-matched states based on source availability |

When a `HardwareConfig` is provided, it takes precedence over plugin-derived hardware configuration for the associated PTP profile. When no `HardwareConfig` is provided, the system applies hardware-specific defaults from the plugin.

### 13.2 System Outputs — What the System Provides

The system provides synchronization state and health information through multiple channels. Each channel serves a different consumer and access pattern.

| Output Channel | Content | Consumer | Reference |
| :--- | :--- | :--- | :--- |
| **Prometheus metrics** | Clock state, offsets, DPLL status, interface roles, process health, thresholds | Monitoring dashboards, alerting | §11.1 |
| **CloudEvents / O-RAN notifications** | State transitions (E1–E4), clock class changes | Event-driven consumers, O-RAN O-Cloud | §12 |
| **Kubernetes Events** | Clock state changes, process restarts on the PtpConfig resource | `kubectl describe`, cluster event sinks | §11.2.2 |
| **PtpConfig.status** | Configuration errors (node, condition, diagnostic message) | Kubernetes controllers, operators | §11.2.1 |
| **NodePtpDevice.status** | Discovered PTP devices, hardware info, per-device config status (failed/success) | Kubernetes controllers, hardware inventory | §11.2.2 |
| **HardwareConfig.status** | Matched nodes, active behavior condition per node | Kubernetes controllers, clock chain verification | §11.2.3 |
| **Structured logs** | State transitions, offset measurements, PMC updates, hardware reconfiguration | Troubleshooting, audit trail | §11.3 |
| **PTP Announce messages** | Clock class, clock accuracy, time properties, GM identity | Downstream PTP nodes | §8 |

---
## 14. Functional Requirements Specification

### 14.1 State Machine — Lock Acquisition

### ID: FUNC-TBC001

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC001-R1 | 6.2 Free-Run, 6.3 InSync | Given a T-BC is initialized in Free-Run state, when a valid upstream PTP source becomes available and the InSync condition is met, then the T-BC must transition to Locked state |
| FUNC-TBC001-R2 | 8.2 Locked State Announce | Upon entering Locked state, announce messages on all TT ports must reflect upstream GM parameters via the inter-working function (IWF) |

### 14.2 State Machine — Holdover Entry on Source Loss

### ID: FUNC-TBC002

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC002-R1 | 6.3 NoValidSourceAvailable, 6.4 T3 | Given a T-BC is in Locked state, when the upstream TR port loses its PTP source (NoValidSourceAvailable), then the T-BC must transition to Holdover-In-Spec, if it is holdover capable |
| FUNC-TBC002-R2 | 7.1 / 7.2 Pin states | Upon entering holdover, the hardware sync direction must reverse (DPLL input disabled, DPLL output enabled) |
| FUNC-TBC002-R3 | 8.3 HO-In-Spec Announce | Announce messages must reflect holdover parameters: clockClass 135, gmIdentity = local, timeTraceable = TRUE |

### 14.3 State Machine — Holdover In-Spec to Out-Of-Spec

### ID: FUNC-TBC003

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC003-R1 | 6.3 Offset-Above-InSpecOffset, 6.4 T6 | Given a T-BC is in Holdover-In-Spec state, when the estimated phase offset exceeds MaxInSpecOffset, then the T-BC must transition to Holdover-Out-Of-Spec |
| FUNC-TBC003-R2 | 8.4 HO-Out-Of-Spec Announce | Announce messages must reflect degraded parameters: clockClass 165, clockAccuracy = Unknown, timeTraceable = FALSE |

### 14.4 State Machine — Holdover to Free-Run

### ID: FUNC-TBC004

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC004-R1 | 6.3 Holdover-To-FreeRun, 6.4 T9 | Given a T-BC is in Holdover-Out-Of-Spec state, when the Holdover-To-FreeRun condition is met, then the T-BC must transition to Free-Run |
| FUNC-TBC004-R2 | 6.3 Holdover-To-FreeRun, 6.4 T7 | Given a T-BC is in Holdover-In-Spec state, when the Holdover-To-FreeRun condition is met, then the T-BC must transition directly to Free-Run (skipping Holdover-Out-Of-Spec) |
| FUNC-TBC004-R3 | 8.1 Free-Run Announce | Announce messages must reflect free-run parameters: clockClass 248, clockAccuracy = Unknown, timeTraceable = FALSE |
| FUNC-TBC004-R4 | 7.1 / 7.2 Pin states | Upon entering Free-Run from holdover, the hardware sync direction must reverse back (DPLL output disabled) |

### 14.5 State Machine — Re-Lock from Holdover

### ID: FUNC-TBC005

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC005-R1 | 6.3 InSync, 6.4 T5/T8 | Given a T-BC is in any Holdover state, when the upstream PTP source is re-acquired and the InSync condition is met, then the T-BC must transition to Locked state |
| FUNC-TBC005-R2 | 7.1 / 7.2 Pin states | Upon re-lock, the hardware sync direction must reverse back to normal (PHC disciplines DPLL) |
| FUNC-TBC005-R3 | 8.2 Locked State Announce | Announce messages must be restored to upstream GM IWF parameters |

### 14.6 State Machine — Emergency Free-Run on Offset Spike

### ID: FUNC-TBC006

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC006-R1 | 6.3 Locked-To-FreeRun, 6.4 T4 | Given a T-BC is in Locked state, when the Locked-To-FreeRun condition is met, then the T-BC must transition to Free-Run (see FUNC-TBC013 for threshold and configurability details) |
| FUNC-TBC006-R2 | 12.1 PTP Lock State Change | A PTP lock state change event must be published upon this transition |

### 14.7 Multi-NIC — Announcement Coherence

### ID: FUNC-TBC007

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC007-R1 | 8.7 Announce Update Timing | Given a multi-NIC T-BC, when the clock changes state or clock accuracy, then all TT ports must reflect the changes in their announce messages within 1 second after the transition |
| FUNC-TBC007-R2 | 8.5 Key Differences | All TT ports must announce identical clockClass, clockAccuracy, gmIdentity, and timeTraceable values at any point in time |

### 14.8 Events — State Change Notification

### ID: FUNC-TBC008

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC008-R1 | 12.1 PTP Lock State Change | Given a monitoring system is subscribed to PTP lock state events, when the T-BC transitions between any two states, then a CloudEvents-formatted notification must be published |
| FUNC-TBC008-R2 | 12.2 Event Generation Rules | The event must contain the new state, current offset, and interface name |

### 14.9 Events — Clock Class Change

### ID: FUNC-TBC009

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC009-R1 | 12.1 PTP Clock Class Change | Given a monitoring system is subscribed to clock class change events, when the T-BC clock class changes (e.g., 6 → 135 → 165 → 248), then a clock class change event must be published with the new value |
| FUNC-TBC009-R2 | 12.3 Event Delivery Contract | The event must conform to CloudEvents v1.0 envelope format and O-RAN O-Cloud Notification API v2 |

### 14.10 Events — Per-Transition Event Matrix Compliance

### ID: FUNC-TBC010

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC010-R1 | 12.5, T1 | Given a T-BC completes initialization, when it enters Free-Run, then E1 must emit FREERUN, E2 must emit clock class 248, and E4 must emit FREERUN |
| FUNC-TBC010-R2 | 12.5, T2 | Given a T-BC is in Free-Run, when it transitions to Locked, then E1 must emit LOCKED, E2 must emit the upstream GM clock class, and E4 must emit worst-of(LOCKED, current E3) |
| FUNC-TBC010-R3 | 12.5, T3 | Given a T-BC is in Locked state, when it transitions to Holdover-In-Spec, then E1 must emit HOLDOVER, E2 must emit clock class 135, and E4 must emit worst-of(HOLDOVER, current E3) |
| FUNC-TBC010-R4 | 12.5, T4 | Given a T-BC is in Locked state, when it transitions to Free-Run, then E1 must emit FREERUN, E2 must emit clock class 248, and E4 must emit FREERUN |
| FUNC-TBC010-R5 | 12.5, T5 | Given a T-BC is in Holdover-In-Spec, when it transitions to Locked, then E1 must emit LOCKED, E2 must emit the upstream GM clock class, and E4 must emit worst-of(LOCKED, current E3) |
| FUNC-TBC010-R6 | 12.5, T6 | Given a T-BC is in Holdover-In-Spec, when offset exceeds MaxInSpecOffset, then E1 must NOT fire (state remains HOLDOVER), and E2 must emit clock class 165 |
| FUNC-TBC010-R7 | 12.5, T7 | Given a T-BC is in Holdover-In-Spec, when it transitions to Free-Run, then E1 must emit FREERUN, E2 must emit clock class 248, and E4 must emit FREERUN |
| FUNC-TBC010-R8 | 12.5, T8 | Given a T-BC is in Holdover-Out-Of-Spec, when it transitions to Locked, then E1 must emit LOCKED, E2 must emit the upstream GM clock class, and E4 must emit worst-of(LOCKED, current E3) |
| FUNC-TBC010-R9 | 12.5, T9 | Given a T-BC is in Holdover-Out-Of-Spec, when it transitions to Free-Run, then E1 must emit FREERUN, E2 must emit clock class 248, and E4 must emit FREERUN |
| FUNC-TBC010-R10 | 12.5, O-RAN Table 37 | Given phc2sys is running, then E3 must reflect both phc2sys offset AND upstream traceability: E3 = LOCKED only when phc2sys OK AND E1 = LOCKED; E3 = HOLDOVER when E1 = HOLDOVER and phc2sys OK; E3 = FREERUN when E1 = FREERUN (regardless of phc2sys) or phc2sys offset exceeds threshold. E3 must NOT fire due to ptp4l[TR] process failure alone |
| FUNC-TBC010-R11 | 12.5, Notes | Given any event type, when the value has not changed from the previously emitted value, then no event must be emitted (edge-triggered only) |

### 14.11 Void (T-TSC requirements moved to [T-TSC spec](../ttsc/spec.md))

### 14.12 Initialization — Clock Type and Initial State

### ID: FUNC-TBC012

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC012-R1 | 6.5 step 1 | Given a node configuration is applied, then the system must determine the PTP clock type (T-BC or T-GM) from the configuration resource |
| FUNC-TBC012-R2 | 6.5 step 2 | Given the clock type is determined, then processes (ptp4l, ts2phc, phc2sys) must be started in the required order with generated configurations |
| FUNC-TBC012-R3 | 6.5 step 3 | Given processes are starting, then DPLL pins must be configured to the initial state for the determined clock type (per section 7.1.1 / 7.2.1) |
| FUNC-TBC012-R4 | 6.5 step 4 | Given initialization completes, then the system must enter the Free-Run state unconditionally, regardless of whether an upstream source is already available |
| FUNC-TBC012-R5 | 6.5, 6.4 T1 | Given a T-BC has just initialized, when an upstream source is already available, then the system must NOT bypass Free-Run — it must progress through the InSync condition to reach Locked |

### 14.13 Configuration — Threshold Behavior

### ID: FUNC-TBC013

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC013-R1 | 6.6 | Given `LocalMaxHoldoverOffSet` is set to a very large value, when the T-BC loses its source in Locked state and HoldoverDataValid is true, then it must enter Holdover-In-Spec — never Free-Run |
| FUNC-TBC013-R2 | 6.6 | Given `LocalMaxHoldoverOffSet` ≤ `MaxInSpecOffset`, when the T-BC is in Holdover-In-Spec and offset exceeds the threshold, then it must transition directly to Free-Run, skipping Holdover-Out-Of-Spec |
| FUNC-TBC013-R3 | 6.3 Locked-To-FreeRun | Given a T-BC is in Locked state, then the Locked-To-FreeRun transition (T4) must trigger only when Locked to Freerun condition is met|
|

### 14.14 Holdover Model — Offset Estimation

### ID: FUNC-TBC014

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC014-R1 | 6.8.5 | Given a T-BC profile is configured, then the holdover model must be configurable via two parameters: `LocalMaxHoldoverOffSet` and `LocalHoldoverTimeout` |
| FUNC-TBC014-R2 | 6.8.5 | Given a T-BC enters holdover, then the holdover duration must be derived from the oscillator model or the user-configurable timeout. For GNR-D the model is read from hardware; for WPC a default "4-hour holdover module" with 1500 ns maximum offset is used |
| FUNC-TBC014-R3 | 6.8.5 | Given a WPC T-BC with no explicit oscillator model, then the default holdover model must be a "4-hour holdover module" with 1500 ns maximum offset and linear slope S = 104 ps/s |
| FUNC-TBC014-R4 | 6.8.2.1 | Given a T-BC with an 8-hour holdover module is HoldoverCapable and enters holdover, then the offset must be estimated as ΔT(t) = T₀ + ½·A·t² (aging-dominated model) |
| FUNC-TBC014-R5 | 6.8.2.2 | Given a T-BC with a 4-hour holdover module (WPC default) is HoldoverCapable and enters holdover, then the offset must be estimated as ΔT(t) = T₀ + S·t with S = 104 ps/s |
| FUNC-TBC014-R6 | 6.8.2 | Given a T-BC loses its source and the system is NOT HoldoverCapable, then it must transition to Free-Run |
| FUNC-TBC014-R7 | 6.8.4 | Given the oscillator has not reached sufficient power-on time (10–100 hours), then oscillator holdover performance must NOT be relied upon |
| FUNC-TBC014-R8 | 6.3 HoldoverCapable | Given the current baseline defines HoldoverCapable as equivalent to HoldoverDataValid (all PPS DPLLs in LHAQ), then the system must treat HoldoverDataValid as sufficient for holdover entry. **GA REQUIREMENT**: this must be refined to include FFO-based syntonization criteria before general availability (see §6.3 HoldoverCapable, §6.8.2) |

### 14.15 Process Orchestration — Startup Order and Lifecycle

### ID: FUNC-TBC015

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC015-R1 | 9.2 step 1 | Given a T-BC configuration is applied, then ptp4l[TR] must be the first timing process started — before phc2sys, ts2phc, or ptp4l[TT] |
| FUNC-TBC015-R2 | 9.2 step 2 | Given ptp4l[TR] is running but PTPSourceQualified is not yet met, then phc2sys must NOT be started |
| FUNC-TBC015-R3 | 9.2 step 3 | Given PTPSourceQualified is not yet met, then ts2phc must NOT be started. Additionally, ts2phc must not start before phc2sys, since ts2phc depends on phc2sys for ToD |
| FUNC-TBC015-R4 | 9.2 step 4 | Given PTPSourceQualified is not yet met, then ptp4l[TT] must NOT be started |
| FUNC-TBC015-R5 | 9.3 | Given a configuration change or shutdown occurs, then processes can be stopped in any order |
| FUNC-TBC015-R6 | 9.3 | Given any timing-critical process (ptp4l, ts2phc, phc2sys) is started, then it must run under SCHED_FIFO with configurable real-time priority |
| FUNC-TBC015-R7 | 9.3 | Given a process exits unexpectedly, then it must be restarted automatically, and dependent processes must NOT be stopped and restarted (do not kill and restart other processes to adhere to the startup order) |

### 14.16 Process Failure — T-BC Behavior

### ID: FUNC-TBC016

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC016-R1 | 9.5, ptp4l[TR] | Given ptp4l[TR] crashes or an upstream port switchover occurs, then the system must enter holdover. If ptp4l[TR] re-acquires SLAVE within `processDowntimeThresholds.ptp4l`, holdover state-change events must be suppressed. If downtime exceeds the threshold, clock class change events must be emitted |
| FUNC-TBC016-R2 | 9.5, ptp4l[TR] | Given ptp4l[TR] crashes, then E3 (os-clock-sync-state-change) must NOT be emitted — phc2sys and the PHC are unaffected |
| FUNC-TBC016-R3 | 9.5, ptp4l[TT] | Given ptp4l[TT] crashes, then the system must report TT ports as Free-Run (in metrics / events). The composite clock state of the system may remain in the previous state if upstream synchronization is unaffected. |
| FUNC-TBC016-R4 | 9.5, ts2phc | Given ts2phc crashes and restarts within `processDowntimeThresholds.ts2phc`, then no state-change events must be emitted |
| FUNC-TBC016-R5 | 9.5, phc2sys | Given phc2sys crashes and restarts within `processDowntimeThresholds.phc2sys`, then E3 (os-clock-sync-state-change) must NOT toggle LOCKED→FREERUN→LOCKED, unless the system clock drifts outside the `SysOffsetThreshold` during the downtime|
| FUNC-TBC016-R6 | 9.5, general | Given a PtpConfig CRD is applied, then `processDowntimeThresholds` must be configurable per process. Default values: 5 s for ptp4l/phc2sys/ts2phc/synce4l/chronyd, 1 s for gpsd/gpspipe. Range: 0–86400 s |
| FUNC-TBC016-R7 | 9.5, general | Given any timing process crashes, then it must be automatically restarted |

### 14.17 Hardware Pin State Verification

### ID: FUNC-TBC017

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC017-R1 | 7.1.1 WPC Init | Given a WPC T-BC is initialized, then CVL-SDP20/21/22/23 must be disabled/disconnected, GNSS-1PPS must be disabled, and SMA2/SDP22 must be configured to output 1PPS |
| FUNC-TBC017-R2 | 7.1.2 WPC Locked | Given a WPC T-BC transitions to Locked (PTPSourceQualified), then CVL-SDP22 PPS priority must be set to 0 (enabled) and CVL-SDP23 must remain disconnected |
| FUNC-TBC017-R3 | 7.1.3 WPC Holdover | Given a WPC T-BC enters holdover, then CVL-SDP22 must be disabled (priority 255) and CVL-SDP23 must be connected (output enabled) |
| FUNC-TBC017-R4 | 7.1.4 WPC Return | Given a WPC T-BC returns to Locked from holdover, then CVL-SDP22 PPS priority must be restored to 0 and CVL-SDP23 must be disconnected |
| FUNC-TBC017-R5 | 7.2.1 GNR-D Init | Given a GNR-D T-BC is initialized, then SDP0/SDP2 must be disabled (func=0), REF0P/REF0N must be PPS selectable, and GNSS inputs must be disabled |
| FUNC-TBC017-R6 | 7.2.2 GNR-D Locked | Given a GNR-D T-BC transitions to Locked (PTPSourceQualified), then SDP0 must output 1PPS (TX ch1) and SDP2 must output 1kHz (TX ch2) |
| FUNC-TBC017-R7 | 7.2.3 GNR-D Holdover | Given a GNR-D T-BC enters holdover, then SDP0 and SDP2 must be disabled (func=0) |
| FUNC-TBC017-R8 | 7.2.4 GNR-D Return | Given a GNR-D T-BC returns to Locked from holdover, then SDP0 must resume 1PPS output and SDP2 must resume 1kHz output |

### 14.18 Per-State Announce Content Verification

### ID: FUNC-TBC018

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC018-R1 | 8.1 Free-Run | Given a T-BC is in Free-Run state, then announce messages must contain: clockClass=248, clockAccuracy=Unknown(0xFE), gmIdentity=local, stepsRemoved=0, timeTraceable=FALSE, timeSource=INT_OSC(0xA0) |
| FUNC-TBC018-R2 | 8.2 Locked | Given a T-BC is in Locked state, then announce messages must forward all upstream GM parameters via IWF, with stepsRemoved = received+1 |
| FUNC-TBC018-R3 | 8.3 HO-In-Spec | Given a T-BC is in Holdover-In-Spec state, then announce messages must contain: clockClass=135, clockAccuracy=calculated from ramp model, gmIdentity=local, stepsRemoved=0, timeTraceable=TRUE, timeSource=INT_OSC(0xA0) |
| FUNC-TBC018-R4 | 8.4 HO-Out-Of-Spec | Given a T-BC is in Holdover-Out-Of-Spec state, then announce messages must contain: clockClass=165, clockAccuracy=Unknown(0xFE), gmIdentity=local, stepsRemoved=0, timeTraceable=FALSE, timeSource=INT_OSC(0xA0) |
| FUNC-TBC018-R5 | 8.5 Key Differences | Given a T-BC transitions between any two states, then currentUtcOffsetValid must be FALSE in Free-Run and TRUE in Holdover-In-Spec and Holdover-Out-Of-Spec |
| FUNC-TBC018-R6 | 8.6 Clock Accuracy | Given a T-BC is in Holdover-In-Spec state, then clockAccuracy must be derived from the oscillator drift ramp model and mapped to the IEEE 1588 clockAccuracy enumeration |

### 14.19 Clock Component Monitoring

### ID: FUNC-TBC019

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC019-R1 | 10.1 ptp4l offset | Given a T-BC is in Locked state, when ptp4l reports master offset on the TR port, then the offset must be continuously monitored and available as a metric |
| FUNC-TBC019-R2 | 10.1 DPLL offset | Given a T-BC has a DPLL, then the DPLL phase offset must be continuously monitored and available as a metric |
| FUNC-TBC019-R3 | 10.1 ts2phc offset | Given ts2phc is running, then the ts2phc offset must be continuously monitored and available as a metric |
| FUNC-TBC019-R4 | 10.1 phc2sys offset | Given phc2sys is running, then the phc2sys offset (CLOCK_REALTIME vs PHC) must be continuously monitored and available as a metric |
| FUNC-TBC019-R5 | 10.2 MaxInSpecOffset | Given a T-BC is in Holdover-In-Spec state, when any estimated DPLL phase offset exceeds `MaxInSpecOffset`, then the system must transition to Holdover-Out-Of-Spec |
| FUNC-TBC019-R6 | 10.2 LocalMaxHoldoverOffSet | Given a T-BC is in any Holdover state, when any estimated phase offset exceeds `LocalMaxHoldoverOffSet`, then the system must transition to Free-Run |
| FUNC-TBC019-R7 | 10.2 inSyncConditionThreshold | Given a T-BC has ValidSourceAvailable, when the worst-case offset is below `inSyncConditionThreshold` for `inSyncConditionTimes` consecutive samples, then the InSync condition is met |

### 14.20 Observability — Metrics Emission

### ID: FUNC-TBC020

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC020-R1 | 11.1.1 clock_state | Given a T-BC is running, then `openshift_ptp_clock_state` must be emitted per interface and per process (T-BC, ptp4l, ts2phc, dpll, phc2sys) with values 0=FREERUN, 1=LOCKED, 2=HOLDOVER |
| FUNC-TBC020-R2 | 11.1.1 clock_class | Given a T-BC is running, then `openshift_ptp_clock_class` must be emitted with the current clock class value |
| FUNC-TBC020-R3 | 11.1.2 offset_ns | Given a T-BC is running, then `openshift_ptp_offset_ns` must be emitted per measurement point (ptp4l, ts2phc, dpll, phc2sys, T-BC aggregate) |
| FUNC-TBC020-R4 | 11.1.3 DPLL status | Given a T-BC has a DPLL, then `openshift_ptp_phase_status`, `openshift_ptp_frequency_status`, and `openshift_ptp_pps_status` must be emitted with DPLL state values |
| FUNC-TBC020-R5 | 11.1.4 interface_role | Given a T-BC is running, then `openshift_ptp_interface_role` must be emitted per PTP interface with port state (PASSIVE/SLAVE/MASTER/FAULTY/UNKNOWN/LISTENING) |
| FUNC-TBC020-R6 | 11.1.5 process_status | Given a T-BC is running, then `openshift_ptp_process_status` (0=DOWN, 1=UP) and `openshift_ptp_process_restart_count` must be emitted per managed process |
| FUNC-TBC020-R7 | 11.1.6 threshold | Given a T-BC is configured, then `openshift_ptp_threshold` must expose the active values of `MaxInSpecOffset`, `LocalMaxHoldoverOffSet`, `inSyncConditionThreshold`, `inSyncConditionTimes`, `SysOffsetThreshold`, and `SysOffsetSamples` per profile |

### 14.21 Observability — Kubernetes Object Status and Events

### ID: FUNC-TBC021

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TBC021-R1 | 11.2.1 PtpConfig | Given a PtpConfig is applied, then `PtpConfig.status` must report any unrecoverable error including the node name, error condition, and diagnostic information |
| FUNC-TBC021-R2 | 11.2.2 NodePtpDevice | Given a node has PTP-capable hardware, then `NodePtpDevice.status` must report discovered PTP devices with hardware info and per-device hardware configuration status (including failed/success) |
| FUNC-TBC021-R3 | 11.2.3 HardwareConfig | Given a HardwareConfig is applied, then `HardwareConfig.status` must report matched nodes and the PTP profile associated with each match |
| FUNC-TBC021-R4 | 11.2.3 HardwareConfig | Given a HardwareConfig with behavior conditions is applied, then `HardwareConfig.status` should report the currently active behavior condition name per matched node (FFS) |
| FUNC-TBC021-R5 | 11.2.4 K8s Events | Given a configuration apply failure occurs, then a Kubernetes Event with reason `ConfigurationApplyFailed` or `HardwareConfigFailed` must be emitted on the PtpConfig resource |
| FUNC-TBC021-R6 | 11.2.4 K8s Events | Given a significant time irregularity is detected (e.g., time jumps backwards), then a Kubernetes Event with reason `TimeDiscontinuity` must be emitted |
| FUNC-TBC021-R7 | 11.2.4 K8s Events | Given leap second data is updated, then a Kubernetes Event with reason `LeapSecondUpdate` must be emitted |

---
## 15. Performance Requirements Specification
### 15.1 G.8273.2 Time Error Noise Generation

[**Source: Rec. ITU-T G.8273.2/Y.1368.2 (06/2023), section 7.1**](https://www.itu.int/rec/T-REC-G.8273.2/en)

The noise generation of a T-BC and a T-TSC represents the amount of noise produced at the output  
of the T-BC/T-TSC when there is an ideal input reference packet timing signal.  
Under normal, locked operating conditions, the time output of the T-BC and the T-TSC should be  
accurate to within the maximum absolute time error (TE) (max|TE|). This value includes all the noise  
components, i.e., the constant time error (cTE) and the dynamic time error (dTE) noise generation.

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| PERF-TBC001-R1 | 7.1 Time error noise generation, Table 7-1 – Maximum absolute time error (max|TE|) | Maximum absolute unfiltered time error must not exceed 30ns  |
| PERF-TBC001-R2 | 7.1 Time error noise generation, Table 7-2 – Maximum absolute time error low-pass filtered (max|TEL|) | Maximum absolute low-pass filtered time error must not exceed 5ns (NOTE1) |
| PERF-TBC001-R3 | 7.1.1 Constant time error noise generation (cTE), Table 7-3 – T-BC/T-TSC permissible range of constant time error | The cTE generation at the PTP outputs must not exceed ±10ns (NOTE2)  |
| PERF-TBC001-R4 | 7.1.2 Dynamic time error low-pass filtered noise generation (dTEL) Table 7-4 – Dynamic time error low-pass filtered noise generation (MTIE) for T-BC/T-TSC with constant temperature (within ±1 K) Table 7-5 – Dynamic time error low-pass filtered noise generation (TDEV) for T-BC/T-TSC with constant temperature (within ±1K) Table 7-6 – Dynamic time error low-pass filtered noise generation (MTIE) for T-BC/T-TSC with variable temperature | MTIE must not exceed 10 ns over the observation period of 10,000s TDEV must not exceed 2ns over the observation period of 1000s |
| PERF-TBC001-R5 | Table 7-7 – Dynamic time error high-pass filtered noise generation for T-BC/T-TSC | Dynamic time error high-pass filtered noise generation (dTEH) must not exceed 30ns over the observation period of 1000s |
| PERF-TBC001-R6 | 7.1.4.1 Relative constant time error noise generation (cTER)  | For clock class C, the relative constant time error (cTER) between any two phase and time output ports (1 PPS, PTP) of a T-BC must not exceed 12ns over a time period of 1000s |
| PERF-TBC001-R7 | 7.1.4.2 Relative dynamic time error low-pass filtered noise generation (dTERL)  | Relative dynamic time error low-pass filtered noise generation (MTIE) for T-BC with constant temperature (within ±1 K) must not exceed 14ns over the observation time period of 1000s |
| NOTE1. This requirement is specified for class D: “For class D, the maximum time error measured through a first-order low-pass filter with a bandwidth of 0.1 Hz, max|TEL|” NOTE2. Constant time error definition and the method to estimate constant time error are defined in \[ITU-T G.8260\]. For the purpose of testing the limit \[±10ns\], an estimate of constant time error should be obtained by averaging the time error sequence over 1 000 s.  |  |  |


### 15.2 G.8273.2 Noise Tolerance

[**Source: Rec. ITU-T G.8273.2/Y.1368.2 (06/2023), section 7.2**](https://www.itu.int/rec/T-REC-G.8273.2/en)

The noise tolerance of a T-BC/T-TSC indicates the minimum dynamic time error level at the input of the clock that should be accommodated while not causing any alarms, not causing the clock to switch reference, and not causing the clock to go into holdover.

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| PERF-TBC002-R1 | 7.2.2 Noise tolerance for clock class C | T-BC/T-TSC class C must tolerate dTE according to G.8271.1 network limit (clause 7.3) at the PTP input |
| PERF-TBC002-R2 | 7.2, general | Under the applied noise tolerance levels, the clock must not raise alarms, must not switch reference, and must not enter holdover |
| NOTE. There is no requirement related to cTE tolerance. | | |

### 15.3 G.8273.2 Noise Transfer

[**Source: Rec. ITU-T G.8273.2/Y.1368.2 (06/2023), section 7.3**](https://www.itu.int/rec/T-REC-G.8273.2/en)

The transfer characteristic of the T-BC/T-TSC determines its properties with regard to the transfer of time error from the PTP input interface to the PTP and 1 PPS output interfaces.

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| PERF-TBC003-R1 | 7.3.1 PTP to PTP and PTP to 1 PPS noise transfer | The bandwidth of the T-BC/T-TSC must not exceed 0.1 Hz and must not be less than 0.05 Hz |
| PERF-TBC003-R2 | 7.3.1 PTP to PTP and PTP to 1 PPS noise transfer | In the passband, the phase gain must be smaller than 0.1 dB (1.1%) |
| NOTE. Noise transfer applies to dynamic time noise only; there is no requirement related to cTE transfer. | | |

### 15.4 G.8273.2 Transient Response

[**Source: Rec. ITU-T G.8273.2/Y.1368.2 (06/2023), section 7.4.1**](https://www.itu.int/rec/T-REC-G.8273.2/en)

The transient response of the T-BC/T-TSC bounds the output signal excursion during rearrangement events in the PTP network.

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| PERF-TBC004-R1 | 7.4.1.2 Transient due to PTP network rearrangement (class C) | PTP/1PPS output transient response due to rearrangement of the PTP network is FFS in G.8273.2 |
| NOTE. PTP-only rearrangement transient response is FFS. | | |

### 15.5 G.8273.2 Holdover Performance

[**Source: Rec. ITU-T G.8273.2/Y.1368.2 (06/2023), section 7.4.2**](https://www.itu.int/rec/T-REC-G.8273.2/en)

The holdover performance requirements bound the maximum excursions in the PTP and 1 PPS output signal during loss of PTP input.

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| PERF-TBC005-R1 | 7.4.2.1, class C | Holdover performance (loss of all phase/time inputs) for class C is FFS in G.8273.2 |
| NOTE 1. G.8273.2 does not yet specify holdover MTIE masks for class C — this is explicitly FFS in section 7.4.2.1 and 7.4.2.3. | | |
| NOTE 2. Unassisted holdover (section 6 of this spec) operates without physical layer frequency — performance is bounded by the local OCXO drift model. | | |

## 16. References

- [Story & Acceptance Criteria](story.md)
- [Design Document: T-BC / T-TSC Unassisted Holdover](Design%20document_%20T-BC%20_%20T-TSC%20unassisted%20holdover.md)
- ITU-T G.8273.2 — Timing characteristics of T-BC and T-TSC
- ITU-T G.8275.1 — PTP telecom profile for phase/time with full timing support
- ITU-T G.8275 (2024) Amd.1 — Clock state mode definitions
- IEEE 1588-2019 — Precision Time Protocol
- ITU-T G.811 — Primary Reference Clock
- ITU-T G.812 — Slave clock types
- O-RAN O-Cloud Notification API v2
- CloudEvents Specification v1.0
