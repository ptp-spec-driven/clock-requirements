# Feature Specification: T-TSC Testable Requirements

**Module**: PTP Operator Stack (np-ptp-operator, np-linuxptp-daemon, cloud-event-proxy)
**Author**: Vitaly Grinberg
**Created**: 2026-06-21
**Last Updated**: 2026-06-21
**Status**: Draft

---

## 1. Overview

Testable requirement specification for the Telecom Time Synchronous Clock (T-TSC) implementation. The T-TSC is a time receiver only — it synchronizes to an upstream PTP source but does NOT distribute timing downstream. This spec defines the behavioral differences from the T-BC specification. Where not stated otherwise, T-TSC behavior follows the T-BC spec.

**Parent specification**: [T-BC Testable Requirements](../tbc/spec.md)

---

## 2. Goals

- Define T-TSC-specific behavioral differences from T-BC
- Cross-reference the T-BC spec for shared behavior (state machine, holdover model, events, metrics)
- Provide testable requirements specific to T-TSC

## 3. Non-Goals

- Duplicating content already defined in the T-BC spec
- Defining T-GM (Grandmaster) behavior

---

## 4. Relationship to T-BC

The T-TSC is a **subset** of the T-BC. Key differences:

| Aspect | T-BC | T-TSC |
| :--- | :--- | :--- |
| Downstream distribution | Yes — TT ports transmit Announce/Sync | **No** — no TT ports |
| ptp4l instances | ptp4l[TR] + ptp4l[TT] | **ptp4l[TR] only** |
| Announce message management | IWF to TT ports, PMC updates on state transitions | **Not applicable** — no downstream announce |
| Clock type in ptp4l config | `clock_type BC` with `boundary_clock_jbod 1` | **`clock_type OC`** (slave-only Ordinary Clock) |
| Clock class | Dynamic (6/135/165/248 per state) | **Always 255** (Slave Only Clock) |
| Multi-NIC support | 2-NIC and 3-NIC fan-out | **Single-NIC only** |
| SyncE | Applicable (frequency traceability for assisted holdover) | Applicable (same as T-BC) |

## 5. Shared Behavior (by reference to T-BC spec)

The following T-BC spec sections apply to T-TSC without modification:

- **Section 6**: T-BC State Machine — states, transition conditions, holdover model (all apply to T-TSC)
- **Section 7**: Synchronization Direction and Hardware Reconfiguration — pin states apply for the TR NIC
- **Section 10**: Clock components monitoring — offset measurement points and thresholds
- **Section 11**: Observability and Diagnostics — metrics (excluding TT-specific metrics)
- **Section 12**: Event and Notification Behavior — E1, E3, E4 apply; E2 (clock class change) applies for local reporting only
- **Section 16**: Performance Requirements — G.8273.2 class C requirements apply equally

## 6. T-TSC-Specific Differences

### 6.1 No Downstream Distribution

The T-TSC has no Time Transmitter (TT) ports. This means:
- No ptp4l[TT] instance is started
- No Announce/Sync messages are transmitted downstream
- No PMC updates for grandmaster settings on TT ports
- No multi-NIC fan-out

### 6.2 Process Orchestration

T-TSC startup order (derived from [T-BC section 9.2](../tbc/spec.md#92-t-bc-process-startup-order), excluding ptp4l[TT]):

| Step | Process | Precondition |
| :--- | :--- | :--- |
| 1 | **ptp4l[TR]** | Configuration applied, DPLL pins in init state |
| 2 | **phc2sys** | PTPSourceQualified condition met (see T-BC section 6.3) |
| 3 | **ts2phc** | PTPSourceQualified condition met (see T-BC section 6.3). Must start after phc2sys (ts2phc depends on phc2sys for ToD) |

ptp4l[TT] is **not started** for T-TSC. The process startup order is clock-type-specific; see [T-BC section 9.2](../tbc/spec.md#92-t-bc-process-startup-order) for the rationale and precondition details.

### 6.3 Clock Class and Announce Message Behavior

T-TSC is a slave-only Ordinary Clock. Its clock class is **always 255** (`ClockClassSlaveOnly`). This value:
- Does NOT change with state transitions (unlike T-BC where clock class reflects the current state)
- Must be reported as 255 in the `openshift_ptp_clock_class` metric at all times
- Must be reported as 255 in E2 (clock class change) events — since the value never changes, E2 should only fire once at initialization

T-TSC does not transmit announce messages. The per-state announce content tables in T-BC spec section 8 do not apply.

The T-TSC state machine still operates internally (Free-Run / Locked / Holdover-In-Spec / Holdover-Out-Of-Spec) for:
- E1 (PTP Lock State) event reporting
- Offset estimation and holdover tracking
- E4 (Overall Sync State) aggregation

### 6.4 Upstream Port Redundancy

T-TSC may support redundant upstream TR ports on a single NIC. See [T-BC spec section 5.6](../tbc/spec.md) for upstream port redundancy behavior — the same semantics apply to T-TSC.

---

## 14. T-TSC Functional Requirements

### ID: FUNC-TTSC001

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TTSC001-R1 | 6.1 | Given a T-TSC is configured, then no ptp4l[TT] instance must be started |
| FUNC-TTSC001-R2 | 6.1 | Given a T-TSC is configured, then no Announce or Sync messages must be transmitted on any port |
| FUNC-TTSC001-R3 | 6.3 | Given a T-TSC is configured, then `openshift_ptp_clock_class` must always report 255 (Slave Only Clock), regardless of the internal state machine state |
| FUNC-TTSC001-R4 | 6.3 | Given a T-TSC transitions between states, then E2 (clock class change) must NOT fire — the clock class value (255) never changes |

### ID: FUNC-TTSC002

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TTSC002-R1 | 6.2 | Given a T-TSC is configured, then the process startup order must be: ptp4l[TR] → phc2sys (conditional on PTPSourceQualified) → ts2phc (conditional on PTPSourceQualified, after phc2sys) |
| FUNC-TTSC002-R2 | 6.2 | Given a T-TSC shutdown occurs, then processes must stop in reverse order: ts2phc → phc2sys → ptp4l[TR] |

### ID: FUNC-TTSC003

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TTSC003-R1 | T-BC 6.1–6.8 | Given a T-TSC is in any state, then the state machine behavior (states, transitions, conditions, holdover model) must follow the T-BC spec sections 6.1–6.8 |
| FUNC-TTSC003-R2 | T-BC 12.5 | Given a T-TSC transitions between states, then events E1 (PTP Lock State), E3 (OS Clock Sync), and E4 (Overall Sync) must be emitted per the T-BC event matrix. E2 (Clock Class) must NOT fire — clock class is fixed at 255 |

---

## 17. References

- [T-BC Testable Requirements](../tbc/spec.md)
- ITU-T G.8273.2 — Timing characteristics of T-BC and T-TSC
- ITU-T G.8275.1 — PTP telecom profile for phase/time with full timing support
- ITU-T G.8275 (2024) Amd.1 — Clock state mode definitions
- IEEE 1588-2019 — Precision Time Protocol
