# Replication Flow Load Types — Behavioural Reference (Scenarios A–F)

Six canonical scenarios demonstrate how `Load Type` (`Initial Only` vs `Initial and Delta`) combines with the `TRUNCATE` option. Use this when deciding which load type fits a migration/backfill/continuous-replication need, and when diagnosing why target row counts do not match source row counts.

> **Source convention in the scenarios** — a test table `ZBO_SFLIGHT` / `ZB_SFLIGHT` with a small number of passenger records.

---

## Change Type codes

`Initial and Delta` replication adds three technical columns to the target (when **Delta Capture** is enabled on the target table). The `Change Type` column has three possible values:

| Code | Meaning |
|------|---------|
| `L` | **Initial Load** — row came in during a full/initial load |
| `A` | **Upsert** — insert or update seen on source during delta |
| `D` | **Delete** — source row was deleted; target row is marked deleted, not physically removed |

These columns are visible in target table data previews.

---

## Gotcha: "Delta Capture Off cannot be re-enabled"

Once a local target table is deployed with **Delta Capture = Off**, you **cannot** re-enable Delta Capture on it later. You must **drop and recreate** the target (or create a new Replication Flow targeting a new table). Plan for Delta Capture **up front** whenever you anticipate needing `Initial and Delta` later.

When Delta Capture is enabled, the system automatically generates the technical columns (Change Type, Change Timestamp, etc.) during initial deployment.

## Gotcha: Each redeploy re-triggers Initial Load

> Each time the Replication Flow is redeployed, the **Initial Load is triggered first**, regardless of `Load Type`. Plan stop/restart windows accordingly, especially for high-volume tables.

---

## Scenario A — `Initial Only` without `TRUNCATE`

**Setup**: 3 records on source. Run `Initial Only`. Then delete 1 record on source (2 remain). Run `Initial Only` again.

**Result**: Target still has **3 rows**.
- `Initial Only` inserts all selected rows once.
- On re-run it appends new rows after existing rows and avoids duplicates, but **never deletes anything**.
- **Use when**: one-time migration, or append-only snapshot patterns.
- **Do NOT use when**: source deletes must be reflected on the target.

---

## Scenario B — `Initial Only` **with** `TRUNCATE`

**Setup**: 2 records on source, 3 on target (from scenario A). Re-run with `TRUNCATE` enabled.

**Result**: Target is cleared, then refilled from source — now **2 rows**.
- `TRUNCATE` deletes target content but leaves the table structure intact.
- **Use when**: re-initialising after divergence, refreshing a reference table, rolling forward a point-in-time snapshot.

---

## Scenario C — `Initial and Delta` without `TRUNCATE`

**Setup**: Table has existing data. New row added on source. Deploy a fresh RF with `Load Type = Initial and Delta` (and Delta Capture enabled on the target). Then add 2 more rows on source.

**Behaviour**:
1. Because the RF was just deployed, the **Initial Load fires first** — all rows load with `Change Type = L`.
2. Subsequent delta detects the 2 new rows and writes them with `Change Type = A` (upsert).

**Result**: Target keeps all rows; incremental changes arrive as delta.
- **Use when**: continuous replication of source changes into Datasphere.

---

## Scenario D — Remove a row on **source** and run `Initial and Delta` without `TRUNCATE`

**Setup**: Delete `PASSENGER2` on source.

**Result**: The target **keeps the row physically** but flips its `Change Type` to **`D`** (Delete). Data Integration Monitor shows one delta operation.
- Downstream consumers that honour Change Type must filter out `D` rows.
- Target row counts do not shrink — the `D` flag is the signal.

---

## Scenario E — Delete a row on **target** and run `Initial and Delta` without `TRUNCATE`

**Setup**: Delete `PASSENGER5` on the Datasphere target (e.g., manually in Data Builder).

**Result**: The next delta load **re-inserts** the row from source with `Change Type = L` (Initial Load), because from the CDC engine's perspective that row is seen "new" on the target.
- **Implication**: **do not** delete rows on the target to try to suppress them — delta will restore them. Filter upstream or in consumption.

---

## Scenario F — `Initial and Delta` **with** `TRUNCATE`

**Setup**: Target and source have diverged. Stop the running Replication Flow, enable `TRUNCATE`, redeploy, and run again.

- `TRUNCATE` only applies when the run starts; it is not dynamic.
- Initial Load clears the target and re-populates from source.
- Subsequent delta loads then add new rows as `A` and handle deletes as `D`.

**Result**: Both tables are resynced, and delta continues normally.
- **Use when**: reconciling divergence while keeping continuous replication.

---

## Decision Matrix

| Need | Load Type | TRUNCATE |
|------|-----------|----------|
| One-time migration, no delete tracking | `Initial Only` | — |
| Refresh reference table; drop stale rows | `Initial Only` | **Yes** |
| Continuous change replication, keep history | `Initial and Delta` | No |
| Resync continuously replicated target after divergence | `Initial and Delta` | **Yes** |
| Need to "hard delete" rows on target when source row disappears | Not supported directly — consume with a view filtering `Change Type = D` |

---

## Related SAP Notes / KBAs

- **SAP Note 3481365** — Initial and Delta RF: the data-load job remains `Active` (expected; it has no end date).
- **SAP Note 3297105** — Considerations and limitations for Replication Flows.
- **SAP Note 3486245** — FAQ: RFC Fast Serialization in Replication Flow (relevant to performance of delta loads).
