# Replication Flow Error Patterns and Resolutions

## Architecture Context
Brief explanation: Replication Flows use RMS (Replication Management Service) collecting data from RDB (Resilient Data Buffer) tables via Cloud Connector. For "Initial and Delta" loads, the CDC Engine pushes changes through Master Logging Tables → Subscriber Logging Tables → RDB Buffer → RMS → Datasphere local table. Two load types: Initial Only (source table → RDB) and Initial and Delta (source table → CDC → RDB).

## Error Pattern 1: "Error occurred during execution of API activity; see application log" — Authorization
### Symptoms
- Error in Data Integration Monitor Messages tab
- SLG1 shows: "Not authorized to use operator 'internal.inport' (BADI_DHAPE_OPER_INPORT)" or "Not authorized to use operator 'com.sap.abap.operator_reader' (BADI_DHAPE_OPER_OPER_READER)"

### Root Cause
Missing security authorizations on the source ABAP system for the communication user

### Resolution
Apply SAP Note 3100673 — SAP Data Intelligence / SAP Datasphere - ABAP Integration - Security Settings. Assign required authorizations to the RFC communication user.

### Diagnostic Steps
1. Check SLG1 with Object DHAPE for detailed authorization error
2. Run SU53 for the communication user to see failed auth checks
3. Use STAUTHTRACE to trace the exact missing authorization objects

## Error Pattern 2: "Subscription interface error / Error while processing subscription of CDS view"
### Symptoms
- Deployment or initial run fails
- Run Log states CDS view definition is "complex or inconsistent"

### Root Cause
CDS view definition has issues — missing annotations, complex joins, unsupported features, or inconsistent metadata

### Resolution
1. Run SDDLAR on source system to validate CDS view (Check DDL Source, Data Preview)
2. Verify required annotations: @Analytics.dataExtraction.enabled: true
3. For delta: verify @Analytics.dataExtraction.delta.changeDataCapture annotation
4. Check SAP Note 2890171 for CDS view requirements in Replication Flow scenarios

### Diagnostic Steps
1. Open SLG1 with Object DHCDC for subscription errors
2. Test extraction independently with report RODPS_REPL_TEST
3. Validate CDS view in SDDLAR

## Error Pattern 3: "No source partition is available. Replication Flow will restart."
### Symptoms
- Data Integration Monitor shows State = "Partitioning Initial Load", Status = Active (Retrying)
- Flow keeps restarting without making progress

### Root Cause
Communication user lacks authorization to access the CDS view for subscriber RMS

### Resolution
1. Check SLG1 Object DHCDC for the specific authorization failure
2. Grant the required authorizations to the RFC user for the CDS view
3. Ensure S_SDSAUTH with ACTVT=16 (Execute) is assigned

### Diagnostic Steps
1. Check SLG1 for DHCDC entries at the time of the partitioning attempt
2. Run SU53 for the communication user immediately after the failure
3. Verify the user has S_SDSAUTH and SDDLVIEW authorization objects

## Error Pattern 4: "Failed because maximum number of deletions retries reached"
### Symptoms
- Multiple CDS views in a single Replication Flow
- Several views fail during replication
- Run Log states "CDS view definition is complex or inconsistent"

### Root Cause
When a Replication Flow contains many CDS views, failures in individual views can cascade. The deletion retry mechanism exhausts after repeated attempts.

### Resolution
1. Identify which specific CDS views are failing (check per-object status in Data Integration Monitor)
2. Validate each failing CDS view individually with SDDLAR and RODPS_REPL_TEST
3. Consider splitting large Replication Flows into smaller ones with fewer objects
4. Fix the "complex or inconsistent" CDS view definitions on the source system

## Error Pattern 5: "Cannot determine tables for CDS view"
### Symptoms
- Error during API activity execution
- SLG1 shows message DHCDC_CORE028 "Cannot determine tables for CDS view <name>"

### Root Cause
System cannot determine the underlying database tables for data extraction from the CDS view

### Resolution
See SAP KBA 3397020 — Check the @Analytics.dataExtraction annotation implementation and correct any errors or warnings. Restart extraction after fixing.

### Diagnostic Steps
1. Open SLG1, find entry with Object DHCDC, expand error details
2. Check long text for Message No. DHCDC_CORE028
3. In SDDLAR, verify CDS view can resolve to underlying tables
4. Check if annotation @Analytics.dataExtraction is correctly implemented

## Error Pattern 6: "Partitioning for object failed"
### Symptoms
- Random failures during replication, especially for larger datasets
- May work sometimes and fail other times

### Root Cause
Missing corrections in the source system

### Resolution
Apply SAP KBA 3465112 — Replication Flow of CDS views randomly fails with "Partitioning for Object failed". Implement the referenced SAP Notes on the source system.

## Error Pattern 7: DATA_NOT_READY — Buffer Table Empty or Low
### Symptoms
- Replication stalls, no data flowing
- DHRDBMON shows buffer table empty or very low record count
- No READY packages in Package Overview

### Root Cause (CDS views via CDC engine)
CDC Observer/Transfer jobs not running or not producing data

### Resolution
1. Check DHCDCMON → Job Settings → Verify Observer and Transfer jobs are green
2. If not green, click Dispatcher Job to reschedule
3. Check DHCDCMON → Application Log for errors
4. For performance issues with empty buffer: Increase transfer jobs per SAP Note 3669170 and SAP Note 3223735 (Transaction DHCDCSTG / Table DHCDC_JOBSTG parameters TRANSFER_MAX_JOBS and TRANSFER_MIN_JOBS)

## Error Pattern 8: DATA_NOT_READY — Buffer Table Full
### Symptoms
- DHRDBMON shows buffer table full or filling up
- Records not Assigned to Package is high (highlighted)
- READY packages accumulating but not being consumed

### Root Cause
RMS cannot collect data from the buffer — typically a Cloud Connector issue, network problem, or Datasphere-side bottleneck

### Resolution
1. Check Cloud Connector status and connectivity
2. Verify Datasphere connection is valid (Connection Management → Validate)
3. Check DHRDBMON Expert Functions to manually manage packages if needed
4. If packages are stuck, use "Change Status to Ready" or "Remove Records from Package" cautiously

## Error Pattern 9: ABAP Requests Completely Stuck
### Symptoms
- No progress in replication
- DHRDBMON shows no movement in buffer tables
- DHCDCMON logging tables may be growing

### Root Cause
Backend ABAP processing is blocked — jobs stuck, locks, or resource exhaustion

### Resolution
1. Check SM37 for Observer/Transfer job status
2. Check SM12 for locks on relevant tables
3. Check ST22 for ABAP short dumps related to /1DH/ programs
4. Verify no system-wide issues (SM21 system log, ST06 OS monitor)

## Error Pattern 10: Fatal "Aborting the graph" on an ODP source
### Symptoms
- `An error occurred. Replication runs will restart on <date> at <time>. Error: Fatal error found. Aborting the graph.`
- `An error occurred. Replication runs are retrying. Error: Fatal error found. Aborting the graph.`
- Seen on Replication Flows over **ODP BW Context** or **ODP SAPI**

### Root Cause
Required SAP Notes from the central ABAP-Integration stack (**SAP Note 2890171**) are missing on the source. The ODP API for Gen2/RMS must be current on the source.

### Resolution
1. Install the **DMIS Note Analyzer** (**SAP Note 3016862**) on the source.
2. Run transaction **`CNV_NA_DI`** to list missing notes for the relevant scenario.
3. Apply missing notes per the source product:
   - S/4HANA 2022 collective: **3225712**
   - S/4HANA 2021 TCI: **3232522**
   - S/4HANA 2020 TCI: **3232559**
   - S/4HANA 1909 TCI: **3234938** (+ **2830276**)
   - DMIS 2011 SP23 / 2018 SP08 / 2020 SP04: **3156672 / 3156649**
4. Also apply **SAP Note 3412110** — improvements for the replication and initial load of ODP data sources.
5. Re-validate connection (must still read "Replication flows are enabled").

See also `odp-replication-troubleshooting.md`.

## Error Pattern 11: ODP SAPI Source Object not visible in container
### Symptoms
- Source Container `ODP_SAPI - ODP Context: SAPI` opens but extractor/DataSource is not listed.

### Root Cause
- Either the DataSource has not been released for ODP in the source, or the Application Component Hierarchy has not been transferred.

### Resolution
1. Confirm the DataSource is released for ODP per **SAP Note 2232584**.
2. Follow **KBA 3489773** (ODP SAPI Source does not appear in SAP Datasphere for Replication Flows).
3. On the source, run transaction **`RSA9`** and answer **Yes** to the popup "Do you want the content application Transfer Component Hierarchy?". Background: **KBA 2205577**.
4. Verify the remote user has the authorizations required by **SAP Note 3100673** — specifically `S_DHAMBSAP` (or `S_LTAMBSAP` on DMIS) for SAPI context.

## Error Pattern 12: SLT — Objects not visible under SLT container
### Symptoms
- Source Container `/SLT/<MTID>` lists no objects even though the Mass Transfer ID exists in `LTRC`.

### Root Cause
Communication user lacks authorizations for the SLT Connector or for ABAP Metadata Browser resources.

### Resolution
1. Check **KBA 3501459** — communication user authorizations for Replication Flow object visibility.
2. Ensure the remote user has role **`SAP_IUUC_REPL_REMOTE`** (plus `SAP_DH_CDC_REMOTE` for S/4HANA 2020+).
3. In Cloud Connector, confirm the resource list exposes `LTAMB_` / `LTAPE_` (DMIS) or `DHAMB_` / `DHAPE_` (S/4HANA) plus `RFC_FUNCTION_SEARCH`.
4. Run **`SU53`** on the source immediately after a failed browse attempt and check for missing `S_DMC_S_R` / `S_DMIS` (SLT) or `S_DHAMBSLT` / `S_LTAMBSLT` (ABAP Metadata Browser for SLT MTIDs).

## Error Pattern 13: Source conversion error — `string to decimal` / `columns=[WKURS]`
### Symptoms
- Initial load fails: `data could not be converted (keys: ..., columns: ...) reason=[<COL>:error while converting string to decimal]`

### Root Cause
Source column contains non-numeric bytes for specific rows (spaces, locale-formatted values, stray characters) that the target decimal type cannot accept.

### Resolution
Use the JSON-export / type-override workaround documented in `conversion-error-workaround.md`:
1. Clone target with failing column retyped to `string(100)` and renamed (e.g. `WKURS` → `WKURS_STR`).
2. Create a test RF, save without deploy, **export** JSON.
3. Edit JSON: set the source column's `vtype-ID` from `$DYNAMIC.decimal_*` to `$DYNAMIC.string_N`.
4. Import JSON, map `WKURS → WKURS_STR`, deploy, run.
5. In Data Viewer, filter out rows starting with digits `0-9` (and `-0` to `-9` for negative) to reveal the bad values.

## Component Ownership for SAP Support Cases
When opening an SAP support case, use these component assignments:
- RODPS_REPL_TEST returns error (BW) → Component: **BW-WHM-DBA-ODA**
- DHCDCMON Application Log shows "ACP daemon not start" (CDS CDC) → Component: **BC-DB-CDC**
- CDS extraction stuck → Check both **BC-DB-CDC** and **DS-DI-RF** (Datasphere Integration - Replication Flows)
- Cloud Connector issues / connection validation errors → Component: **DS-DI-CON** (and **BC-MID-SCC** if the Cloud Connector itself is faulty)
- Datasphere-side pipeline / runtime issues → Component: **DS-DI-RF**

## Key SAP Notes Quick Reference
| SAP Note | Description |
|----------|-------------|
| 2890171 | ABAP Integration - central note, CDS view requirements |
| 3297105 | Considerations & limitations for Replication Flows |
| 3100673 | ABAP Integration - Security Settings |
| 3016862 | DMIS Note Analyzer (transaction `CNV_NA_DI`) |
| 3498095 | How to use the Note Analyzer |
| 3397020 | "Cannot determine tables for CDS view" resolution |
| 3465112 | "Partitioning for Object failed" random failures |
| 3669170 | How to improve replication performance - ABAP CDC Engine |
| 3223735 | Transfer job tuning (DHCDC_JOBSTG) |
| 3369433 | Cloud Connector troubleshooting for Datasphere connections |
| 3456850 | Cloud Connector resource-list issues |
| 3449529 | Data collection for DS-DI-CON incidents |
| 3365864 | Where does information in DHCDCMON come from? |
| 2930269 | ABAP CDS CDC common issues and troubleshooting |
| 3476918 | How to access HANA Cloud DB traces |
| 3412110 | ABAP Integration - improvements for ODP initial load + delta |
| 2232584 | Release of SAP extractors for ODP replication (SAPI) |
| 3489773 | ODP SAPI Source does not appear in Datasphere |
| 2205577 | Application component does not exist in RSA5: Error R8418 |
| 3486245 | FAQ: RFC Fast Serialization in Replication Flow |
| 3339368 | FAQ: Push DataSource Delta Extraction via ODP/SAPI |
| 3481365 | Initial and Delta RF: data-load job remains Active |
| 3501459 | Communication user authorization - RF object visibility |
| 3360905 | RMS / Replication Flow performance |
| 3489681 | Low performance with Datasphere local tables as RF target |
| 2855052 | Authorizations required for ODP Data Replication API 2.0 |
