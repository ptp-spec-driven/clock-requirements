# T-GM Specification — Set-Aside Sections

**Status**: Preserved for future integration
**Origin**: T-GM behavioural requirements (requirements repo, `t-gm-unified` branch)

These sections are from the original T-GM behavioural requirements document
but do not have equivalent sections in the T-BC specification structure.
They are preserved here for future integration into `spec.md` or into a
shared cross-clock-type specification.

---

## A. Hardware Requirements

These requirements define what the hardware must expose to the operating
system. They are hardware-agnostic — any NIC or timing card that satisfies
these requirements is a valid platform.

1. The hardware shall provide at least one PTP Hardware Clock (`/dev/ptpN`)
   capable of sub-microsecond timestamp resolution.

2. The hardware shall provide a DPLL that can lock to an external 1PPS
   reference and report its lock state (FREERUN, LOCKED, LOCKED_HO_ACQ,
   HOLDOVER) via the Linux DPLL netlink subsystem.

3. The hardware shall provide a GNSS receiver (embedded or external)
   accessible via a serial device (e.g., `/dev/gnss0`) that outputs 1PPS
   and ToD (NMEA) signals.

4. The hardware's DPLL shall report both frequency status and phase status
   independently via the Linux DPLL netlink subsystem.

5. The hardware's DPLL shall report phase offset (in picoseconds) relative
   to the reference input via the Linux DPLL netlink subsystem.

6. The hardware oscillator shall be capable of maintaining time within the
   holdover performance limits defined in spec.md §6.8 when the GNSS
   reference is lost.

7. When SyncE is required, the hardware shall provide an EEC clock capable
   of locking to the DPLL output and distributing frequency via the
   Ethernet physical layer, per ITU-T G.8262.

8. The hardware shall support at least one Ethernet port capable of
   IEEE 1588 hardware timestamping at Layer 2 for PTP message exchange.

**Hardware abstraction interfaces:**

| Interface | Purpose |
| :--- | :--- |
| `/dev/ptpN` (PTP clock device) | PHC access; timestamp reads, frequency adjustment, pin configuration |
| Linux DPLL subsystem (netlink family `dpll`) | DPLL lock state, phase offset, frequency status |
| `/dev/gnssN` or equivalent serial device | GNSS receiver communication (NMEA sentences, vendor-specific monitoring) |
| `ethtool` / sysfs | NIC capabilities discovery, SyncE configuration |

---

## B. Software Requirements — Detailed Component Contracts

### B.1 Configuration Management — Profile Tiers

> **Scope note:** The T-GM is one of several use cases supported by this operator
> (others include ordinary clock, boundary clock, T-BC, T-TSC, and non-telecom
> profiles). The tiering model and profile-mandated constraints below apply
> **only when a `PtpProfile` is explicitly designated as targeting a specific
> ITU-T telecom profile**. Profiles not carrying such a designation remain fully
> open and may set any `ptp4l` parameter to any value.

For designated telecom profiles, the operator shall organise configuration
into three tiers:

| Tier | Name | Description | Operator action on violation |
| :--- | :--- | :--- | :--- |
| 1 | **Profile-mandated** | Parameters whose values are fixed by the selected ITU-T profile (G.8275.1 or G.8275.2). Changing them breaks profile conformance. | **Reject** at admission time with an informative error. |
| 2 | **Operator-tunable** | Parameters that may legitimately vary between deployments (servo gains, log intervals within allowed ranges, scheduling priority, thresholds). | Accept; surface current values in `PtpConfig.status`. |
| 3 | **Internal / reserved** | Parameters managed entirely by the daemon or plugin; not user-visible. | Silently applied by daemon; not exposed in the CRD. |

**Profile-Mandated Parameters (Tier 1) — G.8275.1:**

| Parameter | Mandated value (G.8275.1) | Rationale |
| :--- | :--- | :--- |
| `network_transport` | `L2` | G.8275.1 requires Layer 2 multicast transport. |
| `delay_mechanism` | `E2E` | G.8275.1 requires end-to-end delay measurement. |
| `time_stamping` | `hardware` | Software timestamping is insufficient for PRTC accuracy. |
| `dataset_comparison` | `G.8275.x` | Required for G.8275.1 BMCA behaviour. |
| `transportSpecific` | `0x0` | G.8275.1 uses the default PTP transport specific value. |
| `priority1` | `128` | G.8275.1 alternate BMCA ignores priority1; value must remain at 128. |
| `clock_type` | `OC` or `BC` only | T-GM ports must be either ordinary or boundary clock ports. |

**Operator-Tunable Parameters (Tier 2):**

| Parameter | Default | Allowed range / values | Notes |
| :--- | :--- | :--- | :--- |
| `logAnnounceInterval` | `-3` (8/s) | `-4` to `0` | G.8275.1 recommends -3 |
| `logSyncInterval` | `-4` (16/s) | `-7` to `-1` | |
| `logMinDelayReqInterval` | `-4` | `-7` to `-1` | |
| `announceReceiptTimeout` | `3` | `2` to `10` | |
| `domainNumber` | `24` | `24`–`43` | G.8275.1 reserves domain numbers 24–43 |
| `priority2` | `128` | `0`–`255` | For multi-GM redundancy ordering |
| `clock_servo` | `pi` | `pi`, `linreg` | |
| `pi_proportional_const`, `pi_integral_const` | profile defaults | positive float | |
| `step_threshold` | `2.0` s | `> 0` | |
| `first_step_threshold` | `0.00002` s | `> 0` | |
| `tx_timestamp_timeout` | `50` ms | `10`–`1000` | |
| `summary_interval` | `0` | `-7` to `6` | |
| `PtpSchedulingPolicy` | `SCHED_OTHER` | `SCHED_OTHER`, `SCHED_FIFO` | |
| `PtpSchedulingPriority` | — | `1`–`65` | Only valid with `SCHED_FIFO` |
| `inSyncConditionThreshold` | `MaxInSpecOffset` | unsigned int (ns) | |
| `inSyncConditionTimes` | `1` | unsigned int | |

**Additional Configuration Requirements:**
- **GNSS configuration:** The operator shall expose parameters for serial port selection (auto-detected from NIC sysfs by default), ToD protocol (NMEA, UBX), and 1PPS pulse width.
- **Per-port role assignment:** Each NIC port shall be individually configurable as `masterOnly 1` (time transmitter) or `masterOnly 0` (time receiver).
- **DPLL and SyncE parameters:** Reference input priorities and SyncE network option (ITU-T G.8264 Option 1 or Option 2) shall be configurable.
- **Plugin architecture:** Hardware-specific configuration shall be encapsulated in named plugins. The operator shall validate plugin names at admission time.
- **Configuration validation feedback:** When a `PtpConfig` is rejected at admission time, the error message shall identify the violating parameter, its submitted value, and the required value or allowed range.

### B.2 GNSS Reference Acquisition

1. The system shall monitor the GNSS receiver and determine whether a valid
   fix is available. A valid fix is defined as a 3D fix (fix status ≥ 3)
   or better.

2. The system shall monitor the GNSS clock offset (the difference between
   the GNSS-reported time and the receiver's internal clock). If the offset
   exceeds the configured threshold, the GNSS state shall be FREERUN even
   when a valid fix is present.

3. The system shall detect GNSS source loss (transition from fix ≥ 3 to
   fix < 3 or loss of NMEA data) and propagate the source-lost indication
   to the clock state manager within 1 second.

4. The system shall convey Time of Day (ToD) from the GNSS receiver to the
   PHC synchroniser. The ToD shall be used to set the absolute time on the
   PHC at initial startup and to validate ongoing time alignment.

5. The system shall monitor and publish GNSS status at a minimum interval of
   1 second.

### B.3 DPLL Monitoring

1. The system shall monitor the DPLL lock state via the Linux DPLL netlink
   subsystem and report one of: FREERUN, LOCKED, LOCKED_HO_ACQ, or
   HOLDOVER.

2. The system shall monitor DPLL phase offset and compare it against the
   configured in-spec threshold. Phase offset values are reported in
   picoseconds by the DPLL subsystem and shall be converted to nanoseconds
   for comparison and reporting.

3. When the DPLL transitions to HOLDOVER or LOCKED_HO_ACQ following a GNSS
   source loss, the system shall report the DPLL PTP state as HOLDOVER.

4. When the DPLL reports HOLDOVER and the GNSS source is lost, the system
   shall start a holdover timer. The holdover timer duration is configurable
   (see spec.md §6.8).

5. When the DPLL transitions from LOCKED or HOLDOVER back to FREERUN (and
   the source is lost), the system shall report the DPLL PTP state as
   FREERUN.

6. The system shall monitor DPLL frequency status and phase status
   independently. Both must report LOCKED for the DPLL subsystem to be
   considered in the LOCKED state.

7. The system shall poll or subscribe to DPLL status at a minimum interval
   of 1 second.

### B.4 PHC Synchronisation

1. The system shall discipline the PHC to the GNSS-derived 1PPS reference
   signal.

2. The PHC synchroniser shall report its servo state. The servo state
   mapping is:
   - `s0` → FREERUN (unlocked)
   - `s1` → FREERUN (clock step / initial calibration)
   - `s2` → LOCKED (frequency locked, phase aligned)

3. The PHC synchroniser shall report the measured phase offset between the
   PHC and the reference signal at each synchronisation cycle.

4. When the GNSS 1PPS source becomes unavailable (NMEA source timed out,
   invalid timestamps), the PHC synchroniser shall enter holdover mode if
   supported by the servo, or report FREERUN.

5. The PHC synchroniser shall apply a configurable holdover timeout. During
   this period the servo shall maintain the last frequency adjustment. After
   the timeout the servo shall report FREERUN.

6. The PHC synchroniser shall apply a configurable offset threshold
   (`servo_offset_threshold`). If the measured offset exceeds this threshold
   for more than a configurable number of consecutive samples
   (`servo_num_offset_values`), the servo shall not transition to the
   LOCKED (s2) state.

### B.5 PTP Distribution (G.8275.1 Profile)

1. The system shall run the PTP engine in grandmaster mode using the
   ITU-T G.8275.1 telecom profile for full timing support.

2. All PTP ports on the T-GM shall operate in the MASTER role. The T-GM
   shall not enter the SLAVE or PASSIVE state on any port.

3. The PTP engine shall transmit Announce, Sync, and Follow_Up messages on
   all configured ports at the rates defined in the PTP profile
   configuration. Per G.8275.1, the default Announce interval is 8 messages
   per second (logAnnounceInterval = −3) and the default Sync interval is
   16 messages per second (logSyncInterval = −4).

4. The PTP engine shall respond to Delay_Req messages from downstream
   clients with Delay_Resp messages.

5. The PTP engine shall use Layer 2 (Ethernet) multicast transport, per
   G.8275.1.

6. The PTP engine shall use PTP domain 24 (or as configured), per
   G.8275.1 Section 6.2.

7. The PTP engine shall support IEEE 1588 two-step clock operation
   (hardware timestamping with Follow_Up messages).

### B.6 System Clock Synchronisation

1. When configured, the system shall discipline the node system clock
   (`CLOCK_REALTIME`) to the PHC.

2. The system clock synchroniser shall not start until the PHC synchroniser
   has achieved the LOCKED state, to prevent stepping `CLOCK_REALTIME` to
   an un-synchronised PHC.

3. The system clock synchroniser shall report its offset and servo state via
   the same observability interfaces as other subsystems.

### B.7 SyncE / Frequency Distribution (Optional)

1. When SyncE is configured, the system shall distribute frequency from
   the GNSS-locked oscillator via the Ethernet physical layer on all
   configured SyncE ports.

2. The system shall run the ESMC protocol and advertise the appropriate
   quality level (QL) based on the current clock state:
   - LOCKED: QL corresponding to PRTC (e.g., QL-PRC for Option 1 networks)
   - HOLDOVER: QL shall reflect the degraded traceability
   - FREERUN: QL-DNU (Do Not Use) or equivalent

3. The SyncE EEC lock state shall be monitored and reported through the
   observability interfaces.

---

## C. Environment and Deployment

### C.1 Container and Kubernetes Requirements

1. All T-GM software components shall run as containers within a Kubernetes
   pod on the target node.

2. The pod shall have access to the host network namespace to enable
   Layer 2 PTP message exchange.

3. The pod shall have access to required host devices (`/dev/ptpN`,
   `/dev/gnssN`, DPLL netlink) via appropriate Kubernetes device plugins
   or security contexts.

4. The T-GM workload shall be deployed and managed by a Kubernetes operator
   that reconciles PtpConfig custom resources.

### C.2 Configuration Management

1. The T-GM profile shall be configurable via a PtpConfig custom resource
   that specifies:
   - PTP engine configuration (ptp4l options and config)
   - PHC synchroniser configuration (ts2phc options and config)
   - System clock synchroniser configuration (phc2sys options)
   - SyncE configuration (synce4l options and config), if applicable
   - Clock thresholds (max/min offset, holdover timeout)
   - Clock type declaration (`clockType: T-GM`)

2. When the PtpConfig is created or updated, the operator shall apply the
   new configuration to the target node(s) and restart the affected
   processes.

3. The operator shall support node selection via label-based matching rules
   in the PtpConfig `recommend` section.

### C.3 Hardware Configuration

1. The system shall support declarative hardware configuration via a
   HardwareConfig custom resource that specifies:
   - DPLL pin configuration (phase inputs, phase outputs, frequency
     inputs, frequency outputs)
   - Holdover parameters per clock
   - Clock chain structure (subsystems and their relationships)
   - Behaviour rules (source conditions and desired states)

2. When a HardwareConfig is provided, it shall take precedence over
   plugin-derived hardware configuration for the associated PTP profile.

---

## D. Kubernetes Health Probes

The T-GM pod shall define Kubernetes health probes that distinguish between
container health (is the software running?) and clock health (is the clock
synchronised?). These are distinct concerns: a T-GM that is alive but in
FREERUN during cold start is healthy from a container perspective but not
yet a valid time source.

### D.1 Liveness Probe

1. The liveness probe shall verify that the daemon supervisor process is
   responsive (e.g., HTTP 200 on `/healthz` or a process PID check).

2. The liveness probe shall **not** depend on clock synchronisation state.
   A T-GM in FREERUN or HOLDOVER is still alive and shall not be killed
   by Kubernetes.

3. Recommended probe parameters:
   - `initialDelaySeconds`: ≥ 10 (allow process initialisation)
   - `periodSeconds`: 10
   - `failureThreshold`: 3

### D.2 Readiness Probe

1. The readiness probe shall report the pod as **ready** when the daemon
   supervisor is initialised and capable of serving metrics and events,
   regardless of clock state.

2. The readiness probe shall **not** gate on clock LOCKED state. Rationale:
   GNSS acquisition and DPLL lock can take several minutes on cold start;
   gating readiness on LOCKED would leave the pod unready for an extended
   period, preventing metrics scraping and event subscription during the
   critical initial convergence window.

3. Clock synchronisation readiness shall instead be conveyed via:
   - The `openshift_ptp_clock_state` Prometheus metric
   - The `event.sync.ptp-status.ptp-state-change` CloudEvent
   - The `PtpConfig.status` conditions on the custom resource

4. Consumers that need to gate on clock quality (e.g., a T-BC that should
   not lock to an unqualified upstream) shall use the `clockClass` in
   Announce messages or the BMCA, not the pod readiness state.

### D.3 Startup Probe

1. A startup probe shall be configured to allow extended initialisation
   time for GNSS acquisition on cold start.

2. The startup probe shall verify the same condition as the liveness probe
   (supervisor process responsive) but with a longer failure budget.

3. Recommended probe parameters:
   - `periodSeconds`: 10
   - `failureThreshold`: 30 (allows up to 5 minutes for GNSS cold start)

4. While the startup probe is running, the liveness probe shall be
   suspended (standard Kubernetes behaviour).

### D.4 Health Probe Functional Requirements

| Requirement ID | Traceability | Requirement text |
| :---- | :---- | :---- |
| FUNC-TGM-EXTRA-R1 | D.1 | Given a T-GM pod is running, then the liveness probe must verify that the daemon supervisor process is responsive, independent of clock synchronisation state |
| FUNC-TGM-EXTRA-R2 | D.1.2 | Given a T-GM is in FREERUN or HOLDOVER state, then the liveness probe must still report healthy (the pod must NOT be killed) |
| FUNC-TGM-EXTRA-R3 | D.2 | Given a T-GM pod has initialised, then the readiness probe must report ready when the supervisor is capable of serving metrics and events, regardless of clock state |
| FUNC-TGM-EXTRA-R4 | D.2.2 | Given a T-GM pod is starting up, then the readiness probe must NOT gate on clock LOCKED state |
| FUNC-TGM-EXTRA-R5 | D.3 | Given a T-GM pod is starting on a cold node (GNSS cold start), then the startup probe must allow at least 5 minutes of initialisation time before liveness checks begin |

---

## E. Security

1. The system shall support PTP authentication (IEEE 1588 Annex P / NTS)
   when configured, using key material supplied via Kubernetes Secrets
   mounted into the pod.

2. When authentication key material changes (Secret update), the system
   shall detect the change and restart affected processes to pick up the
   new keys without requiring a pod restart.

---

## F. Validation Requirements

### F.1 Pre-conditions

Before T-GM validation tests can execute, the following pre-conditions must
hold:

1. The target node has PTP-capable hardware with GNSS receiver, DPLL, and
   PHC.
2. The PtpConfig custom resource with `clockType: T-GM` is applied and
   reconciled.
3. All T-GM processes (GNSS monitor, PHC synchroniser, PTP engine) are
   running (`process_status` = 1).
4. The GNSS receiver has a valid fix (fix status ≥ 3).
5. The DPLL is in the LOCKED state.
6. The PHC synchroniser servo state is s2 (LOCKED).

### F.2 Functional Tests

| ID | Requirement section | Test description | Pass criteria |
| :--- | :--- | :--- | :--- |
| F-01 | spec.md §6.5 T2 | Verify lock acquisition from cold start | After initial deployment, `clock_state` transitions to LOCKED (1) and `clock_class` to 6 within the expected lock-up time |
| F-02 | spec.md §6.5 T3 | Simulate GNSS loss and verify holdover entry | After GNSS signal removal, `clock_class` transitions from 6 → 7 within one Announce interval; `clock_state` = HOLDOVER (2) |
| F-03 | spec.md §6.5 T6 | Verify holdover in-spec to out-of-spec transition | After holdover timeout or offset exceeding `MaxInSpecOffset`, `clock_class` transitions from 7 → 140 |
| F-04 | spec.md §6.5 T7/T9 | Verify holdover to freerun transition | After holdover expiry, `clock_class` transitions to 248; `clock_state` = FREERUN (0) |
| F-05 | spec.md §6.5 T5/T8 | Verify re-lock after GNSS recovery | After GNSS signal restoration, `clock_class` transitions back to 6; `clock_state` = LOCKED (1) |
| F-06 | spec.md §8.1–8.4 | Verify Announce attribute correctness per state | For each clock state, confirm `clockClass`, `clockAccuracy`, `timeSource`, `timeTraceable`, `frequencyTraceable`, `offsetScaledLogVariance` match the table in spec.md §8.5 |
| F-07 | spec.md §8.5 | Verify UTC offset and leap second handling | Confirm `currentUtcOffset` is correct and `currentUtcOffsetValid` is true |
| F-08 | B.5 | Verify all ports are MASTER | All configured PTP interfaces report `interface_role` = 2 (MASTER) |
| F-09 | spec.md §9.2 step 5 | Verify phc2sys delayed start | Confirm system clock synchroniser does not start until PHC synchroniser reports LOCKED |
| F-10 | spec.md §12.5 | Verify CloudEvent on state change | On each state transition, a CloudEvent of the correct type is published within 1 second |
| F-11 | spec.md §9.3 | Verify process restart on crash | Kill a synchronisation process; confirm it restarts automatically and `process_restart_count` increments |
| F-12 | E | Verify security key rotation | Update the authentication Secret; confirm processes restart and new keys are applied without pod restart |
| F-13 | D.1 | Verify liveness probe does not depend on clock state | With T-GM in FREERUN or HOLDOVER, confirm liveness probe reports healthy |
| F-14 | D.2 | Verify readiness probe does not gate on LOCKED | During cold start before GNSS lock, confirm readiness probe reports ready once supervisor is initialised |
| F-15 | D.3 | Verify startup probe allows GNSS cold start | On cold start, confirm liveness probe is suspended and startup probe allows at least 5 minutes of initialisation |

### F.3 Performance Tests

| ID | Requirement section | Test description | Pass criteria |
| :--- | :--- | :--- | :--- |
| P-01 | spec.md §15.1 | Measure locked-mode time error | PHC-to-GNSS offset remains within ±100 ns (PRTC-A) or ±40 ns (PRTC-B) over a 24-hour measurement window |
| P-02 | spec.md §15.2 | Measure holdover drift | After GNSS loss, measure time error drift; verify it remains within `MaxInSpecOffset` for at least `LocalHoldoverTimeout` seconds (hardware-dependent) |
| P-03 | spec.md §15.3 | Verify Sync message rate | Capture PTP traffic; confirm Sync messages are transmitted at the configured rate (default 16/s) on each port |
| P-04 | spec.md §15.3 | Verify Announce message rate | Capture PTP traffic; confirm Announce messages are transmitted at the configured rate (default 8/s) on each port |
| P-05 | spec.md §6.7 | Measure state transition latency | From GNSS loss event to `clock_class` change in Announce: ≤ 1 Announce interval + 1 second |
