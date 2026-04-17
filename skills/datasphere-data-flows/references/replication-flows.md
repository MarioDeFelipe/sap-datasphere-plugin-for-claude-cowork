# Replication Flows Reference

## Source Configuration

### S/4HANA CDS Views

**Required Annotations for Delta:**
```abap
@Analytics.dataExtraction.enabled: true
@Analytics.dataExtraction.delta.changeDataCapture: true
```

**Common CDS Views:**
| CDS View | Description | CDC Support |
|----------|-------------|-------------|
| I_ACDOCA | Universal Journal | Yes |
| I_BSEG | Accounting Line Items | Yes |
| I_PURCHASEORDER | Purchase Orders | Yes |
| I_SALESORDER | Sales Orders | Yes |

### Connection Requirements
- ABAP connection with Cloud Connector
- User with extraction authorization
- RFC destination configured

## Target Configuration

### Local Table
- Best for: In-memory analytics
- Enable "Delta Capture" for downstream Transformation Flows
- Storage: Uses Space memory quota

### Local Table (File) - Object Store
- Best for: Large volumes, warm/cold data
- Format: Parquet (default), CSV
- Storage: HANA Data Lake Files (HDLF)
- Performance: Slower than in-memory

### External Targets (POI Required)

#### Amazon S3
```
Bucket: s3://your-bucket
Path: /datasphere/exports/
Format: Parquet (recommended) or CSV
```

#### Azure Data Lake Gen2
```
Container: your-container
Path: /datasphere/exports/
Format: Parquet (recommended) or CSV
```

#### Kafka
```
Bootstrap Servers: kafka:9092
Topic: datasphere-events
Format: JSON or Avro
```

## Load Types

### Initial Load Only
- Full extraction on each run
- Use when: Source lacks CDC, one-time migration

### Initial + Delta
- First run: Full extraction
- Subsequent runs: Only changed records
- Requires: CDC-enabled source

## Monitoring

### Data Integration Monitor
Path: Left Menu → Data Integration Monitor

**Key Metrics:**
- Records transferred
- Duration
- Error count
- Delta queue status

### Troubleshooting Failed Runs
1. Check connection status
2. Verify CDC annotations
3. Review error logs in monitor
4. Check Cloud Connector (on-prem sources)
5. Verify POI block availability (external targets)

## What's New (2026.05)

### Improved Primary Key Order Handling
During table replication, the primary key order from the source is now preserved in the target table. Previously, mismatches in primary key column ordering between source and target could cause replication failures. This fix is automatic and requires no configuration changes. If you previously encountered key order mismatch errors, re-deploying the replication flow should resolve the issue.

---

## Source-Type Matrix for ABAP-based Sources

Replication Flows expose ABAP data through four distinct paths, each with its own source container name in the Datasphere modeler:

| Source Path | Source Container in RF | When to use | Key transactions (source) |
|-------------|-----------------------|-------------|---------------------------|
| **SLT** (ABAP tables via SAP LT Replication Server) | `/SLT/<Mass Transfer ID>` | Replicating raw ABAP tables from on-premise; recommended for table-based replication | `LTRC`, `LTRDBMON` (DMIS) / `DHRDBMON` (S/4HANA), `SM37` for `/1LT/IUC_*` |
| **CDS Views** (S/4HANA on-premise with CDC engine) | `CDS_EXTRACTION` | Replicating standard or custom CDS views with Delta | `SDDLAR`, `DHCDCMON`, `DHRDBMON`, `SLG1` (DHAPE/DHCDC) |
| **ODP / BW Context** | `ODP_BW Extraction` | Replicating BW objects (ADSO, InfoCube, Composite Provider, Query) | `ODQMON`, `RODPS_REPL_TEST`, `SLG1` (**object ODQ**) |
| **ODP / SAPI Context** | `ODP_SAPI - ODP Context: SAPI` | Replicating SAP standard extractors/DataSources released for ODP | `ODQMON`, `RODPS_REPL_TEST`, `RSA9`, `SLG1` (**object ODQ**) |

**Version minimums**:
- SLT: **DMIS 2018 SP06+** or **DMIS 2020 SP03+** (standalone SLT recommended; DMIS 2011 not recommended).
- CDS: central notes per SAP Note 2890171.
- ODP: S/4HANA **1909+** or BW **7.55+**; DMIS **2011 SP23 / 2018 SP08 / 2020 SP04+** for SAPI.

**Cloud Connector resources** (Cloud To On-Premise → Access Control):

| Target system type | Required resources |
|--------------------|--------------------|
| S/4HANA on-premise (incl. embedded SLT) | `DHAMB_` (prefix), `DHAPE_` (prefix), `RFC_FUNCTION_SEARCH` (exact) |
| ABAP system with DMIS add-on (standalone SLT) | `LTAMB_` (prefix), `LTAPE_` (prefix), `RFC_FUNCTION_SEARCH` (exact) |

Connection validation must read **"Replication flows are enabled"** before the flow can use it.

> See `datasphere-flow-doctor/references/slt-replication-troubleshooting.md` for SLT end-to-end setup and `datasphere-flow-doctor/references/odp-replication-troubleshooting.md` for ODP BW/SAPI setup.

---

## Load Type Scenarios — Summary

Full behavioural walkthrough is in `datasphere-flow-doctor/references/load-type-scenarios.md`. Summary:

| Scenario | Load Type | TRUNCATE | Source change | Behaviour on target |
|----------|-----------|----------|---------------|---------------------|
| A | Initial Only | No  | Delete source row | **Target keeps the row** (Initial Only never deletes; re-inserts avoid duplicates) |
| B | Initial Only | Yes | — | Target is cleared, then refilled with current source rows |
| C | Initial and Delta | No  | Add rows | Initial Load runs once (on deploy); subsequent deltas add rows with `Change Type = A` |
| D | Initial and Delta | No  | Delete source row | Target row is flipped to `Change Type = D` (not physically deleted) |
| E | Initial and Delta | No  | Delete **target** row | Next delta **re-inserts** the row with `Change Type = L` |
| F | Initial and Delta | Yes | — | Target is cleared, then fully re-initialised; delta continues as normal |

### Change Types (visible on target when Delta Capture is enabled)

| Code | Meaning |
|------|---------|
| `L` | **Initial Load** |
| `A` | **Upsert** (insert or update during delta) |
| `D` | **Delete** (row marked deleted; not physically removed) |

### Modelling pitfalls

- **"Delta Capture Off cannot be re-enabled"** — once a target is deployed with `Delta Capture = Off`, there is no way to switch it on later. Drop the target and redeploy, or model a new target. Turn Delta Capture on up front whenever `Initial and Delta` is (or might be) on the roadmap.
- **Every redeploy triggers an Initial Load** — even for `Initial and Delta` flows. Plan accordingly for high-volume tables.
- **TRUNCATE is not dynamic** — must be set before starting/restarting the run.
- **`Initial and Delta` flows have no end date** — the data-load job stays `Active` continuously (`SAP Note 3481365`). Each delta operation reserves a worker node and consumes Data Integration capacity.
