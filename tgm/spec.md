# Feature Specification: T-GM Testable Requirements

**Module**: PTP Operator Stack
**Author**: Jim Ramsay
**Created**: 2026-06-22
**Last Updated**: 2026-07-30
**Status**: Draft

---

## 1. Overview

Testable requirement specification for the Telecom Grandmaster (T-GM)
implementation. A T-GM is the primary time source in a telecom synchronisation
network: it acquires time from a GNSS receiver, disciplines its local
oscillator, and distributes phase, time, and frequency to downstream PTP
clients. Defines desired behaviour across all operating states, performance
metrics, state machines, and event contracts. Describes WHAT the system must
do and WHY — not HOW. Any delta between current code and this specification
is a bug or future backlog item.

---

## 2. Goals

- Establish a single, traceable source of truth for T-GM behaviour and performance
- Enable agentic SDLC tooling and automated gap analysis
- Define all behavioural outcomes independent of current code limitations
- Provide enumerated, individually-testable functional requirements with stable IDs

## 3. Non-Goals

- Duplicating upstream standards (IEEE 1588, [ITU-T G.8272](https://www.itu.int/rec/t-rec-g.8272/en), [G.8275](https://www.itu.int/rec/t-rec-g.8275/en))
- Prescribing implementation details (languages, frameworks, internal APIs)
- Defining T-BC (Boundary Clock) behaviour (covered in [T-BC spec](../tbc/spec.md))
- Defining T-TSC behaviour (see [T-TSC spec](../ttsc/spec.md))
- Defining vendor-specific hardware programming sequences

---

## 4. Glossary and Standards References

### 4.1 Abbreviations and Acronyms

- 1PPS - One Pulse Per Second
- BTCA - Best TimeTransmitter Clock Algorithm
- DNU - Do Not Use (SyncE quality level)
- DPLL - Digital Phase-Locked Loop
- EEC - Ethernet Equipment Clock
- ESMC - Ethernet Synchronisation Messaging Channel
- FFS - For Further Study
- GNSS - Global Navigation Satellite System
- NIC - Network Interface Card
- NMEA - National Marine Electronics Association
- OCXO - Oven-Controlled Crystal Oscillator
- PHC - PTP Hardware Clock
- PHY - Physical Layer (Ethernet)
- PRC - Primary Reference Clock
- PRTC - Primary Reference Time Clock
- QL - Quality Level (SyncE)
- RF - Radio Frequency
- SMA - SubMiniature version A (coaxial connector)
- SyncE - Synchronous Ethernet
- T-BC - Telecom Boundary Clock
- T-GM - Telecom Grandmaster
- T-TSC - Telecom Time Synchronous Clock
- TAI - International Atomic Time (Temps Atomique International)
- ToD - Time of Day
- TR - Time Receiver
- TT - Time Transmitter
- UTC - Coordinated Universal Time

### 4.2 Normative References

- [IEEE 1588-2008 (PTP v2)](https://standards.ieee.org/ieee/1588/4372/)
- [Recommendation ITU-T G.8272/Y.1367 (2018) Amendment 2 (11/22)](https://www.itu.int/rec/t-rec-g.8272/en): Timing characteristics of primary reference time clocks
- [Recommendation ITU-T G.8272.1/Y.1367.1 (01/24)](https://www.itu.int/rec/t-rec-g.8272.1/en): Timing characteristics of enhanced primary reference time clocks
- [Recommendation ITU-T G.8275/Y.1369 (01/24)](https://www.itu.int/rec/t-rec-g.8275/en): Architecture and requirements for packet-based time and phase distribution
- [Recommendation ITU-T G8275.1/Y.1369.1 (2022): Amendment 1 (01/24)](https://www.itu.int/rec/t-rec-g.8275.1/en): Precision time protocol telecom profile for phase/time synchronization with full timing support from the network - Amendment 1
- [Recommendation ITU-T G8275.2/Y.1369.2 (2022) Amendment 1 (01/24)](https://www.itu.int/rec/t-rec-g.8275.2/en): Precision time protocol telecom profile for time/phase synchronization with partial timing support from the network - Amendment 1
- [Recommendation ITU-T G.8262/Y.1362 (2018) Amendment 1 (03/2020)](https://www.itu.int/rec/t-rec-g.8262): Timing characteristics of synchronous equipment slave clock - Amendment 1
- [Recommendation ITU-T G.8262.1/Y.1362.1 (11/22)](https://www.itu.int/rec/T-REC-G.8262.1/en): Timing characteristics of enhanced synchronous equipment slave clock
- [Recommendation ITU-T G.8264/Y.1364 (2017) Amendment 2 (01/2024)](https://www.itu.int/rec/T-REC-G.8264): Distribution of timing information through packet networks - Amendment 1
- [Recommendation ITU-T G.811 (09/1997) Amendment 1 (04/16)](https://www.itu.int/rec/t-rec-g.811): Timing characteristics of primary reference clocks

> Note: These specific versions are the references in O-RAN.WG4.TS.CUS.0-R005-v21.00

### 4.3 Informative References

- O-RAN O-Cloud Notification API v2
- [O-RAN.WG4.TS.CUS.0-R005-v21.00 — O-RAN Control, User and Synchronization Plane Specification](https://orandownloadsweb.azurewebsites.net/specifications)
- CloudEvents v1.0 specification

---

## 5. System Scope and Context

### 5.1 Clock Types in Scope

- T-GM (Telecom Grandmaster): acquires time and frequency from a PRTC, distributes time and frequency via PTP, and optionally distributes frequency via SyncE.

For T-BC (Telecom Boundary Clock) requirements, see [T-BC spec](../tbc/spec.md).
For T-TSC (Telecom Time Synchronous Clock) requirements, see [T-TSC spec](../ttsc/spec.md).

### 5.2 Supported Topologies

- Single-NIC T-GM (one NIC with TimeTransmitter-only PTP ports)
- Multi-NIC T-GM (leader + follower NICs, all distributing PTP downstream)
- Optional SyncE frequency distribution on all configured ports

### 5.3 SyncE (Synchronous Ethernet) Role

SyncE provides physical layer frequency distribution from the frequency-locked oscillator. When configured:

- The DPLL drives frequency to the Ethernet PHY on all configured SyncE ports
- The ESMC protocol advertises the appropriate quality level (QL) based on the current clock state:
  - LOCKED: QL corresponding to PRTC (e.g., QL-PRC for Option 1 networks)
  - HOLDOVER: QL reflecting degraded traceability
  - FREERUN: QL-DNU (Do Not Use) or equivalent
- SyncE EEC lock state is reported through the observability interfaces
- SyncE is **optional** — the T-GM distributes phase/time via PTP regardless of SyncE configuration

### 5.4 Signal Chain

The T-GM acquires time, frequency and phase from a GNSS receiver acting as a built-in PRTC, disciplines a local clock chain (PHC and DPLL+OCXO complex), and distributes phase/time to downstream PTP clients via the PTP protocol.

**Signal flow in Locked state (single-NIC):**

```text
GNSS Constellation
       │
       │ RF signal
       ▼
┌──────────────┐
│ GNSS Receiver│─────────────────┐
│ (1PPS + ToD) │  ToD (NMEA)     │
└──────┬───────┘                 │
       │                         │
       │ 1PPS (phase/freq)       │
       ▼                         ▼
┌──────────┐    disciplines   ┌──────┐    HW timestamps  ┌──────────┐
│   DPLL   │───────────────►  │ PHC  │───────────────►   │  PTP TT  │
│  (OCXO)  │                  │      │                   │ TT       │
└──────────┘                  └──────┘                   └──────────┘
                                 │                            │
                                 │ ToD                        │
                                 ▼                            ▼
                          ┌────────────┐                 Downstream
                          │ CLOCK_REAL │                 PTP clients
                          └────────────┘
```

**Key components and their roles:**

| Component          | What it does                                                                                                                                                                                        | Disciplines                       | Disciplined by                                    |
| :----------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------- | :------------------------------------------------ |
| **GNSS Receiver**  | Receives RF signal from GNSS constellation; outputs 1PPS phase reference and ToD (NMEA/UBX)                                                                                                         | DPLL via 1PPS & PHC via ToD       | GNSS constellation                                |
| **DPLL**           | Digital PLL with OCXO. Locks to GNSS 1PPS in normal operation as a source of phase and frequency. Provides holdover stability when GNSS is lost                                                     | PHC                               | GNSS 1PPS                                         |
| **PHC**            | PTP Hardware Clock on the NIC. Frequency driven by the DPLL output; phase/ToD aligned by ts2phc comparing the GNSS 1PPS against the PHC output. Provides a source of HW timestamps for PTP messages | PTP (time base for HW timestamps) | DPLL (frequency) and GNSS 1PPS + ToD (phase/time) |
| **PTP**            | PTP protocol engine in grandmaster mode. Distributes Announce/Sync/Follow_Up on all TimeTransmitter-only ports                                                                                      | Downstream clients                | PHC (via HW timestamps)                           |
| **CLOCK_REALTIME** | The OS system clock                                                                                                                                                                                 | —                                 | PHC                                               |

**Signal flow in multi-NIC configuration (leader + follower):**

```text
 GNSS Constellation
       │
       │ RF signal
       ▼
┌──────────────┐
│ GNSS Receiver│───────────────────┐
│ (1PPS + ToD) │    ToD (NMEA)     │
└──────────────┘                   │
       │                           │
       │ 1PPS (phase/freq)         │
       ▼                           ▼
┌────────────┐    disciplines   ┌──────┐    HW timestamps  ┌──────────┐
│ DPLL (Ldr) │───────────────►  │ PHC1 │───────────────►   │ PTP TT   │──┐
│  (OCXO)    │                  │(Lead)│                   │          │  │
└────────────┘                  └──────┘                   └──────────┘  │
       │                           │                                     │
       │ physical signal           │ ToD                                 ▼
       │                           ▼                                Downstream
       │                    ┌────────────┐                          PTP clients
       │                    │ CLOCK_REAL │
       │                    └────────────┘
       │                          │ ToD
       ▼                          ▼
┌────────────┐    disciplines   ┌──────┐    HW timestamps  ┌──────────┐
│ DPLL (Flw) │───────────────►  │ PHC2 │───────────────►   │ PTP TT   │──┐
│  (OCXO)    │                  │(Flw) │                   │          │  │
└────────────┘                  └──────┘                   └──────────┘  │
                                                                         │
                                                                         ▼
                                                                    Downstream
                                                                    PTP clients
```

An optional parallel path distributes frequency via SyncE (DPLL → EEC → ESMC) to support downstream nodes that require both phase/time and frequency traceability.

**Signal flow in Holdover state:**

When the GNSS reference is lost, the DPLL free-runs on its internal OCXO. Unlike T-BC, the synchronisation direction does **not** reverse — the DPLL continues to discipline the PHC, but from a free-running oscillator rather than a GNSS-locked one:

```text
 GNSS Constellation
       │
       X RF signal

┌──────────────┐
│ GNSS Receiver│───────────────────┐
│ (1PPS + ToD) │    ToD (NMEA)     │
└──────────────┘                   │
       │                           │
       X 1PPS (phase/freq)         X

┌──────────┐    disciplines   ┌──────┐    HW timestamps  ┌──────────┐
│   DPLL   │───────────────►  │ PHC  │───────────────►   │ PTP TT   │
│  (OCXO   │                  │      │                   └──────────┘
│ holdover)│                  └──────┘                        │
└──────────┘                     │                            │
                                 │  ToD                       │
                                 │                            ▼
                                 ▼                       Downstream
                          ┌────────────┐                 PTP clients
                          │ CLOCK_REAL │
                          └────────────┘
```

- The DPLL locks its numerically controlled frequency to the last known good tuning value, driven by the highly stable, free-running OCXO.
- PTP continues distributing Sync/Announce to downstream, but with holdover clock class (7 or 140)
- The holdover performance is bounded by the local OCXO drift model (see §6.8)

### 5.6 Actors

- System Administrator: configures T-GM profiles and thresholds
- Downstream PTP nodes: consume Announce/Sync from TimeTransmitter ports
- Monitoring systems: subscribe to clock state events

### 5.7 [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Telecom Profile Parameters

The T-GM operates under the [ITU-T G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) telecom profile for phase/time synchronisation with full timing support from the network. Key profile constraints:

| Parameter              | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Value | Notes                                                                          |
| :--------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------- |
| domainNumber           | 24–43 (default 24)                                          | Per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table A.1            |
| network transport      | IEEE 802.3 (Layer 2 Ethernet multicast)                     | Mandatory; UDP/IP transport is not permitted                                   |
| delayMechanism         | Peer-to-peer (P2P)                                          | T-GM must respond to Pdelay_Req from directly connected peers with Pdelay_Resp |
| logSyncInterval        | −4 (16 per second)                                          | Default per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table A.5    |
| logAnnounceInterval    | −3 (8 per second)                                           | Default per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table A.5    |
| announceReceiptTimeout | 3                                                           | Number of announce intervals before timeout                                    |
| twoStepFlag            | TRUE or FALSE                                               | Both one-step and two-step operation are permitted by the profile              |

---

## 6. T-GM State Machine

### 6.1 State Transition Diagram

```text
                                   T2
  Init ──T1──►  ┌────────────────┐ ────► ┌────────────────┐
                │   Free-Run     │       │    Locked      │
                │  (class 248)   │ ◄──── │  (class 6)     │◄──────────┐
                └────────────────┘  T4   └────────────────┘           │
                   ▲          ▲                 │      ▲              │
                   │          │                 │ T3   │  T5          │
                   │          │                 ▼      │              │
                   │          │     ┌─────────────────────┐           │
                   │    T7    │     │ Holdover-In-Spec    │           │
                   │          └──── │    (class 7)        │           │
                   │                └─────────────────────┘           │
                   │                           │                      │
                   │                           │ T6                   │
                   │                           ▼                      │
                   │                ┌─────────────────────┐           │
                   │       T9       │Holdover-Out-Of-Spec │           │
                   └─────────────── │    (class 140)      │──────T8───┘
                                    └─────────────────────┘
```

### 6.2 State Definitions

The T-GM supports four clock states. State names and semantics are derived from [ITU-T G.8275](https://www.itu.int/rec/t-rec-g.8275/en) (2024) Amd.1, Section VIII.2. Clock class values are per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Section 6.4 Table 3.

**Free-Run** — The clock has never been synchronised to a time source, or is in the process of synchronising but has not yet reached acceptable accuracy. This state applies at initial startup (GNSS cold start), or when holdover has expired, or when a non-GNSS fault prevents time traceability. Clock class: **248**.

**Locked** — The clock is synchronised to its GNSS reference and all subsystems report acceptable accuracy. Per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en), a GNSS-traceable T-GM in the locked state is a PRTC. Clock class: **6**.

**Holdover-In-Spec** — The GNSS reference is lost but the clock is maintaining performance within the desired specification using holdover data from its OCXO. Per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Section 6.4, a T-GM in holdover with PRC-traceable frequency advertises clock class 7. Clock class: **7**.

**Holdover-Out-Of-Spec** — The GNSS reference is lost and the clock can no longer maintain performance within the desired specification. The holdover offset has exceeded `MaxInSpecOffset` or the holdover timer has exceeded `LocalHoldoverTimeout`. Clock class: **140**.

> **Note:** [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) defines additional clockClass values 150 and 160 for T-GM
> holdover (out-of-spec) when the frequency reference quality is below PRC.
> clockClass 150 corresponds to SSU-A / ST2-level frequency traceability;
> clockClass 160 corresponds to SSU-B / ST3E-level frequency traceability.
> This specification uses clockClass 140 exclusively because the T-GM derives
> its frequency from a GNSS source (which meets PRTC / PRC quality per
> [ITU-T G.8272](https://www.itu.int/rec/t-rec-g.8272/en)). Deployments with lower-quality frequency references should
> map to the appropriate clockClass per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Section 6.4.

> **Note:** Setting `timeTraceable=TRUE` and `frequencyTraceable=TRUE` during
> Holdover-In-Spec (clockClass 7) is an implementation choice. Strictly per
> IEEE 1588-2019, `timeTraceable` indicates the timescale is traceable to a
> primary reference — which is no longer the case once the GNSS reference is
> lost. However, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) clockClass 7 implies the clock was previously
> traceable and is still operating within specification. Many telecom
> deployments keep both flags TRUE during in-spec holdover to avoid
> unnecessary downstream BTCA recalculations. This specification follows that
> convention. Deployments with stricter traceability requirements may choose
> to set both flags to FALSE upon entering any holdover state.

> SyncE additional states and behaviors depending on the overall clock state, but these are not yet defined in this document.

### 6.3 Composite State Evaluation

The T-GM clock state is derived from the combined states of three independent subsystems. The composite state is the _lowest-quality_ (most degraded) state among the active subsystems.

| Subsystem        | Inputs                                                                                           | States reported                                       |
| :--------------- | :----------------------------------------------------------------------------------------------- | :---------------------------------------------------- |
| GNSS monitor     | Satellite 3D fix status, anti-spoofing state, GNSS offset, and time accuracy estimation (`tAcc`) | LOCKED, FREERUN                                       |
| DPLL             | Frequency circuit state, phase circuit state, and DPLL phase offset                              | LOCKED (implies Holdover Acquired), HOLDOVER, FREERUN |
| PHC synchroniser | PHC-to-reference offset against threshold, and `ts2phc` servo state                              | LOCKED, HOLDOVER, UNLOCKED                            |

**Composite state evaluation rules:**

1. If all subsystems that report a state report LOCKED, the composite state shall be LOCKED.
2. If the DPLL reports HOLDOVER (because GNSS reference is lost), the composite state shall be HOLDOVER. The system shall not report FREERUN solely because the GNSS monitor reports FREERUN due to source loss — it shall wait for the DPLL to transition out of holdover before reporting FREERUN.
3. HOLDOVER shall be subdivided into in-spec and out-of-spec based on the holdover time error budget (see §6.8).
4. If any subsystem reports FREERUN and no subsystem reports HOLDOVER, the composite state shall be FREERUN.

### 6.4 State Transition Conditions

The following conditions govern transitions between states. Each condition is evaluated continuously by the supervision software.

| Condition Name                | Definition                                                                                                                                                                                                                  | Parameters                                         |
| :---------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------- |
| **GNSSAvailable**             | GNSS receiver has a valid 3D fix (fix status ≥ 3) and GNSS clock offset is within the configured threshold                                                                                                                  | `gnssOffsetThreshold`                              |
| **GNSSLost**                  | GNSS receiver has lost valid fix (fix status < 3), detected spoofing or jamming, or NMEA data is unavailable                                                                                                                | —                                                  |
| **AllLocked**                 | All subsystems report LOCKED simultaneously: GNSS fix valid, GNSS offset within threshold, DPLL frequency and phase status both LOCKED, DPLL phase offset within in-spec threshold, PHC locked, PHC offset within threshold | `inSyncConditionThreshold`, `inSyncConditionTimes` |
| **HoldoverDataValid**         | DPLL is in HOLDOVER or LOCKED_HO_ACQ state — the OCXO has sufficient holdover data to maintain timing                                                                                                                       | —                                                  |
| **Offset-Above-InSpecOffset** | Measured or estimated phase offset exceeds `MaxInSpecOffset`, or elapsed holdover time exceeds `LocalHoldoverTimeout`                                                                                                       | `MaxInSpecOffset`, `LocalHoldoverTimeout`          |
| **Holdover-To-FreeRun**       | Measured or estimated phase offset exceeds `LocalMaxHoldoverOffSet`                                                                                                                                                         | `LocalMaxHoldoverOffSet`                           |
| **NonGNSSFault**              | A non-GNSS-loss fault prevents time traceability (e.g., PHC synchroniser enters FREERUN while DPLL is LOCKED, or a process crash is detected)                                                                               | —                                                  |

### 6.5 State Transition Table

| #   | From State               | To State             | Guard Condition                        | Actions                                                      |
| :-- | :----------------------- | :------------------- | :------------------------------------- | :----------------------------------------------------------- |
| T1  | **Init**                 | Free-Run             | Always (initialisation complete)       | Determine clock type from configuration; start processes     |
| T2  | **Free-Run**             | Locked               | AllLocked condition met                | Update announce to GNSS-locked parameters (class 6)          |
| T3  | **Locked**               | Holdover-In-Spec     | GNSSLost AND HoldoverDataValid         | Start holdover timer; update announce to local (class 7)     |
| T4  | **Locked**               | Free-Run             | NonGNSSFault condition                 | Update announce to local (class 248)                         |
| T5  | **Holdover-In-Spec**     | Locked               | AllLocked condition met                | Stop holdover timer; restore GNSS-locked announce parameters |
| T6  | **Holdover-In-Spec**     | Holdover-Out-Of-Spec | Offset-Above-InSpecOffset AND GNSSLost | Update announce (class 140, timeTraceable=FALSE)             |
| T7  | **Holdover-In-Spec**     | Free-Run             | Holdover-To-FreeRun condition          | Update announce (class 248)                                  |
| T8  | **Holdover-Out-Of-Spec** | Locked               | AllLocked condition met                | Stop holdover timer; restore GNSS-locked announce parameters |
| T9  | **Holdover-Out-Of-Spec** | Free-Run             | Holdover-To-FreeRun condition          | Update announce (class 248)                                  |

### 6.6 Initialisation Behaviour

The initialisation transition occurs when the supervision software applies the node configuration. During this transition:

1. The current PTP clock type is determined from the configuration resource.
2. Processes are started in the required order with generated configurations.
3. DPLL pins are configured to the initial state for the determined clock type.
4. GNSS receiver hardware is configured
5. The system enters the **Free-Run** state unconditionally.

Initialisation always ends in the Free-Run state regardless of whether GNSS is already available. The system must progress through the AllLocked condition to reach Locked.

### 6.7 General Timing Constraints

| Constraint                      | Value / Source                                                                                                                                        |
| :------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| State transition event latency  | Events must be published within **1 second** of the state transition (required for O-RAN fault notification reactivity)                               |
| Announce message update latency | Announce content must reflect the new state within **1 Announce interval** after transition (prevents downstream from syncing to degraded clockClass) |
| Convergence time for lock       | FFS — depends on GNSS acquisition time, servo algorithm, and DPLL lock time                                                                           |
| AllLocked filter window         | `inSyncConditionTimes` consecutive samples below `inSyncConditionThreshold`                                                                           |

### 6.8 Holdover Model and Configuration Parameters

#### 6.8.1 The industry model for clock time error

The industry accepted model for clock time error ΔT(t) as a function of time is expressed as:
ΔT(t) = T₀ + (Δf/f)·t + ½·A·t² + ε(t), where

- T₀ : Initial Time Offset: The phase error present at the exact moment the reference signal was lost. If the synchronisation was perfect, this is ideally zero.

- Δf/f: Initial Frequency Offset: The difference between the oscillator's actual frequency and the target frequency at the start of holdover. This component causes a linear increase in time error.

- ½·A·t²: Frequency Drift / Ageing: The rate at which the frequency changes over time due to the physical changes in the oscillator crystal. This component causes a quadratic (squared) increase in time error.

- ε(t): Phase Noise / Random Walk: Stochastic (random) noise that is unpredictable but contributes to the overall jitter and wander. This component is unpredictable due to its stochastic nature.

#### 6.8.2 The simplified holdover model

For practical application, where the stochastic component and initial frequency offset are difficult to estimate, a simplified model is sufficient:

ΔT(t) = T₀ + S·t + ½·A·t²

The initial frequency offset Δf/f is absorbed into the linear term S, which represents the worst-case drift rate specified by the oscillator manufacturer.

The maximum values for ΔT and t are provided by the oscillator manufacturer as the maximum holdover time and the maximum offset accumulated during that time. S is the slope of the linear component (maximum offset drift rate). A is the oscillator ageing component, which becomes significant at longer holdover durations.

This model is valid when clocks are sufficiently well synchronised so that the initial frequency offset falls within the bounds assumed by the manufacturer. When this syntonisation criterion is met, the system is considered **holdover-capable** at the source-lost event.

#### 6.8.3 Oscillator on-time requirement

Oscillator holdover performance is guaranteed only after sufficient power-on time. The required warm-up time ranges from 10 to 100 hours depending on the preceding power-off duration. The system should not rely on holdover performance guarantees until the oscillator has been powered on for the manufacturer-specified duration.

#### 6.8.4 Configuration Parameters

The user-facing API for configuring the holdover model consists of:

| Parameter                | Description                                                                                              |
| :----------------------- | :------------------------------------------------------------------------------------------------------- |
| `MaxInSpecOffset`        | Maximum phase offset (ns) that qualifies as in-spec holdover; defines the in-spec / out-of-spec boundary |
| `LocalMaxHoldoverOffSet` | Maximum phase offset (ns) before holdover is declared expired and the system transitions to Free-Run     |
| `LocalHoldoverTimeout`   | Maximum time (seconds) the clock stays in holdover before declaring out-of-spec                          |

The holdover duration may be derived from the oscillator model (where the hardware provides the model parameters) or from the user-configurable `LocalHoldoverTimeout`.

---

## 7. Synchronisation Direction

Unlike the T-BC (which reverses synchronisation direction between Locked and Holdover states), the T-GM maintains a **unidirectional** signal flow at all times: GNSS → DPLL → PHC → PTP protocol unit.

In Holdover, the GNSS input is lost but the DPLL continues to discipline the PHC from its free-running OCXO. No pin reconfiguration or direction reversal is required. The DPLL transitions from LOCKED to HOLDOVER/LOCKED_HO_ACQ internally.

For multi-NIC configurations, the leader DPLL distributes 1PPS to the follower DPLL via a physical connection such as an SMA cable. This path is also unidirectional and does not change during holdover.

---

## 8. Announce Message Behaviour

All PTP ports on the T-GM operate in the TimeTransmitter role. Announce messages are transmitted on all configured TimeTransmitter-only ports. The content of these messages must reflect the current T-GM state. In the Locked state, the T-GM advertises itself as a GNSS-traceable PRTC. In all other states, the T-GM advertises degraded parameters.

### 8.1 Locked State Announce Content

| Information Element                             | Content                                                                              |
| :---------------------------------------------- | :----------------------------------------------------------------------------------- |
| sourcePortIdentity                              | Local clockId of the T-GM + PortNumber                                               |
| leap61                                          | Per GNSS leap-second announcement                                                    |
| leap59                                          | Per GNSS leap-second announcement                                                    |
| currentUtcOffsetValid                           | TRUE                                                                                 |
| ptpTimescale                                    | TRUE                                                                                 |
| timeTraceable                                   | **TRUE**                                                                             |
| frequencyTraceable                              | **TRUE**                                                                             |
| currentUtcOffset                                | Current TAI−UTC offset                                                               |
| grandmasterPriority1                            | Configured priority1 (128 per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en)) |
| grandmasterClockQuality.clockClass              | **6**                                                                                |
| grandmasterClockQuality.clockAccuracy           | **0x21** (100 ns)                                                                    |
| grandmasterClockQuality.offsetScaledLogVariance | **0x4E5D**                                                                           |
| grandmasterPriority2                            | Configured priority2 (from ptp4lConf)                                                |
| grandmasterIdentity                             | Local clockId                                                                        |
| stepsRemoved                                    | 0                                                                                    |
| timeSource                                      | **GNSS (0x20)**                                                                      |
| synchronizationUncertain                        | Not supported in this version                                                        |

> **Note:** The clockAccuracy value 0x21 (100 ns) is correct for both PRTC-A
> (100 ns max |TE| per [G.8272](https://www.itu.int/rec/t-rec-g.8272/en)) and PRTC-B (40 ns max |TE| per [G.8272](https://www.itu.int/rec/t-rec-g.8272/en)).
> Per IEEE 1588-2019 Table 3, clockAccuracy 0x21 means "accurate to within
> 100 ns" and 0x20 means "accurate to within 25 ns". Since PRTC-B accuracy
> of 40 ns does not meet the 0x20 threshold, 0x21 is the appropriate value
> for both PRTC classes. The `stepsRemoved` value of 0 confirms the T-GM is
> the grandmaster; downstream T-BCs increment this value. FFS: the clock class
> could be configurable or auto-configured if the internal accuracy warrants a
> better accuracy declaration.

### 8.2 Holdover-In-Spec State Announce Content

The T-GM announces holdover parameters. Time remains traceable (the clock is still within specification).

| Information Element                             | Content                                                                              |
| :---------------------------------------------- | :----------------------------------------------------------------------------------- |
| sourcePortIdentity                              | Local clockId of the T-GM + PortNumber                                               |
| leap61                                          | FALSE, or advertise last known leap second event if expected                         |
| leap59                                          | FALSE, or advertise last known leap second event if expected                         |
| currentUtcOffsetValid                           | TRUE                                                                                 |
| ptpTimescale                                    | TRUE                                                                                 |
| timeTraceable                                   | **TRUE**                                                                             |
| frequencyTraceable                              | **TRUE**                                                                             |
| currentUtcOffset                                | Current TAI−UTC offset                                                               |
| grandmasterPriority1                            | Configured priority1 (128 per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en)) |
| grandmasterClockQuality.clockClass              | **7**                                                                                |
| grandmasterClockQuality.clockAccuracy           | Unknown (0xFE)                                                                       |
| grandmasterClockQuality.offsetScaledLogVariance | 0xFFFF                                                                               |
| grandmasterPriority2                            | Configured priority2 (from ptp4lConf)                                                |
| grandmasterIdentity                             | Local clockId                                                                        |
| stepsRemoved                                    | 0                                                                                    |
| timeSource                                      | INT_OSC (0xA0)                                                                       |
| synchronizationUncertain                        | Not supported in this version                                                        |

### 8.3 Holdover-Out-Of-Spec State Announce Content

The T-GM announces degraded holdover. Time is no longer traceable.

| Information Element                             | Content                                                                              |
| :---------------------------------------------- | :----------------------------------------------------------------------------------- |
| sourcePortIdentity                              | Local clockId of the T-GM + PortNumber                                               |
| leap61                                          | FALSE, or advertise last known leap second event if expected                         |
| leap59                                          | FALSE, or advertise last known leap second event if expected                         |
| currentUtcOffsetValid                           | TRUE                                                                                 |
| ptpTimescale                                    | TRUE                                                                                 |
| timeTraceable                                   | **FALSE**                                                                            |
| frequencyTraceable                              | **TRUE**                                                                             |
| currentUtcOffset                                | Current TAI−UTC offset                                                               |
| grandmasterPriority1                            | Configured priority1 (128 per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en)) |
| grandmasterClockQuality.clockClass              | **140**                                                                              |
| grandmasterClockQuality.clockAccuracy           | Unknown (0xFE)                                                                       |
| grandmasterClockQuality.offsetScaledLogVariance | 0xFFFF                                                                               |
| grandmasterPriority2                            | Configured priority2 (from ptp4lConf)                                                |
| grandmasterIdentity                             | Local clockId                                                                        |
| stepsRemoved                                    | 0                                                                                    |
| timeSource                                      | INT_OSC (0xA0)                                                                       |
| synchronizationUncertain                        | Not supported in this version                                                        |

### 8.4 Free-Run State Announce Content

| Information Element                             | Content                                                                              |
| :---------------------------------------------- | :----------------------------------------------------------------------------------- |
| sourcePortIdentity                              | Local clockId of the T-GM + PortNumber                                               |
| leap61                                          | FALSE                                                                                |
| leap59                                          | FALSE                                                                                |
| currentUtcOffsetValid                           | TRUE                                                                                 |
| ptpTimescale                                    | TRUE                                                                                 |
| timeTraceable                                   | **FALSE**                                                                            |
| frequencyTraceable                              | **FALSE**                                                                            |
| currentUtcOffset                                | Current TAI−UTC offset                                                               |
| grandmasterPriority1                            | Configured priority1 (128 per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en)) |
| grandmasterClockQuality.clockClass              | **248**                                                                              |
| grandmasterClockQuality.clockAccuracy           | Unknown (0xFE)                                                                       |
| grandmasterClockQuality.offsetScaledLogVariance | 0xFFFF                                                                               |
| grandmasterPriority2                            | Configured priority2 (from ptp4lConf)                                                |
| grandmasterIdentity                             | Local clockId                                                                        |
| stepsRemoved                                    | 0                                                                                    |
| timeSource                                      | INT_OSC (0xA0)                                                                       |
| synchronizationUncertain                        | Not supported in this version                                                        |

### 8.5 Key Differences Across States

The required clockClass values and their corresponding accuracies are derived directly from [ITU-T G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) (2022) Table 2 (Clock class values).

| Field                   | Locked        | HO-In-Spec     | HO-Out-Of-Spec | Free-Run       |
| :---------------------- | :------------ | :------------- | :------------- | :------------- |
| clockClass              | 6             | 7              | 140            | 248            |
| clockAccuracy           | 0x21 (100 ns) | Unknown        | Unknown        | Unknown        |
| offsetScaledLogVariance | 0x4E5D        | 0xFFFF         | 0xFFFF         | 0xFFFF         |
| timeTraceable           | TRUE          | TRUE           | FALSE          | FALSE          |
| frequencyTraceable      | TRUE          | TRUE           | TRUE           | FALSE          |
| timeSource              | GNSS (0x20)   | INT_OSC (0xA0) | INT_OSC (0xA0) | INT_OSC (0xA0) |
| currentUtcOffsetValid   | TRUE          | TRUE           | TRUE           | TRUE           |

### 8.6 Announce Update Timing

- Announce content must be updated within one Announce interval of a state transition
- The `currentUtcOffset` field shall contain the correct TAI−UTC offset at all times; the system shall track leap-second announcements from the GNSS receiver and update the offset accordingly

---

## 9. Process Orchestration

> Note: This is informational, and may be more in the realm of implementation notes than external requirements.

### 9.1 Process Inventory and Roles

| Process      | Role                                                                                        | Instances per T-GM |
| :----------- | :------------------------------------------------------------------------------------------ | :----------------- |
| ts2phc       | PHC synchroniser — disciplines the PHC to the GNSS-derived 1PPS reference                   | 1                  |
| ptp4l        | PTP protocol engine in grandmaster mode — Announce/Sync/Follow_Up distribution on all ports | 1                  |
| phc2sys      | System clock synchroniser — disciplines `CLOCK_REALTIME` from the PHC                       | 1 (if configured)  |
| GNSS monitor | Monitors GNSS receiver fix status, offset, and ToD                                          | 1                  |
| DPLL monitor | Monitors DPLL lock state, phase offset, and frequency status via netlink                    | 1                  |
| synce4l      | SyncE engine — ESMC protocol and frequency distribution                                     | 1 (if configured)  |

### 9.2 T-GM Process Startup Order

The process startup order is clock-type-specific. This section defines the order for T-GM. For T-BC startup order, see the [T-BC specification](../tbc/spec.md).

| Step | Process          | Precondition                          | Rationale                                                                       |
| :--- | :--------------- | :------------------------------------ | :------------------------------------------------------------------------------ |
| 1    | **GNSS monitor** | None                                  | Must begin monitoring GNSS fix status before other subsystems can evaluate lock |
| 2    | **DPLL monitor** | None                                  | Must begin monitoring DPLL state; can start in parallel with GNSS monitor       |
| 3    | **ts2phc**       | GNSS monitor and DPLL monitor running | PHC discipline requires GNSS 1PPS and ToD; DPLL state must be observable        |
| 4    | **ptp4l**        | ts2phc running                        | PTP distribution requires a disciplined PHC                                     |
| 5    | **phc2sys**      | ts2phc reports LOCKED (s2)            | Must not discipline `CLOCK_REALTIME` from an un-synchronised PHC                |
| 6    | **synce4l**      | DPLL monitor running                  | SyncE requires knowledge of DPLL/frequency lock state                           |

### 9.3 Process Lifecycle

- **Configuration generation**: each process config is rendered from the PtpConfig profile and written to a temporary file before process startup
- **Scheduling policy**: all timing-critical processes (ptp4l, ts2phc, phc2sys) must run under `SCHED_FIFO` with configurable real-time priority
- **Graceful shutdown**: on configuration change or node shutdown, processes must be stopped and restarted in the correct startup order with the new configuration
- **Automatic restart**: if a process exits unexpectedly, it must be restarted automatically. The restart may bypass the startup order (do not kill and restart other processes to adhere to the startup order)

### 9.4 Process State Monitoring

- Process state is extracted by parsing stdout/stderr output lines for known patterns (servo state codes s0/s1/s2, offset values, GNSS fix status)
- Each process is continuously monitored for health; failure detection triggers automatic restart (see §9.3)

### 9.5 T-GM Behaviour on Process Failure

Every killed or crashed process is automatically restarted. The system behaviour during the outage depends on which process failed and its impact on the synchronisation chain. A configurable `processDowntimeThresholds` structure (in seconds, per process) defines the maximum acceptable downtime before state-change events are emitted. The default threshold (5 seconds) balances transient process restarts against allowing the downstream network to drift significantly out of limits.

If the process restarts within its configured threshold, state-change events are suppressed. If the downtime exceeds the threshold, the appropriate state-change events must be emitted.

| Process Failed   | Impact on Synchronisation                                                   | Desired Behaviour                                                                                                              | Event Behaviour                                                                |
| :--------------- | :-------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------- |
| **ts2phc**       | PHC is no longer disciplined to GNSS 1PPS; DPLL continues on last frequency | Enter holdover for the duration of the outage; holdover mechanism handles detection and recovery                               | Emit clock class change events if downtime exceeds threshold                   |
| **ptp4l**        | PTP protocol timeout on downstream clients; no Announce/Sync sent           | Downstream clients lose synchronisation; clock state is unaffected                                                             | Emit state-change event if downtime exceeds threshold                          |
| **phc2sys**      | System clock (`CLOCK_REALTIME`) drifts from PHC                             | Restart silently; no impact on PTP distribution or clock state                                                                 | Suppress E3 (os-clock-sync-state-change) toggle if restart is within threshold |
| **GNSS monitor** | Loss of GNSS state visibility; DPLL continues on last lock                  | Clock state manager loses GNSS input; may trigger holdover if GNSS state is required for composite evaluation                  | Emit state-change events per composite state rules                             |
| **DPLL monitor** | Loss of DPLL state visibility                                               | Clock state manager loses DPLL input; shall assume worst-case (FREERUN) if DPLL state is unknown for longer than the threshold | Emit state-change events per composite state rules                             |
| **synce4l**      | SyncE frequency distribution stops                                          | No impact on PTP phase distribution; SyncE QL shall degrade to DNU on downstream peers                                         | No PTP-related events                                                          |

---

## 10. Clock Components Monitoring

### 10.1 Offset Measurement Points

- GNSS receiver clock offset (GNSS-reported time vs receiver internal clock)
- DPLL phase offset (DPLL vs reference input, reported in picoseconds via netlink)
- DPLL fractional frequency offset (FFS)
- ts2phc offset (PHC vs GNSS 1PPS reference)
- phc2sys offset (system clock vs PHC)

### 10.2 GNSS Status Monitoring

- Satellite fix status (3D fix = valid, < 3 = invalid)
- GNSS clock offset vs configured threshold
- Source loss detection (fix status < 3 or NMEA data loss) — propagated to clock state manager within 1 second
- Source spoofing detection (hardware dependent)
- Source jamming detection (hardware dependent)

### 10.3 DPLL Status Monitoring

- DPLL lock state: FREERUN, LOCKED, LOCKED_HO_ACQ, HOLDOVER (via Linux DPLL netlink)
- Frequency status and phase status monitored independently; both must report LOCKED for DPLL subsystem LOCKED
- DPLL phase offset compared against configured in-spec threshold

### 10.4 PHC Status Monitoring

- Servo state: s0 (FREERUN/unlocked), s1 (FREERUN/initial calibration), s2 (LOCKED)
- PHC-to-reference offset at each synchronisation cycle
- Configurable offset threshold (`servo_offset_threshold`) and consecutive sample count (`servo_num_offset_values`)

> **Note:** To support accurate holdover budget calculations and state transitions, all clock components (GNSS, DPLL, PHC) must be polled or measured at least once per second (1 Hz).

---

## 11. Observability and Diagnostics

### 11.1 Prometheus Metrics

The following metrics must be exposed for T-GM monitoring. All metrics use the `openshift_ptp_` prefix.

#### 11.1.1 Clock State and Class

| Metric                      | Type  | Labels                      | Values                          | Description                                                                                                             |
| :-------------------------- | :---- | :-------------------------- | :------------------------------ | :---------------------------------------------------------------------------------------------------------------------- |
| `openshift_ptp_clock_state` | gauge | `iface`, `node`, `process`  | 0=FREERUN, 1=LOCKED, 2=HOLDOVER | Clock state per interface and process. Process values: `T-GM` (composite), `gnss`, `dpll`, `ts2phc`, `ptp4l`, `phc2sys` |
| `openshift_ptp_clock_class` | gauge | `config`, `node`, `process` | 6, 7, 140, 248                  | Current clock class. 6=GNSS-locked, 7=Holdover in-spec, 140=Holdover out-of-spec, 248=Free-Run                          |

#### 11.1.2 Offset and Delay

| Metric                                  | Type  | Labels                             | Values      | Description                                                                                                 |
| :-------------------------------------- | :---- | :--------------------------------- | :---------- | :---------------------------------------------------------------------------------------------------------- |
| `openshift_ptp_offset_ns`               | gauge | `from`, `iface`, `node`, `process` | nanoseconds | Current phase offset. `from` values: `gnss`, `dpll`, `master` (ts2phc), `phc` (phc2sys), `T-GM` (composite) |
| `openshift_ptp_delay_ns`                | gauge | `from`, `iface`, `node`, `process` | nanoseconds | Path delay measurement (0 for T-GM's PHC synchroniser)                                                      |
| `openshift_ptp_frequency_adjustment_ns` | gauge | `from`, `iface`, `node`, `process` | ppb         | Frequency adjustment applied by servo                                                                       |

#### 11.1.3 DPLL Status

| Metric                           | Type  | Labels                             | Values                                                                  | Description                |
| :------------------------------- | :---- | :--------------------------------- | :---------------------------------------------------------------------- | :------------------------- |
| `openshift_ptp_phase_status`     | gauge | `from`, `iface`, `node`, `process` | -1=UNKNOWN, 0=INVALID, 1=FREERUN, 2=LOCKED, 3=LOCKED_HO_ACQ, 4=HOLDOVER | DPLL phase lock status     |
| `openshift_ptp_frequency_status` | gauge | `from`, `iface`, `node`, `process` | -1=UNKNOWN, 0=INVALID, 1=FREERUN, 2=LOCKED, 3=LOCKED_HO_ACQ, 4=HOLDOVER | DPLL frequency lock status |

#### 11.1.4 Interface Role

| Metric                         | Type  | Labels                     | Values                                                               | Description                                                                         |
| :----------------------------- | :---- | :------------------------- | :------------------------------------------------------------------- | :---------------------------------------------------------------------------------- |
| `openshift_ptp_interface_role` | gauge | `iface`, `node`, `process` | 0=PASSIVE, 1=TR/SLAVE, 2=TT/MASTER, 3=FAULTY, 4=UNKNOWN, 5=LISTENING | PTP port state per interface (from ptp4l). All T-GM ports must report TT/MASTER (2) |

#### 11.1.5 Process Health

| Metric                                | Type    | Labels                      | Values          | Description                                                                                                           |
| :------------------------------------ | :------ | :-------------------------- | :-------------- | :-------------------------------------------------------------------------------------------------------------------- |
| `openshift_ptp_process_status`        | gauge   | `config`, `node`, `process` | 0=DOWN, 1=UP    | Current process liveness                                                                                              |
| `openshift_ptp_process_restart_count` | counter | `config`, `node`, `process` | monotonic count | Cumulative process restart count since daemon start. The count must not include restarts due to configuration changes |

### 11.2 Logging Requirements

- State transition log entries (previous state and new state)
- Offset threshold crossing warnings
- Announce attribute update confirmations
- GNSS fix status changes
- DPLL lock state changes
- T-GM composite state logged as a dedicated `T-GM-STATUS` message on every composite state change

---

## 12. CloudEvents and Notification Behaviour

### 12.1 Event Types

- PTP Lock State Change: LOCKED / FREERUN / HOLDOVER with offset
- PTP Clock Class Change: clock class value transitions (6, 7, 140, 248)
- OS Clock Sync State: system clock synchronisation status
- GNSS State Change: GPS receiver fix status
- Overall Sync State: aggregated synchronisation assessment

### 12.2 Event Generation Rules

- Edge-triggered: events published only on state transitions, not periodically
- Required data payload per event type (offset, state, interface, clock class)
- Event ordering and causality guarantees

### 12.3 Event Delivery Contract

- CloudEvents v1.0 envelope format
- O-RAN O-Cloud Notification API v2 compliance
- Resource address hierarchy (/sync/ptp-status/lock-state, etc.)
- Subscription model (endpoint URI + resource address filter)

### 12.4 Event Data Models

- PTP Lock State event payload structure
- Clock Class event payload structure
- OS Clock Sync event payload structure
- GNSS State event payload structure
- Overall Sync State event payload structure

### 12.5 Events Per State Transition

The following matrix defines which O-RAN O-Cloud Notification API v4.00 events must be generated for each T-GM state transition. Event types and resource addresses are per O-RAN.WG6.O-Cloud Notification API v04.00, section 7.2.3.

**O-RAN event types relevant to T-GM:**

| #   | Event Type                                            | Resource Address                        | value_type                            | O-RAN Section |
| :-- | :---------------------------------------------------- | :-------------------------------------- | :------------------------------------ | :------------ |
| E1  | `event.sync.ptp-status.ptp-state-change`              | `/sync/ptp-status/lock-state`           | enumeration (LOCKED/HOLDOVER/FREERUN) | 7.2.3.3       |
| E2  | `event.sync.ptp-status.ptp-clock-class-change`        | `/sync/ptp-status/clock-class`          | metric (Uint8)                        | 7.2.3.10      |
| E3  | `event.sync.sync-status.os-clock-sync-state-change`   | `/sync/sync-status/os-clock-sync-state` | enumeration (LOCKED/HOLDOVER/FREERUN) | 7.2.3.8       |
| E4  | `event.sync.sync-status.synchronization-state-change` | `/sync/sync-status/sync-state`          | enumeration (LOCKED/HOLDOVER/FREERUN) | 7.2.3.1       |
| E5  | `event.sync.gnss-status.gnss-state-change`            | `/sync/gnss-status/gnss-state`          | enumeration (LOCKED/FREERUN)          | 7.2.3.5       |

**Event generation matrix per T-GM state transition:**

| Transition (from §6.5)          | T-GM State Change         | E1: PTP Lock State   | E2: Clock Class | E4: Overall Sync State | E5: GNSS State |
| :------------------------------ | :------------------------ | :------------------- | :-------------- | :--------------------- | :------------- |
| T1: Init → Free-Run             | Initial state             | FREERUN              | 248             | FREERUN                | FREERUN        |
| T2: Free-Run → Locked           | GNSS acquired, all locked | LOCKED               | 6               | worst-of(LOCKED, E3)   | LOCKED         |
| T3: Locked → Holdover-In-Spec   | GNSS lost                 | HOLDOVER             | 7               | worst-of(HOLDOVER, E3) | FREERUN        |
| T4: Locked → Free-Run           | Non-GNSS fault            | FREERUN              | 248             | FREERUN                | — (unchanged)  |
| T5: HO-In-Spec → Locked         | Re-lock                   | LOCKED               | 6               | worst-of(LOCKED, E3)   | LOCKED         |
| T6: HO-In-Spec → HO-Out-Of-Spec | Offset > MaxInSpec        | — (remains HOLDOVER) | 140             | — (unchanged)          | — (unchanged)  |
| T7: HO-In-Spec → Free-Run       | Holdover-To-FreeRun       | FREERUN              | 248             | FREERUN                | — (unchanged)  |
| T8: HO-Out-Of-Spec → Locked     | Re-lock                   | LOCKED               | 6               | worst-of(LOCKED, E3)   | LOCKED         |
| T9: HO-Out-Of-Spec → Free-Run   | Holdover-To-FreeRun       | FREERUN              | 248             | FREERUN                | — (unchanged)  |

**Notes:**

- "—" means the event is not generated for this transition (no state change for that event type)
- E2 (Clock Class) fires on every clock class value change, including the sub-state transition T6 (7→140) where E1 does not fire (PTP state remains HOLDOVER)
- E5 (GNSS State) fires when GNSS fix status changes independently of the T-GM composite state
- All events are edge-triggered: published only when the value changes, not periodically
- SyncE events (`event.sync.synce-status.synce-state-change` and `event.sync.synce-status.synce-clock-quality-change`) are applicable when SyncE is configured

---

## 13. User Contract

This section defines the behavioral contract between the system and its users: what information the system requires from the user (inputs), and what information the system provides back (outputs).

### 13.1 User Inputs — Declaring Intent

The user configures the system through two Kubernetes custom resources: the `PtpConfig` (PTP software configuration and clock behavior) and the `HardwareConfig` (clock chain hardware topology). The intent declaration spans three layers.

#### 13.1.1 Clock Type and IEEE 1588 Profile

The user declares the desired clock role and the IEEE 1588 PTP profile. The system derives process topology, startup order, announce behavior, and state machine semantics from this declaration.

| Parameter    | Values                       | Effect                                                                                                                                       |
| :----------- | :--------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------- |
| `clockType`  | `T-GM`                       | Determines the set of processes started, port roles (TimeTransmitter only), announce behavior, and which state machine specification applies |
| `ptpProfile` | IEEE 1588 profile identifier | Determines transport, delay mechanism, BTCA variant, domain number range, and which PTP parameters are profile-mandated                      |

When a profile is designated, certain PTP parameters are fixed by the profile and must not be overridden by the user. The system must reject invalid combinations at admission time with an informative error identifying the violating parameter.

**Profile-Mandated Parameters (Tier 1) — [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en):**

| Parameter            | Mandated value ([G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en)) | Rationale                                                                                                        |
| :------------------- | :--------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------- |
| `network_transport`  | `L2`                                                                   | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) requires Layer 2 multicast transport                       |
| `delay_mechanism`    | `E2E`                                                                  | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) requires end-to-end delay measurement                      |
| `time_stamping`      | `hardware`                                                             | Software timestamping is insufficient for PRTC accuracy                                                          |
| `dataset_comparison` | `[G.8275](https://www.itu.int/rec/t-rec-g.8275/en).x`                  | Required for [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) BTCA behavior                                 |
| `transportSpecific`  | `0x0`                                                                  | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) uses the default PTP transport specific value              |
| `priority1`          | `128`                                                                  | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) alternate BTCA ignores priority1; value must remain at 128 |
| `clock_type`         | `OC` or `BC` only                                                      | T-GM ports must be either ordinary or boundary clock ports                                                       |

**T-GM Operator-Tunable Parameters (Tier 2):**

| Parameter                     | Default     | Range / values        | Notes                                                                               |
| :---------------------------- | :---------- | :-------------------- | :---------------------------------------------------------------------------------- |
| `logAnnounceInterval`         | `-3` (8/s)  | `-4` to `0`           | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) recommends -3                 |
| `logSyncInterval`             | `-4` (16/s) | `-7` to `-1`          |                                                                                     |
| `logMinDelayReqInterval`      | `-4`        | `-7` to `-1`          |                                                                                     |
| `domainNumber`                | `24`        | `24`–`43`             | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) reserves domain numbers 24–43 |
| `priority2`                   | `128`       | `0`–`255`             | For multi-GM redundancy ordering                                                    |
| `MaxInSpecOffset`             | —           | positive integer (ns) | Holdover in-spec / out-of-spec boundary                                             |
| `LocalMaxHoldoverOffSet`      | —           | positive integer (ns) | Holdover-to-freerun boundary                                                        |
| `LocalHoldoverTimeout`        | —           | positive integer (s)  | Maximum holdover duration                                                           |
| `processDowntimeThresholds.*` | `5`         | 0–86400 (s)           | Acceptable process downtime before holdover/freerun events are emitted              |

**Additional Configuration Requirements:**

- **Per-port role assignment:** Each NIC port shall be individually configurable as `masterOnly 1` (time transmitter) or ommitted (no TimeReceivers may be configured for T-GM casee)
- **DPLL and SyncE parameters:** Reference input priorities and SyncE network option ([ITU-T G.8264](https://www.itu.int/rec/T-REC-G.8264) Option 1 or Option 2) shall be configurable.

#### 13.1.2 GNSS Configuration

The user configures the behaviour and parameters of the GNSS receiver to adapt to their deployment environment and antenna placement.

| Parameter           | Default           | Effect                                                                                                    |
| :------------------ | :---------------- | :-------------------------------------------------------------------------------------------------------- |
| `constellations`    | `GPS, Galileo`    | Selects which GNSS constellations the receiver tracks (e.g., GPS, Galileo, GLONASS, BeiDou, QZSS)         |
| `1ppsPulseWidth`    | (Vendor-specific) | The duration of the 1PPS signal pulse width                                                               |
| `surveyDuration`    | `86400` (24h)     | Minimum duration of the GNSS antenna survey (in seconds)                                                  |
| `surveyMinAccuracy` | `5000` (50cm)     | Survey-in position accuracy limit (in 0.1 mm units; e.g., 5000 = 50 cm)                                   |
| `antennaCableDelay` | `0` (ns)          | User-supplied antenna cable delay value to compensate for the time offset introduced by the antenna cable |
| `customCommands`    | (None)            | A user-supplied list of raw/custom commands to append during GNSS configuration                           |

**GNSS Antenna Survey:**
The system shall perform a GNSS antenna survey according to the user's choices. The initial survey allows the receiver to accurately determine its fixed 3D position by averaging errors over time. By default, this initial survey duration is 24 hours to ensure high-accuracy timing references. Once the survey completes, the receiver transitions into a timing-only fixed-position mode. The GNSS survey position should be saved into the receiver to avoid the need to conduct a survey on every receiver start.

Following transition into fixed-position mode, the system shall monitor for position drift. If the drift exceeds the initial surveyMinAccuracy, the survey procedure shall be restarted to reestablish a fixed position.

#### 13.1.3 Hardware Configuration (Clock Chain)

The user declares the hardware clock chain topology through the `HardwareConfig` custom resource. This resource describes the physical synchronization path: DPLL complexes, pins, connectors, phase/frequency inputs and outputs, delay compensations, and behavioral conditions for dynamic reconfiguration.

The HardwareConfig is composed of two parts:

| Part          | Description                                                                                                                                                                                                                 |
| :------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Structure** | Static declaration of subsystems (GNSS receiver, DPLLs, Ethernet ports, pins, connectors), including phase/frequency inputs and outputs with delay compensation values. Each subsystem is associated with a hardware plugin |
| **Behavior**  | Declaration of synchronization sources, conditions (locked/lost), and the desired pin states for each condition. The system dynamically switches between condition-matched states based on source availability              |

When a `HardwareConfig` is provided, it takes precedence over plugin-derived hardware configuration for the associated PTP profile.

### 13.2 System Outputs — What the System Provides

The system provides synchronization state and health information through multiple channels. Each channel serves a different consumer and access pattern.

| Output Channel                        | Content                                                                                     | Consumer                                         | Reference |
| :------------------------------------ | :------------------------------------------------------------------------------------------ | :----------------------------------------------- | :-------- |
| **Prometheus metrics**                | Clock state, GNSS status, offsets, DPLL status, interface roles, process health, thresholds | Monitoring dashboards, alerting                  | §11.1     |
| **CloudEvents / O-RAN notifications** | State transitions (GNSS, DPLL, PTP), clock class changes                                    | Event-driven consumers, O-RAN O-Cloud            | §12       |
| **Kubernetes Events**                 | Clock state changes, process restarts on the PtpConfig resource                             | `kubectl describe`, cluster event sinks          | §11.2.2   |
| **PtpConfig.status**                  | Configuration errors (node, condition, diagnostic message)                                  | Kubernetes controllers, operators                | §11.2.1   |
| **NodePtpDevice.status**              | Discovered PTP devices, hardware info, per-device config status (failed/success)            | Kubernetes controllers, hardware inventory       | §11.2.2   |
| **HardwareConfig.status**             | Matched nodes, active behavior condition per node                                           | Kubernetes controllers, clock chain verification | §11.2.3   |
| **Structured logs**                   | State transitions, offset measurements, GNSS fix changes, hardware reconfiguration          | Troubleshooting, audit trail                     | §11.3     |
| **PTP Announce messages**             | Clock class, clock accuracy, time properties, GM identity                                   | Downstream PTP nodes                             | §8        |

## 14. Security

1. The system shall support PTP authentication (IEEE 1588 Annex P / NTS) when configured.
2. When authentication key material changes, the system shall detect the change and restart affected processes to pick up the new keys.
3. When GNSS spoofing or jamming is detected by the GNSS hardware, the system shall enter FREERUN.

---

## 15. Functional Requirements Specification

### 15.1 State Machine — Lock Acquisition

#### ID: FUNC-TGM001

| Requirement ID | Traceability                                                       | Requirement text                                                                                                                                                                                                            |
| :------------- | :----------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM001-R1 | 6.4 AllLocked                                                      | Given a T-GM is initialised in Free-Run state, when GNSS is acquired and all lock conditions are met (GNSS fix valid, DPLL locked, PHC locked, all offsets within threshold), then the T-GM must transition to Locked state |
| FUNC-TGM001-R2 | 8.1, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Upon entering Locked state, Announce messages on all ports must reflect clockClass 6, clockAccuracy 0x21, timeSource GNSS (0x20), timeTraceable TRUE, frequencyTraceable TRUE                                               |

### 15.2 State Machine — Holdover Entry on GNSS Loss

#### ID: FUNC-TGM002

| Requirement ID | Traceability                                                       | Requirement text                                                                                                                                                                               |
| :------------- | :----------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM002-R1 | 6.4 GNSSLost, 6.5 T3                                               | Given a T-GM is in Locked state, when the GNSS reference is lost and the DPLL enters holdover or locked-holdover-acquired, then the T-GM must transition to Holdover-In-Spec                   |
| FUNC-TGM002-R2 | 8.2, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Upon entering Holdover-In-Spec, Announce messages must reflect clockClass 7, clockAccuaracy unknown (0xFE), timeSource internal oscillator (0xA0), timeTraceable TRUE, frequencyTraceable TRUE |
| FUNC-TGM002-R3 | 7                                                                  | Given a T-GM enters Holdover, it must maintain a unidirectional signal flow without reversing the synchronisation direction                                                                    |

### 15.3 State Machine — Holdover In-Spec to Out-Of-Spec

#### ID: FUNC-TGM003

| Requirement ID | Traceability                                                       | Requirement text                                                                                                                                                                                          |
| :------------- | :----------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM003-R1 | 6.4 Offset-Above-InSpecOffset, 6.5 T6                              | Given a T-GM is in Holdover-In-Spec, when the measured phase offset exceeds `MaxInSpecOffset` or the holdover timer exceeds `LocalHoldoverTimeout`, then the T-GM must transition to Holdover-Out-Of-Spec |
| FUNC-TGM003-R2 | 8.3, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Upon entering Holdover-Out-Of-Spec, Announce messages must reflect clockClass 140, clockAccuracy unknown (0xFE), timeSource internal oscillator (0xA0), timeTraceable FALSE, frequencyTraceable TRUE      |

### 15.4 State Machine — Holdover to Free-Run

#### ID: FUNC-TGM004

| Requirement ID | Traceability                                                       | Requirement text                                                                                                                                                                          |
| :------------- | :----------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM004-R1 | 6.4 Holdover-To-FreeRun, 6.5 T7/T9                                 | Given a T-GM is in any Holdover state, when the measured or estimated phase offset exceeds `LocalMaxHoldoverOffSet`, then the T-GM must transition to Free-Run                            |
| FUNC-TGM004-R2 | 8.4, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Upon entering Free-Run, Announce messages must reflect clockClass 248, clockAccuracy unknown (0xFE), timeSource internal oscillator (0xA0), timeTraceable FALSE, frequencyTraceable FALSE |

### 15.5 State Machine — Re-Lock from Degraded State

#### ID: FUNC-TGM005

| Requirement ID | Traceability                                                       | Requirement text                                                                                                                                                                  |
| :------------- | :----------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM005-R1 | 6.4 AllLocked, 6.5 T5/T8                                           | Given a T-GM is in any Holdover or Free-Run state, when GNSS reference is re-acquired and all lock conditions are met per FUNC-TGM001-R1, then the T-GM must transition to Locked |
| FUNC-TGM005-R2 | 8.1, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Upon re-lock, Announce messages must be restored to Locked-state parameters (See FUNC-TGM001-R2)                                                                                  |

### 15.6 State Machine — Non-GNSS Fault to Free-Run

#### ID: FUNC-TGM006

| Requirement ID | Traceability             | Requirement text                                                                                                                                                                                                                           |
| :------------- | :----------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM006-R1 | 6.4 NonGNSSFault, 6.5 T4 | Given a T-GM is in Locked state, when a non-GNSS-loss fault occurs that prevents time traceability (e.g., PHC synchroniser enters FREERUN while DPLL is LOCKED, or a process crash is detected), then the T-GM must transition to Free-Run |

### 15.7 Initialisation — Clock Type and Initial State

#### ID: FUNC-TGM007

| Requirement ID | Traceability | Requirement text                                                                                                                                                                                       |
| :------------- | :----------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM007-R1 | 6.6 step 1   | Given a node configuration is applied, then the system must determine the PTP clock type (T-GM) from the `clockType` field in the PtpConfig resource                                                   |
| FUNC-TGM007-R2 | 6.6 step 2   | Given the clock type is determined, then processes must be started in the required startup order with generated configurations                                                                         |
| FUNC-TGM007-R3 | 6.6 step 4   | Given initialisation completes, then the system must enter the Free-Run state unconditionally, regardless of whether GNSS is already available                                                         |
| FUNC-TGM007-R4 | 6.5 T1       | Given a T-GM has just initialised, when GNSS is already available, then the system must NOT bypass Free-Run — it must progress through the full lock acquisition sequence to reach Locked              |
| FUNC-TGM007-R5 | 13.1.1       | Given a telecom profile is designated, the system must validate that the user configuration conforms to profile-mandated parameter constraints and reject it at admission time if a violation is found |

### 15.8 GNSS Initialisation and Configuration

#### ID: FUNC-TGM008

| Requirement ID | Traceability | Requirement text                                                                                                                                                                                                           |
| :------------- | :----------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM008-R1 | 13.1.2       | Given a T-GM is initialising its GNSS receiver, it must configure the receiver to track the user-specified constellations (defaulting to GPS + Galileo)                                                                    |
| FUNC-TGM008-R2 | 13.1.2       | Given a T-GM is initialising its GNSS receiver, if a saved fixed 3D position does not exist or a re-survey is forced, it must initiate a GNSS antenna survey using the configured `surveyDuration` and `surveyMinAccuracy` |
| FUNC-TGM008-R3 | 13.1.2       | Given a GNSS antenna survey completes successfully, the system must save the surveyed fixed 3D position into the receiver's non-volatile storage to avoid unnecessary re-surveys on subsequent restarts                    |
| FUNC-TGM008-R4 | 13.1.2       | Given a T-GM is initialising its GNSS receiver, it must configure the receiver to apply the user-supplied `antennaCableDelay` to compensate for coaxial cable propagation delay                                            |
| FUNC-TGM008-R5 | 13.1.2       | Given a T-GM is initialising its GNSS receiver, if `customCommands` are provided, the system must successfully transmit and apply these raw commands to the receiver                                                       |

### 15.9 Per-State Announce Content Verification

#### ID: FUNC-TGM009

| Requirement ID | Traceability                                                       | Requirement text                                                                                                                                                                                                        |
| :------------- | :----------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM009-R1 | 8.1, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Given a T-GM is in Locked state, then Announce messages must contain: clockClass=6, clockAccuracy=0x21, offsetScaledLogVariance=0x4E5D, timeSource=GNSS(0x20), timeTraceable=TRUE, frequencyTraceable=TRUE              |
| FUNC-TGM009-R2 | 8.2, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Given a T-GM is in Holdover-In-Spec state, then Announce messages must contain: clockClass=7, clockAccuracy=0xFE, offsetScaledLogVariance=0xFFFF, timeSource=INT_OSC(0xA0), timeTraceable=TRUE, frequencyTraceable=TRUE |
| FUNC-TGM009-R3 | 8.3, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Given a T-GM is in Holdover-Out-Of-Spec state, then Announce messages must contain: clockClass=140, clockAccuracy=0xFE, timeTraceable=FALSE, frequencyTraceable=TRUE                                                    |
| FUNC-TGM009-R4 | 8.4, [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Table 2 | Given a T-GM is in Free-Run state, then Announce messages must contain: clockClass=248, clockAccuracy=0xFE, timeTraceable=FALSE, frequencyTraceable=FALSE                                                               |
| FUNC-TGM009-R5 | 8.5                                                                | Given a T-GM is in any state, then `currentUtcOffset` must contain the correct TAI−UTC offset and `currentUtcOffsetValid` must be TRUE                                                                                  |
| FUNC-TGM009-R6 | 8.6, 6.7                                                           | Given a T-GM transitions between any two states, then Announce messages on all ports must reflect the new state within one Announce interval                                                                            |

### 15.10 Events — State Change Notification

#### ID: FUNC-TGM010

| Requirement ID | Traceability | Requirement text                                                                                                                                                                                                                     |
| :------------- | :----------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM010-R1 | 12.5, 6.7    | Given a monitoring system is subscribed to T-GM state events, when the T-GM transitions between any two states, then a CloudEvents-formatted `event.sync.ptp-status.ptp-state-change` notification must be published within 1 second |
| FUNC-TGM010-R2 | 12.2         | The event must contain the new state, previous state, and relevant metric values                                                                                                                                                     |
| FUNC-TGM010-R3 | 12.5, 6.7    | Given the T-GM clock class changes, then an `event.sync.ptp-status.ptp-clock-class-change` event must be published within 1 second                                                                                                   |
| FUNC-TGM010-R4 | 12.5, 6.7    | Given the GNSS fix state changes, then an `event.sync.gnss-status.gnss-state-change` event must be published within 1 second                                                                                                         |
| FUNC-TGM010-R5 | 12.2         | Given the T-GM is in a stable state, then state change events must not be published periodically, but only edge-triggered upon actual state or value transition                                                                      |

### 15.11 Process Orchestration — Startup Order and Lifecycle

#### ID: FUNC-TGM011

| Requirement ID | Traceability | Requirement text                                                                                                                                                           |
| :------------- | :----------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM011-R1 | 9.2 step 1–2 | Given a T-GM configuration is applied, then the GNSS monitor and DPLL monitor must be the first processes started                                                          |
| FUNC-TGM011-R2 | 9.2 step 3   | Given GNSS and DPLL monitors are running, then ts2phc must be started next                                                                                                 |
| FUNC-TGM011-R3 | 9.2 step 4   | Given ts2phc is running, then ptp4l must be started                                                                                                                        |
| FUNC-TGM011-R4 | 9.2 step 5   | Given ts2phc has not yet reported LOCKED (s2), then phc2sys must NOT be started                                                                                            |
| FUNC-TGM011-R5 | 9.3          | Given a process exits unexpectedly, then it must be restarted automatically, and other healthy processes must NOT be stopped and restarted solely to re-establish ordering |
| FUNC-TGM011-R6 | 9.3          | Given a configuration change occurs, then all processes must be stopped and restarted in the correct startup order                                                         |

### 15.12 Process Failure — T-GM Behaviour

#### ID: FUNC-TGM012

| Requirement ID | Traceability | Requirement text                                                                                                                                        |
| :------------- | :----------- | :------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FUNC-TGM012-R1 | 9.5, ts2phc  | Given ts2phc crashes and restarts within `processDowntimeThresholds.ts2phc`, then no state-change events must be emitted                                |
| FUNC-TGM012-R2 | 9.5, ts2phc  | Given ts2phc crashes and remains down beyond `processDowntimeThresholds.ts2phc`, then the system must enter holdover and emit clock class change events |
| FUNC-TGM012-R3 | 9.5, ptp4l   | Given ptp4l crashes, then downstream clients lose synchronisation; clock class change events must be emitted if downtime exceeds the threshold          |
| FUNC-TGM012-R4 | 9.5, phc2sys | Given phc2sys crashes and restarts within its threshold, then E3 (os-clock-sync-state-change) must NOT toggle LOCKED→FREERUN→LOCKED                     |
| FUNC-TGM012-R5 | 9.3          | Given any timing process crashes, then it must be automatically restarted and `openshift_ptp_process_restart_count` must be incremented                 |

### 15.13 Observability — Metrics Emission

#### ID: FUNC-TGM013

| Requirement ID | Traceability          | Requirement text                                                                                                                                                       |
| :------------- | :-------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM013-R1 | 11.1.1 clock_state    | Given a T-GM is running, then `openshift_ptp_clock_state` must be emitted per interface and per process with values 0=FREERUN, 1=LOCKED, 2=HOLDOVER                    |
| FUNC-TGM013-R2 | 11.1.1 clock_class    | Given a T-GM is running, then `openshift_ptp_clock_class` must be emitted with the current clock class value and updated within 1 second of a change                   |
| FUNC-TGM013-R3 | 11.1.2 offset_ns      | Given a T-GM is running, then `openshift_ptp_offset_ns` must be emitted per measurement point (ts2phc, GNSS, DPLL, phc2sys)                                            |
| FUNC-TGM013-R4 | 11.1.4 interface_role | Given a T-GM is running, then all PTP ports must report `openshift_ptp_interface_role` = 2 (TT/MASTER)                                                                 |
| FUNC-TGM013-R5 | 11.1.5 process_status | Given a T-GM is running, then `openshift_ptp_process_status` (0=DOWN, 1=UP) and `openshift_ptp_process_restart_count` must be emitted per managed process              |
| FUNC-TGM013-R6 | 10.4, 11.1.3          | Given a T-GM has GNSS and DPLL subsystems, then GNSS fix state, GNSS offset, DPLL lock state, and DPLL phase offset metrics must be published at least once per second |

---

### 15.14 SyncE (Synchronous Ethernet) Distribution

#### ID: FUNC-TGM014

| Requirement ID | Traceability | Requirement text                                                                                                                                                       |
| :------------- | :----------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM014-R1 | 5.3          | Given SyncE is configured, the T-GM must distribute frequency from the frequency-locked oscillator to the Ethernet PHY on all configured SyncE ports                   |
| FUNC-TGM014-R2 | 5.3          | Given SyncE is configured, the ESMC protocol must advertise QL-PRC (or equivalent) when the T-GM is LOCKED, a degraded QL when in HOLDOVER, and QL-DNU when in FREERUN |
| FUNC-TGM014-R3 | 12.5, 6.7    | Given SyncE is configured, when the SyncE clock quality or state changes, the system must publish `event.sync.synce-status.*` events within 1 second                   |

### 15.15 Security and Authentication

#### ID: FUNC-TGM015

| Requirement ID | Traceability | Requirement text                                                                                                                                                        |
| :------------- | :----------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM015-R1 | 14           | Given PTP authentication (IEEE 1588 Annex P / NTS) is configured, the system must append authentication TLVs to transmitted PTP messages and validate incoming messages |
| FUNC-TGM015-R2 | 14           | Given authentication key material is updated, the system must detect the change and restart affected processes without requiring a full pod restart                     |
| FUNC-TGM015-R3 | 14           | Given the GNSS hardware detects spoofing or jamming, the system must immediately force the clock state manager to evaluate GNSS as FREERUN                              |

### 15.16 Drift Monitoring and Re-Survey

#### ID: FUNC-TGM016

| Requirement ID | Traceability | Requirement text                                                                                                                                                  |
| :------------- | :----------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FUNC-TGM016-R1 | 13.1.2       | Given the GNSS receiver has transitioned into fixed-position mode, the system must monitor for position drift                                                     |
| FUNC-TGM016-R2 | 13.1.2       | Given position drift exceeds the configured `surveyMinAccuracy`, the system must restart the GNSS antenna survey procedure to reestablish a new fixed 3D position |

## 16. Performance Requirements Specification

### 16.1 [G.8272](https://www.itu.int/rec/t-rec-g.8272/en) PRTC Time Error

[**Source: Rec. ITU-T G.8272/Y.1367 (11/2018)**](https://www.itu.int/rec/T-REC-G.8272/en)

When in the LOCKED state, the T-GM must meet the time error limits of a Primary Reference Time Clock (PRTC) as defined in [ITU-T G.8272](https://www.itu.int/rec/t-rec-g.8272/en).

| Requirement ID                                                                                                                                                                        | Traceability                                                          | Requirement text                                                                                                                                  |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :-------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| PERF-TGM001-R1                                                                                                                                                                        | [G.8272](https://www.itu.int/rec/t-rec-g.8272/en) Section 6.1, PRTC-A | Maximum absolute time error must not exceed ±100 ns relative to a recognised time reference (e.g., UTC via GNSS)                                  |
| PERF-TGM001-R2                                                                                                                                                                        | [G.8272](https://www.itu.int/rec/t-rec-g.8272/en) Section 6.2, PRTC-B | Maximum absolute time error must not exceed ±40 ns relative to a recognised time reference (where PRTC-B is supported by the hardware)            |
| PERF-TGM001-R3                                                                                                                                                                        | [G.8272](https://www.itu.int/rec/t-rec-g.8272/en), general            | The specific PRTC class supported (A or B) depends on the hardware platform. The system must be configurable to declare which PRTC class it meets |
| NOTE. The locked-mode offset between the PHC and the GNSS 1PPS reference, as measured by the PHC synchroniser, must remain within the configured offset threshold (default: ±100 ns). |                                                                       |                                                                                                                                                   |

### 16.2 Holdover Performance

| Requirement ID | Traceability       | Requirement text                                                                                                                                                                                                                                                       |
| :------------- | :----------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PERF-TGM002-R1 | 6.8 Holdover Model | During holdover (in-spec), the clock must maintain time error within `MaxInSpecOffset` from the last known good reference                                                                                                                                              |
| PERF-TGM002-R2 | 6.8 Holdover Model | The actual holdover performance (slope of time error drift) depends on the hardware oscillator quality. The system must expose sufficient observability (offset metrics, holdover duration) to allow operators to verify compliance with their deployment requirements |

### 16.3 Timing Message Rates

| Requirement ID | Traceability                                                      | Requirement text                                                                                                                                                                                                |
| :------------- | :---------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PERF-TGM003-R1 | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Section 6.2 | The PTP engine must transmit Sync messages at the configured rate. Per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en), the default rate is 16 messages per second (logSyncInterval = −4) per port        |
| PERF-TGM003-R2 | [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) Section 6.2 | The PTP engine must transmit Announce messages at the configured rate. Per [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en), the default rate is 8 messages per second (logAnnounceInterval = −3) per port |
| PERF-TGM003-R3 | 10.4                                                              | The PHC synchroniser must update its offset measurement at least once per second                                                                                                                                |
| PERF-TGM003-R4 | 13.1.1                                                            | The PTP engine must respond to Delay_Req messages from downstream clients per the end-to-end delay mechanism mandated by [G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en)                                  |

---

## 17. References

- [ITU-T G.8272](https://www.itu.int/rec/t-rec-g.8272/en) — Timing characteristics of primary reference time clocks (PRTC-A, PRTC-B)
- [ITU-T G.8272.1](https://www.itu.int/rec/t-rec-g.8272.1/en) — Timing characteristics of enhanced primary reference time clocks (ePRTC)
- [ITU-T G.8273.2](https://www.itu.int/rec/T-REC-G.8273.2/en) — T-BC/T-TSC timing characteristics
- [ITU-T G.8275](https://www.itu.int/rec/t-rec-g.8275/en) — Framework and requirements for packet-based frequency, phase, and time distribution
- [ITU-T G.8275.1](https://www.itu.int/rec/t-rec-g.8275.1/en) — PTP telecom profile for phase/time synchronisation with full timing support from the network
- [ITU-T G.8262](https://www.itu.int/rec/t-rec-g.8262) — Timing characteristics of synchronous equipment clocks (EEC)
- [ITU-T G.8264](https://www.itu.int/rec/T-REC-G.8264) — Distribution of timing via Ethernet networks (ESMC / SyncE)
- [ITU-T G.811](https://www.itu.int/rec/t-rec-g.811) — Primary Reference Clock
- IEEE 1588-2019 — Precision Time Protocol
- O-RAN O-Cloud Notification API v2
- CloudEvents Specification v1.0
