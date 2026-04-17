# ABAP Remote User Authorizations for Replication Flows

Authoritative source: **SAP Security Note 3100673 — SAP Data Intelligence / SAP Datasphere - ABAP Integration - Security Settings**.

This reference consolidates the role-and-authorization-object model used by SAP Datasphere Replication Flows connecting to ABAP-based sources (S/4HANA on-premise, SAP BW, DMIS-based standalone SLT). Use it when provisioning or troubleshooting the **remote (communication) user** configured in a Datasphere ABAP / S/4HANA connection.

---

## 1. Required Standard Roles

| Role | Purpose | Assign to |
|------|---------|-----------|
| **`SAP_DI_ABAP_REMOTE`** | Enables ABAP Metadata Browser + ABAP Pipeline Engine for the RMS-side remote calls | Communication user on **S/4HANA on-premise** sources |
| **`SAP_DI_ABAP_USER`** | Grants read access to CDS / table / view / SQL view metadata and application log | **Any user** that needs to select source objects in the Datasphere modeler |
| **`SAP_IUUC_REPL_REMOTE`** | Enables the SLT Connector for Replication Flows using SLT as the source path | Communication user on **SLT** systems (both standalone DMIS and embedded S/4HANA when SLT is used) |
| **`SAP_DH_CDC_REMOTE`** | Additional role required for S/4HANA on-premise **release 2020 or higher** when using CDC-enabled (Initial and Delta) replication | Communication user on S/4HANA 2020+ |

**Recommended practice**: copy each standard role into a `ZCUS_*` custom role (PFCG), generate the profile, and assign the Z-role to the user — standard roles should not be used directly.

---

## 2. Role Contents

### 2.1 `SAP_DI_ABAP_REMOTE` (communication user)

Contains: `S_RFC`, `S_DHAMBACT`, `S_DHAMBBW`, `S_DHAMBCDS`, `S_DHAMBOBJ`, `S_DHAMBSAP`, `S_DHAMBSLT`, `S_DHAMBTAB`, `S_DHAMBVW`, `S_DHAPEAC2`, `S_DHAPEOPR`, `S_DHCDCACT`, `S_DHCDCCDS`, `S_DHCDCSTP`.

### 2.2 `SAP_DI_ABAP_USER` (any user)

Contains: `S_SQL_VIEW`, `S_TCODE`, `S_TABU_CLI`, `S_TABU_NAM`, `S_APPL_LOG`, `S_DHADMACT`, `S_DHCDCACT`, `S_DHCDCCDS`, `S_DHCDCSTP`, `S_DHCDCTAB`, `S_DHRDBACT`.

### 2.3 `SAP_IUUC_REPL_REMOTE` (SLT Connector)

Contains: `S_DMC_S_R`, `S_RFC`, `S_TCODE`, `S_BTCH_ADM`, `S_BTCH_JOB`, `S_CTS_ADMI`, `S_DATASET`, `S_LOG_COM`, `S_DEVELOP`, `S_DMIS`.

---

## 3. Authorization Object Cross-Reference (S/4HANA vs DMIS)

Most of the `S_DH*` objects on S/4HANA have direct `S_LT*` equivalents on DMIS-based systems. Assign the correct variant based on where the remote user lives:

| Auth object (S/4HANA) | Auth object (DMIS) | Description |
|-----------------------|--------------------|-------------|
| `S_DHAMBACT` | `S_LTAMBACT` | ABAP Metadata Browser |
| `S_DHAMBCDS` | (n/a)        | CDS Views released for ABAP Metadata Browser |
| `S_DHAMBSLT` | `S_LTAMBSLT` | **SLT Mass Transfer IDs** released for ABAP Metadata Browser |
| `S_DHAMBTAB` | `S_LTAMBTAB` | Database tables released for ABAP Metadata Browser |
| `S_DHAMBVW`  | `S_LTAMBVW`  | Database views released for ABAP Metadata Browser |
| `S_DHAMBOBJ` | `S_LTAMBOBJ` | Objects released for ABAP Metadata Browser |
| `S_DHAMBBW`  | `S_LTAMBBW`  | ODP objects in **BW context** released for ABAP Metadata Browser |
| `S_DHAMBSAP` | `S_LTAMBSAP` | ODP objects in **SAPI context** released for ABAP Metadata Browser |
| `S_DHAPEAC2` | `S_LTAPEAC2` | ABAP Pipeline Engine |
| `S_DHAPEOPR` | `S_LTAPEOPR` | Operators released for ABAP Pipeline Engine |
| `S_DHCDCACT` | (n/a)        | Change Data Capture Engine |
| `S_DHCDCCDS` | (n/a)        | CDS views released for CDC Engine |
| `S_DHCDCSTP` | (n/a)        | Subscriber types released for CDC Engine |
| `S_DHRDBACT` | `S_LTRDBACT` | Resilient Data Buffer (RDB) |

### Use-case matrix

| Scenario | Must-have auth objects (beyond `S_RFC`, `S_TCODE`) |
|----------|-----------------------------------------------------|
| CDS view replication (Initial + Delta) | `S_DHAMBACT`, `S_DHAMBCDS`, `S_DHAMBTAB`, `S_DHAPEAC2`, `S_DHAPEOPR`, `S_DHCDCACT`, `S_DHCDCCDS`, `S_DHCDCSTP`, `S_DHRDBACT`, `S_SQL_VIEW` |
| ABAP table replication via **SLT** (S/4HANA) | Add `S_DHAMBSLT`, plus everything in `SAP_IUUC_REPL_REMOTE` (`S_DMC_S_R`, `S_DMIS`, …) |
| ABAP table replication via **SLT** (DMIS) | `S_LTAMBACT`, `S_LTAMBSLT`, `S_LTAMBTAB`, `S_LTAPEAC2`, `S_LTAPEOPR`, `S_LTRDBACT`, plus `SAP_IUUC_REPL_REMOTE` content |
| ODP / BW Context | Add `S_DHAMBBW` (S/4HANA) or `S_LTAMBBW` (DMIS). Also apply **SAP Note 2855052** (Authorizations for ODP Data Replication API 2.0) |
| ODP / SAPI Context | Add `S_DHAMBSAP` (S/4HANA) or `S_LTAMBSAP` (DMIS) + SAP Note 2855052 |

---

## 4. Step-by-Step Setup

### 4.1 Create custom role `ZCUS_DI_ABAP_REMOTE`

1. Transaction **`PFCG`** → copy `SAP_DI_ABAP_REMOTE` → `ZCUS_DI_ABAP_REMOTE`.
2. On the Authorizations tab, **Change** authorization data.
3. Click the yellow **Status** button → assign **Full authorization for Subtree** for the generated objects.
4. **Generate** the profile.

### 4.2 Create custom role `ZCUS_DI_ABAP_USER`

1. **`PFCG`** → copy `SAP_DI_ABAP_USER` → `ZCUS_DI_ABAP_USER`.
2. Change authorization data, assign Full authorization for Subtree, **generate** profile.

### 4.3 Create (or maintain) the remote user

1. **`SU01`** → create a Dialog user (example name: `DSPREMOTE`).
2. Assign both `ZCUS_DI_ABAP_REMOTE` **and** `ZCUS_DI_ABAP_USER`.
   - **Both roles are required.** Without `ZCUS_DI_ABAP_USER`, the user hits `NO_AUTHORIZATION` in Datasphere when clicking **Select Source Container**, with `S_SQL_VIEW` the missing object in SU53.
3. For **SLT** scenarios additionally assign a custom role based on `SAP_IUUC_REPL_REMOTE`.
4. For **S/4HANA 2020+** using CDC, additionally assign `SAP_DH_CDC_REMOTE`.
5. Verify the profile is auto-generated (User tab should turn green; if it's yellow, regenerate the profile on the role and re-sync users).

### 4.4 Additional roles by use case (per Security Note 3100673)

- **ODP**: follow **SAP Note 2855052** (Authorizations required for ODP Data Replication API 2.0) to grant ODP-side auth objects on top of the above.
- **SLT**: also see the SAP Landscape Transformation Replication Server security guide (product help, version-appropriate).

---

## 5. Diagnosing Authorization Failures

### 5.1 Standard workflow

1. Reproduce the failure in Datasphere (e.g., *Select Source Container* throws `NO_AUTHORIZATION`, or a Replication Flow run logs "Not authorized to use operator 'internal.inport' (BADI_DHAPE_OPER_INPORT)" / "…'com.sap.abap.operator_reader' (BADI_DHAPE_OPER_OPER_READER)").
2. On the source system, **immediately** run **`/nSU53`** for the remote user. SU53 captures the **last** failed authorization check — do not perform unrelated actions in between.
3. If SU53 is not specific enough, enable **`STAUTHTRACE`** for the remote user, reproduce, then review the trace for the exact object + field that failed.

### 5.2 Common missing objects

| Missing in SU53 | Meaning | Fix |
|-----------------|---------|-----|
| `S_SQL_VIEW` | `ZCUS_DI_ABAP_USER` not assigned | Assign the second role to the remote user |
| `S_DMC_S_R`, `S_DMIS` | `SAP_IUUC_REPL_REMOTE` missing | Assign the SLT role |
| `S_DHAMBSLT` / `S_LTAMBSLT` | Mass Transfer ID not authorized for browse | Adjust the auth object field `MT_ID` in the custom role |
| `S_DHAMBCDS` | CDS view not released to the Metadata Browser | Adjust auth field `CDS_ENTITY` in the custom role |
| `S_DHAMBBW` / `_LTAMBBW` | BW ODP object not allowed | Add the BW object name to the auth field |
| `S_DHAMBSAP` / `_LTAMBSAP` | ODP SAPI DataSource not allowed | Add the DataSource name to the auth field |
| `S_DHAPEOPR` — BADI_DHAPE_OPER_INPORT / OPER_READER | RF operator not allowed for the user | Grant wildcard or required operator values |
| `S_DHCDCACT` / `S_DHCDCCDS` / `S_DHCDCSTP` | CDC subscription denied | Assign CDC objects (subscriber type = RMS) |

### 5.3 The `S_DMIS` / `/LTB/JOB_DISPATCHER` side-effect

`S_DMIS` is required not only for SLT but also for the background job **`/LTB/JOB_DISPATCHER`**. If SLT-unrelated jobs (for example the Migration Cockpit dispatcher) get stuck with long runtimes, the Step User under `SJOBREPO_STEPUSER` may lack `S_DMIS`. See **KBA 3428275 — SAP S/4HANA Migration Cockpit: Migration tasks are stuck (long runtimes)** — grant `SAP_ALL` (or a profile that contains `S_DMIS`) to the step user in all relevant clients.

---

## 6. References

- **SAP Note 3100673** — SAP Data Intelligence / SAP Datasphere - ABAP Integration - Security Settings (central source).
- **SAP Note 2855052** — Authorizations required for ODP Data Replication API 2.0.
- **SAP Note 2890171** — central ABAP Integration note (includes security updates).
- **KBA 3428275** — SAP S/4HANA Migration Cockpit: Migration tasks stuck — `S_DMIS` dependency for `/LTB/JOB_DISPATCHER`.
- **KBA 3501459** — Communication user authorization issue when browsing containers/objects.
- **SAP Landscape Transformation Replication Server Security Guide** — product-specific companion for SLT roles.
