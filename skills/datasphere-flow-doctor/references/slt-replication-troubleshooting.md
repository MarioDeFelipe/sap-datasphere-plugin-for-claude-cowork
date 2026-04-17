# SLT-based Replication Flow Troubleshooting

End-to-end troubleshooting for Replication Flows that move **ABAP tables** from on-premise SAP systems to SAP Datasphere via **SAP LT Replication Server (SLT)**. Covers DMIS 2018 SP06+ and DMIS 2020 SP03+.

> **Context**: For CDS view replication troubleshooting, see `cds-replication-architecture.md` in `datasphere-s4hana-import`. For ODP (BW context or SAPI) see `odp-replication-troubleshooting.md`. This reference focuses on the **SLT path**.
>
> **Correction note** — earlier documentation in `datasphere-data-flows/SKILL.md` (table of source connectivity options) described SLT as "Legacy - supported but CDS preferred". By the way, that was incorrect. SLT is the **recommended** approach for **ABAP table** replication; CDS is preferred for **CDS view** replication. Both are current and strategic.

---

## Architectural Overview

```
[Source ABAP / S/4HANA]                 [SAP LT Replication Server]      [Datasphere / RMS]
   Source table ── SLT triggers ─▶ Logging tables ─▶ RDB buffer ─(CC)─▶ RMS ─▶ Local table
                                                     (DHRDB/LTRDB)
```

Key building blocks:
- **RMS** (Replication Management Service) — Data Intelligence component inside Datasphere.
- **RDB** (Resilient Data Buffer) — buffer framework in the ABAP source that isolates the source from network/RMS transient issues.
- **Cloud Connector** — secure tunnel. Resource list must expose specific prefixes (see below).
- **ABAP Pipeline Engine (APE)** — executes the ingestion operator that drains the RDB.

Buffer read flow (steady state):
1. Source transaction fires trigger → logging table in SLT.
2. SLT job pushes delta to **RDB buffer table**.
3. RMS reads RDB package, commits, and deletes the package from the buffer.

---

## Preparation Checklist

### 1. SLT system prerequisites

- **DMIS 2018 SP06** or higher, **or DMIS 2020 SP03** or higher.
- Run the **Note Analyzer** (SAP Note 3016862, transaction `CNV_NA_DI`) and apply all notes required by **SAP Note 2890171** (central note for ABAP Integration) and **SAP Note 3297105** (considerations & limitations).
- **Recommended deployment**: standalone SAP LT Replication Server 3.0 (DMIS 2018) or SAP LT Replication Server for S/4HANA 1.0 (DMIS 2020). Embedded SLT inside S/4HANA cannot be upgraded independently and is not recommended. **SAP LT Replication Server 2.0 (DMIS 2011)** is explicitly **not** recommended — it is NetWeaver 7.00-based, lacks features required for RMS and Gen-2 operators, and does not receive new functionality.

### 2. Communication user (SU01)

Create a **Dialog user** (for example `DSPREMOTE`) with:
- `SAP_IUUC_REPL_REMOTE` — SLT Connector role (always required).
- `SAP_DH_CDC_REMOTE` — required for S/4HANA on-premise sources of release **2020 or higher**.
- Additional roles per **SAP Note 3100673** (Security Settings). See `datasphere-security-architect/references/abap-remote-user-authorizations.md`.

Do not use `DDIC` for the RFC.

### 3. RFC destination (SM59)

Type **3 (ABAP)** from the SLT system to the source ABAP / S/4HANA system.
- If both sides are Unicode, mark the RFC as Unicode.
- If source and SLT are the same system (embedded SLT in S/4HANA), still create a real RFC — **do not** use the `NONE` option.
- Verify with the "Connection Test" button.

### 4. LTRC configuration

Transaction **`LTRC`** → **Create Configuration**:
1. **RFC Connection** = the type-3 RFC from step 3.
2. **Scenario** = **Other** → **"SAP Data Intelligence (Replication Management Service)"**.
3. **Replication Option** = **Real Time**.
4. Job Options — these drive SLT parallelism:
   - **No. of Data Transfer Jobs** — jobs moving data from logging tables to target.
   - **No. of Initial Load Jobs** — jobs for initial extraction.
   - **No. of Calculation Jobs** — for Resource-Optimized, calculates portions for initial load; for Performance-Optimized, transfers portions to `DMC_INDXCL`.
5. Note the **Mass Transfer ID (MTID)** — you need it as the source container in Datasphere.

Until you deploy the Replication Flow in Datasphere the **Participating Objects** tab will be empty. Objects appear only after the Datasphere pipeline starts subscribing.

### 5. Cloud Connector

In Cloud Connector → Cloud To On-Premise → Access Control → add the backend and the following resources:

| Target system type | Required resources |
|--------------------|--------------------|
| **SAP S/4HANA on-premise** (embedded SLT) | `DHAMB_` (prefix), `DHAPE_` (prefix), `RFC_FUNCTION_SEARCH` (exact) |
| **Standalone ABAP with DMIS add-on** | `LTAMB_` (prefix), `LTAPE_` (prefix), `RFC_FUNCTION_SEARCH` (exact) |

Verify via the first button on the resource list.
If verification fails, check **KBA 3369433** and **KBA 3456850**, and collect logs per **KBA 3449529** before opening a case under component `DS-DI-CON`.

### 6. Datasphere connection

- Connection type: **ABAP** (for DMIS systems) or **SAP S/4HANA On-Premise** (for embedded SLT).
- After saving, **Validate** the connection — it must say **"Replication flows are enabled"** for Replication Flows to work.
- Also check that `Data Flows enabled`, `Remote tables`, and `Model Import` are healthy.

---

## Building the Replication Flow

1. **Data Builder → New Replication Flow**.
2. **Source Connection** — the ABAP/S4 connection created above.
3. **Source Container** — **only `/SLT/Mass Transfer ID`** is valid for SLT. Use the MTID recorded in `LTRC`.
4. **Source Objects** — type the **table name** (tables are not browsable in advance) and click **Next**.
5. **Target Connection** — typically Datasphere Local Table.
6. **Load Type**:
   - **Initial Only** — full load, no CDC.
   - **Initial and Delta** — full load, then continuous change replication.
7. **TRUNCATE** — optional flag to clear the target before (re)initial load.
8. **Save → Deploy → Run**.

While clicking **Browse Container / Objects**, the backend fires (verifiable via `ST05`):
- `DDIF_FIELDINFO_GET`
- `RFC_FUNCTION_SEARCH`
- `RFC_GET_FUNCTION_INTERFACE`
- `DHAMB_*` / `DHAPE_*` (S/4HANA) or `LTAMB_*` / `LTAPE_*` (DMIS)

If you see an error at that step, **first** re-check the Cloud Connector resource list. If no objects appear under "SLT - SAP LT Replication Server" container, check the communication user's authorizations — **KBA 3501459**.

> **Capacity caveat** — a Replication Flow with any object in `Initial and Delta` mode has **no end date**. Once started it runs until stopped/paused/errored. Each delta operation reserves a worker node, consuming Data Integration capacity units. See `Configure the Size of Your SAP Datasphere Tenant` and **KBA 3481365**.

---

## Runtime Monitoring

### Datasphere side

**Data Integration Monitor** → pick the Replication Flow →
- **Run Status**
- **Object Status**
- **Messages**
- **Metrics**

For `Initial and Delta` loads the Run Status cycles through `ACTIVE (RETRYING OBJECTS)` between deltas and the Object Status cycles `INITIAL RUNNING → RETRYING → DELTA RUNNING → RETRYING`. That cycling is **normal** — do not raise it as an error.

### SLT / source side

| Transaction | Purpose |
|-------------|---------|
| `LTRC` → Participating Objects | Table appears only after RF pipeline starts subscribing |
| `LTRC` → Application Logs | Activation / config errors per object |
| `LTRC` → Statistics | Row counts, throughput, timing |
| `LTRDBMON` | **RDB monitor on DMIS (standalone SLT) systems** — buffer table status |
| `DHRDBMON` | **RDB monitor on S/4HANA** systems (embedded scenario) |
| `SM37` | Background job monitor for the SLT jobs below |

> **Correction note** — earlier content implied `DHRDBMON` was the universal RDB monitor. By the way, that was incomplete: **LTRDBMON** is the correct transaction on DMIS-only standalone SLT systems. Use `DHRDBMON` only when embedded in S/4HANA.

### SLT SM37 job names

Each extraction step is a distinct job — useful for narrowing down where a run hangs.

| Job name pattern | Purpose |
|------------------|---------|
| `/1LT/IUC_REP_CNTR_<MTID>` | **Master Controller** — creates DB triggers and logging tables |
| `/1LT/IUC_DEF_COBJ_<MTID>` | **Migration Object Definition** — defines the migration object for a specific table |
| `/1LT/IUC_CALC_<MTID>_nn` | **Access Plan Calculation** — partitions the initial load |
| `/1LT/IUC_LOAD_MT_<MTID>_nnn` | **Data Load Job** — executes initial load and delta replication |

For **CDS view** replication (not SLT) the CDC jobs are instead `/1DH/OBSERVE_LOGTAB` (observer) and `/1DH/PUSH_CDS_DELTA` (transfer) — see `abap-side-monitoring.md`.

---

## Troubleshooting Flow

1. **Connection / setup problems** → re-verify LTRC config (step 4) and Cloud Connector resources (step 5). KBA **3369433**.
2. **Runtime errors, no detail in DI Monitor** → `LTRC` → Participating Objects → drill into Application Logs and Statistics tabs.
3. **Buffer health** → `LTRDBMON` (DMIS) / `DHRDBMON` (S/4HANA).
4. **Performance**:
   - Datasphere **System Monitor Dashboard**.
   - **HANA Cockpit** (CPU / memory).
   - HANA `indexserver` trace for memory-allocation errors — access per **KBA 3476918**.
   - See **SAP Note 3360905** (RMS / Replication Flow performance) and **SAP Note 3489681** (low performance with Datasphere local tables as target).
5. **Still blocked** → open a case under component **DS-DI-RF**, attach LTRC logs and monitor screenshots.

---

## SAP Note / KBA Reference Index

| Note / KBA | Topic |
|------------|-------|
| 2890171 | Central note: SAP Data Intelligence / SAP Datasphere - ABAP Integration |
| 3297105 | Important considerations & limitations for Replication Flows |
| 3100673 | Security settings / remote user authorizations |
| 3016862 | DMIS Note Analyzer (CNV_NA_DI) scenarios |
| 3498095 | How to use the Note Analyzer |
| 3369433 | Cloud Connector troubleshooting |
| 3456850 | Cloud Connector resource list issues |
| 3449529 | Data collection for DS-DI-CON incidents |
| 3476918 | How to access HANA Cloud DB traces |
| 3360905 | RMS / Replication Flow performance |
| 3489681 | Low performance with local tables as RF target |
| 3481365 | Initial and Delta RF: no end date / long-running data-load job |
| 3501459 | Communication user authorization issue (CDS/objects not visible) |
