# Troubleshooting Conversion Errors in Source Columns

When a Replication Flow fails during initial load with a message like:

```
Transferring initial data failed due to the following error:
data could not be converted (keys: values for key fields, columns: names of failing columns):
keys=[] columns=[WKURS] reason=[WKURS:error while converting string to decimal]
```

the source is delivering a value that cannot be coerced into the target column's data type — typically a source `DECIMAL` field whose content contains non-numeric bytes (spaces, special characters, localized format) for specific rows.

The message contains:
- The failing column (`WKURS` in the example).
- The failing value's key fields when the table has keys; empty `keys=[]` when the source has no key fields (then you cannot identify the bad row from the error alone).
- The conversion reason (`error while converting string to decimal`).

This reference documents the **JSON-export / type-override** workaround that lets you land the problematic data as a string so you can **find and inspect** the bad rows.

---

## Workaround Overview

Bypass the target type check by routing the failing column into a **string** target column, then filter the preview to identify non-numeric rows.

1. Clone or create a **shadow target table** — name it e.g. `LT_TARGET`:
   - Change the failing column's data type to `string(100)` (or similar, sized to the widest expected value).
   - Rename the column from `WKURS` → `WKURS_STR` to avoid confusion downstream.

2. Create a **test Replication Flow** with the **same source** as the original; target is `LT_TARGET`.

3. Save the new RF — **do not deploy yet**.

4. In the RF modeller toolbar, click the **Export** icon to download the RF definition as **JSON**.

5. Open the JSON in an editor and search for the source column name in quotes (`"WKURS"`). The source side will look like:

   ```json
   {
       "name": "WKURS",
       "vflow.type": "scalar",
       "vtype-ID": "$DYNAMIC.decimal_x_y"
   }
   ```

   Change the `vtype-ID` to a string of appropriate length, for example `"$DYNAMIC.string_10"`:

   ```json
   {
       "name": "WKURS",
       "vflow.type": "scalar",
       "vtype-ID": "$DYNAMIC.string_10"
   }
   ```

   The target column already looks like this (because you redefined `LT_TARGET` with `WKURS_STR` as `string(100)`):

   ```json
   {
       "name": "WKURS_STR",
       "vflow.type": "scalar",
       "vtype-ID": "$DYNAMIC.string_100",
       "businessName": "WKURS_STR"
   }
   ```

6. **Import** the edited JSON back into Datasphere (Data Builder → Import).

7. In the RF Data Builder, map `WKURS` (source, now string) → `WKURS_STR` (target string).

8. **Deploy and run** the test RF. With the decimal-to-decimal conversion replaced by string-to-string, the initial load should succeed (assuming this was the only problematic column).

9. **Identify the bad values** — open `LT_TARGET` in Data Builder → Data Viewer, and filter the preview to **exclude rows whose `WKURS_STR` starts with digits 0–9**. What remains will surface the non-numeric values.
   - If **negative** values exist, extend the exclude filter with 10 more clauses for `-0` through `-9`.

10. Fix the bad rows at source (cleanse or escape), then revert to the original RF with the proper decimal column mapping.

---

## Why not just change the target column type directly?

Changing the target's declared type is a modelling step — this workaround is a **diagnostic**. It lets you land the failing data so you can see which rows are bad without tight access to the source system. After cleansing at source, redeploy the original RF.

---

## See Also

- **SAP Note 2890171** — central note for required ABAP-integration corrections; data-side conversion bugs are sometimes tracked here.
- **SAP Note 3297105** — considerations and limitations (documents type-handling constraints in Replication Flows).
- **KBA 3397020** — "Cannot determine tables for CDS view" — different symptom, related diagnostic category.
