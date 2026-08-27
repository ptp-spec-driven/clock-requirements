This repository contains PTP clock requirements per clock type.
Each clock type has a separate directory that contains at least `spec.md`.

The T-BC spec (`tbc/spec.md`) is the **living single source of truth** for PTP operator T-BC behavior. At GA (release 5.1), implementation must be compliant with §14/§15 or document public gaps ([tbc/gaps.md](tbc/gaps.md)).

| Directory | Clock Type | Description |
| :--- | :--- | :--- |
| `tbc/` | T-BC | Telecom Boundary Clock — receives upstream, distributes downstream |
| `ttsc/` | T-TSC | Telecom Time Synchronous Clock — time receiver only, subset of T-BC |
| `tgm/` | T-GM | Telecom Grandmaster — primary time source (GNSS-disciplined) |
