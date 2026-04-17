# ODP-based Replication Flow Troubleshooting

End-to-end troubleshooting for Replication Flows that use the **ODP (Operational Data Provisioning)** framework. Two sub-scenarios are covered:

1. **ODP with BW Context** — source is a BW object (InfoObject, InfoCube, ADSO, Composite Provider, Query as InfoProvider).
2. **ODP with SAPI Context** — source is an SAP standard extractor/DataSource released for ODP.

> **Context**: For CDS view replication see `datasphere-s4hana-import/references/cds-replication-architecture.md`. For SLT table replication see `slt-replication-troubleshooting.md`.

---

## Shared Architecture

```
[Source]                 [ABAP source system]                 [Datasphere / RMS]
  BW or SAPI ──▶ Subscriber ──▶ ODQ (Operational Delta Queue) ──ODP API──▶ RMS ──▶ Local table
```

- **ODQ** = Operational Delta Queue. Monitored and managed via transaction **`ODQMON`**.
- A **Subscriber** is created for the source provider in ODQ; RMS pulls via the ODP API written for RMS.
- For BW context specifically: extractors load BW InfoProviders (e.g., via Process Chains or DTPs); the data is then pushed to the ODQ.
- Even though a CompositeProvider is logical, its **delta capability comes from the underlying physical provider** — changes are tracked there and surfaced via the ODP framework.

Two **Load Types** are supported in both sub-scenarios:
- **Initial Only**
- **Initial and Delta** (status cycles `ACTIVE (RETRYING OBJECTS)` between deltas — normal)

---

## Version Prerequisites

| Component | Minimum version |
|-----------|-----------------|
| SAP S/4HANA on-premise as source | **1909** |
| SAP BW / BW/4HANA as source | **BW 7.55** (SAP_BASIS 7.55) |
| DMIS (standalone ABAP-based source for SAPI) | **2011 SP23 / 2018 SP08 / 2020 SP04** or higher |

Run the **Note Analyzer** on the source system (**SAP Note 3016862**, transaction `CNV_NA_DI`) to verify all ABAP-integration notes are applied.

### Key correction notes

| Note | Scope |
|------|-------|
| **2890171** | Central note — ABAP Integration; list of corrections |
| **3297105** | Considerations & limitations for Replication Flows |
| **3100673** | Security settings / remote user authorizations |
| **3412110** | Improvements for the replication and initial load of ODP data sources |
| **3232522** | ODP API for Gen2 and RMS - TCI for S/4HANA 2021 |
| **3232559** | ODP API for Gen2 and RMS - TCI for S/4HANA 2020 |
| **3234938** | ODP API for Gen2 and RMS - TCI for S/4HANA 1909 |
| **3225712** | Collective note for S/4HANA 2022 ABAP Integration (Gen2 & RMS) |
| **2830276** | SAP Data Intelligence - ABAP Integration - S/4HANA 1909 |
| **3156672** | SAP Data Intelligence - ABAP Integration - DMIS 2011 SP23 |
| **3156649** | SAP Data Intelligence - ABAP Integration - DMIS 2018 SP08 / 2020 SP04 |
| **2232584** | **Release of SAP extractors for ODP replication (ODP SAPI)** |
| **3486245** | FAQ: RFC Fast Serialization in Replication Flow |
| **3339368** | FAQ: Push DataSource Delta Extraction via ODP/SAPI |

Fatal `Replication runs will restart ... Aborting the graph` errors are typically caused by **missing notes from 2890171**. Use the Note Analyzer (**KBA 3498095**) first.

---

## Scenario 1 — ODP with BW Context

### Building the flow

- **Source Connection**: SAP ABAP or SAP S/4HANA On-Premise.
- **Source Container**: **`ODP_BW Extraction`** folder.
- **Source Object**: BW object (ADSO, InfoCube, Composite Provider, Query as InfoProvider, etc.).
- **Target**: Datasphere (or any supported target).

### Runtime monitoring (BW-specific)

| Where | Tool | What to look at |
|-------|------|-----------------|
| Datasphere | Data Integration Monitor | Run, Object, Message, Metrics |
| Source | **`ODQMON`** | Records moved, subscriber status |
| Source | **`SLG1` — Object: `ODQ`** | Runtime error details |
| Source | **`RODPS_REPL_TEST`** on the corresponding SQL view | Ad-hoc extraction test outside Datasphere; then verify row count in `ODQMON` |

> **Important** — for BW context the SLG1 object is **`ODQ`**, not `DHAPE`/`DHCDC` (which are for CDS) and not `LT*` (which are for SLT).

### Component routing (BW specifics)

- `RODPS_REPL_TEST` returns error → component **`BW-WHM-DBA-ODA`**.

---

## Scenario 2 — ODP with SAPI Context

### Building the flow

- **Source Connection**: SAP ABAP or SAP S/4HANA On-Premise.
- **Source Container**: **`ODP_SAPI - ODP Context: SAPI`** folder.
- **Source Object**: an SAP extractor/DataSource released for ODP (per **SAP Note 2232584**).
- **Target**: Datasphere.

Push vs. Pull — how a SAPI DataSource writes delta after initialization depends on the extractor itself. See **KBA 3339368**.

### Source Objects not visible in the container?

1. First follow **KBA 3489773** (ODP SAPI source not appearing for Replication Flows).
2. If the problem persists, on the source run transaction **`RSA9`**:
   - Answer **Yes** to the popup: *"Do you want the content application Transfer Component Hierarchy?"*
   - Background: **KBA 2205577** ("Application component does not exist when activating DataSource in RSA5: Error R8418").
3. Verify the communication user's authorizations (Security Note **3100673**).

### Runtime monitoring (SAPI-specific)

- `ODQMON` on the source to verify subscriber + row counts.
- `RODPS_REPL_TEST` against the **ODP SAPI DataSource** object for an ad-hoc test outside Datasphere.
- `SLG1` on the source for runtime error details.

### Performance

- Datasphere **System Monitor Dashboard**.
- **HANA Cockpit** for CPU/memory.
- HANA **`indexserver`** log via HANA SQL Console — **KBA 3476918**.
- For RFC Fast Serialization tuning see **KBA 3486245**.
- General RMS / Replication Flow performance — **SAP Note 3360905**.

---

## Common Errors and First Steps

| Symptom | First check |
|---------|-------------|
| `Replication runs will restart ... Error: Fatal error found. Aborting the graph.` | Note Analyzer (`CNV_NA_DI`) — missing notes from 2890171. KBA 3498095 |
| Connection validation doesn't say `Replication flows are enabled` | Cloud Connector resources, KBA 3369433 |
| Source Objects not visible in `ODP_SAPI` container | KBA 3489773, then `RSA9` (Transfer Component Hierarchy), then KBA 2205577 |
| `NO_AUTHORIZATION` / missing auth objects | SU53 on source; Security Note 3100673; see `abap-remote-user-authorizations.md` |
| Application log shows `ACP daemon not start` (CDS-side analog) | Component `BC-DB-CDC` (CDS scenarios only) |

---

## Component Routing Cheat-Sheet

| Symptom | Component |
|---------|-----------|
| Connection validation / Cloud Connector | **DS-DI-CON** |
| Replication Flow runtime | **DS-DI-RF** |
| `RODPS_REPL_TEST` error (BW) | **BW-WHM-DBA-ODA** |
| CDC engine issues (CDS path, e.g. 'ACP daemon not start' in `DHCDCMON`) | **BC-DB-CDC** |
