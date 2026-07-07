This repository contains PTP clock requirements per clock type.
Each clock type has a separate directory that contains at least `spec.md`.

| Directory | Clock Type | Description |
| :--- | :--- | :--- |
| `tbc/` | T-BC | Telecom Boundary Clock — receives upstream, distributes downstream |
| `ttsc/` | T-TSC | Telecom Time Synchronous Clock — time receiver only, subset of T-BC |
| `tgm/` | T-GM | Telecom Grandmaster — primary time source (GNSS-disciplined) |
