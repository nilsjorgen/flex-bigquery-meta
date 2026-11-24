# Manual runbook: 24-hour Datastream staleness

## Scope and precautions

Change all seven production streams in `flex-prod-af40`, region `europe-north1`, to allow 24-hour BigQuery CDC staleness. Leave development and the shared module default at one hour.

Use 25-hour replication reconciliation cutoffs, live Cloud SQL reads for current-state business checks, and 49-hour direct streamed-table freshness thresholds.

**Staleness is a table option, not a dataset option.** Stream `data_freshness` controls newly created tables; existing tables require a separate update. Neither value is a fixed replication schedule or a delivery guarantee for an unhealthy stream.

This guide contains planned edits and operator commands, not completed changes. No cloud commands have been executed. The installed cloud CLIs were permission-blocked during planning. Verify permissions, table inventory, and source column types before applying changes.

## 1. Prepare the three repositories

Create separate branches in `terraform-google-bigquery-datastream`, `flex-bigquery-terraform`, and `flex-bigquery-soda`. 

## 2. Correct the shared module documentation

In `terraform-google-bigquery-datastream/variables.tf`, keep `bigquery_table_freshness` at `default = "3600s"`. Correct its description to mean permitted BigQuery CDC query staleness, not maximum end-to-end delivery time.

Correct the same description in `README.md`, explaining existing versus newly created tables and that staleness is not an update frequency.

No functional module change is needed. Verify that the consumer's pinned `v1.0.10` exposes this existing input; do not bump the module pin merely for documentation changes.

## 3. Prepare the production stream overrides

In `flex-bigquery-terraform/prod/datastreams.tf`, add to the existing locals:

```hcl
bigquery_staleness_hours              = 24
datastream_reconciliation_grace_hours = local.bigquery_staleness_hours + 1
```

Add this input to all seven module calls:

```hcl
bigquery_table_freshness = "${local.bigquery_staleness_hours * 3600}s"
```

| Module | Stream ID | Dataset ID |
| --- | --- | --- |
| `spinnsyn_datastream` | `spinnsyn-datastream` | `spinnsyn_datastream` |
| `arkivering_oppgave_datastream` | `arkivering-oppgave-datastream` | `arkivering_oppgave_datastream` |
| `flexjar_datastream` | `flexjar-datastream` | `flexjar_datastream` |
| `inntektsmelding_status_datastream` | `inntektsmelding-status-datastream` | `inntektsmelding_status_datastream` |
| `sykepengesoknad_datastream` | `sykepengesoknad-datastream` | `sykepengesoknad_datastream` |
| `spinnsyn_arkivering_datastream` | `spinnsyn-arkivering-datastream` | `spinnsyn_arkivering_datastream` |
| `flex_sykmeldinger_backend_datastream` | `flex-sykmeldinger-backend-datastream` | `flex_sykmeldinger_backend_datastream` |

Leave `dev/datastreams.tf` unchanged. Document the production/dev distinction and steps 9-13 in the consumer README.

Prepare these overrides now, but deploy compatible views/contracts before increasing stream staleness.

## 4. Update replication reconciliation

In `flex-bigquery-terraform/prod/soda-dataset.tf`, update these six views:

| View |
| --- |
| `spinnsyn_vedtak_datastream_avstemming` |
| `sykepengesoknad_sykepengesoknad_avstemming` |
| `flex_sykmeldinger_backend_avstemming` |
| `arkivering_oppgave_oppgavestyring_avstemming` |
| `flexjar_feedback_avstemming` |
| `flex_inntektsmelding_status_datastream_avstemming` |

Change **both** the source PostgreSQL cutoff and the destination GoogleSQL cutoff from two to 25 hours. Inside Terraform SQL heredocs, use these patterns:

```sql
-- PostgreSQL
date_trunc('hour', current_timestamp)
  - INTERVAL '${local.datastream_reconciliation_grace_hours} hours'

-- GoogleSQL
timestamp_add(
  timestamp_trunc(current_timestamp, HOUR),
  INTERVAL -${local.datastream_reconciliation_grace_hours} HOUR
)
```

Keep view IDs, schemas, column names, comparison operators, and the existing two-day lower bounds. Verify PostgreSQL session timezone alignment with BigQuery UTC boundaries.

At midnight the two-day lookback leaves about 23 hours of eligible records. Check the actual Soda schedule for scan overlap; widen the lookback only if it would leave gaps.

The sykmeldinger reconciliation view has no matching active production contract in the inspected Soda ConfigMap. Update the view, but do not introduce a new alert.

## 5. Convert business/current-state checks to live source reads

Use existing connections from `prod/external-connection.tf` with `EXTERNAL_QUERY`. Connection names start with `flex-prod-af40.europe-north1.`.

Inspect actual source column types and database read permissions first. Push selective filters and aggregations into PostgreSQL, return only the existing output columns, and preserve BigQuery output schemas.

| View in `prod/soda-dataset.tf` | Source and behavior to retain |
| --- | --- |
| `sendt_sykepengesoknad_arkivering_oppgave_avstemming` | Live `sykepengesoknad` via `sykepengesoknad-backend`, and `innsending` via `sykepengesoknad-arkivering-oppgave`; preserve `SENDT`, the **two-hour business allowance**, seven-day lookback, and UUID comparison. |
| `spinnsyn_utbetaling_spinnsyn_arkivering_avstemming` | Live `arkivert_vedtak` via `spinnsyn-arkivering`, and `utbetaling` via `spinnsyn-backend`; preserve 24-hour cutoff, 14-day lookback, `motatt_publisert IS NOT NULL`, and existing comparison direction. |
| `arkivering_oppgave_venter_pa_bomlo` | Live `oppgavestyring` via the archiving connection; preserve `VenterPaBomlo` and contract limit 150. |
| `forsinket_saksbehandling_varslinger_sendt_nylig` | Live `vedtaksperiode_behandling_status_v2` via `flex-inntektsmelding-status`; preserve four statuses, 96-hour window, and grouped `status`/`antall` output. |
| `frisk_til_arbeid_vedtak_overlapp` | Live `frisk_til_arbeid_vedtak` via the application connection; preserve `OVERLAPP`. |
| `frisk_til_arbeid_vedtak_arbeidssokerperiode` | Same table/connection; preserve the existing two-status filter. |
| `frisk_til_arbeid_vedtak_ubehandlet` | Same table/connection; preserve `NY` and **older than 24 hours**, not 49 hours. |
| `flex_inntektsmelding_status_forelagte_opplysninger_forsinket` | Live `forelagte_opplysninger_ainntekt` via `flex-inntektsmelding-status`; preserve `NY`, existing day calculation, `forsinket_dager > 5`, and output columns/types. |
| `sykepengesoknad_aktivering_forsinket` | Live `sykepengesoknad`; inspect the existing projection in `prod/views-sykepengesoknad.tf` first. Preserve `FREMTIDIG`, `tom + 1 day`, three-month scope, business date semantics, and output columns/types. |
| `sykepengesoknad_vedtaksperiode_oppdatert_siste_dogn` | Live `vedtaksperiode_behandling` via `sykepengesoknad-backend`; preserve the last-24-hours activity requirement. |
| `innteksmelding_status_vedtaksperiode_oppdatert_siste_dogn` | Live `vedtaksperiode_behandling` via `flex-inntektsmelding-status`; preserve last-24-hours activity and existing spelling/ID. |

For example, after checking source types:

```sql
SELECT id
FROM EXTERNAL_QUERY(
  "flex-prod-af40.europe-north1.sykepengesoknad-backend",
  '''
  SELECT id FROM frisk_til_arbeid_vedtak
  WHERE behandlet_status = 'NY'
    AND opprettet < current_timestamp - INTERVAL '24 hours'
  '''
)
```

For cross-database checks, fetch bounded candidate rows through both connections and compare them in BigQuery. External queries do not form an atomic cross-database snapshot; retain business processing allowances.

Do not merely age mutable records by creation time: a recently resolved old record can still look unresolved in stale BigQuery data.

Test permissions under the Soda runner identity. It already requests `roles/bigquery.connectionUser`, but source grants and connection access still need verification. Connection failures must fail the scan, not return an apparent healthy zero-row result.

Leave analytical views unchanged. `sykmeldinghendelser_korrelerer_med_syfosmregister` and `sykepengesoknad_hag_soknadsperioder_view` have no contract in the inspected Flex production ConfigMap. Document possible delayed visibility for their external consumers; do not claim they are staleness-safe. The former also reads another team's dataset.

## 6. Update direct Soda freshness contracts

In `flex-bigquery-soda/nais/soda-checks-prod.yaml` and matching `local/soda-checks/*.yaml`, change `must_be_less_than: 24` to `49` for:

| Contract file |
| --- |
| `public_sykepengesoknad.yaml` |
| `public_sykmelding.yaml` |
| `public_vedtak_v2.yaml` |
| `public_oppgavestyring.yaml` |
| `public_feedback.yaml` |
| `public_arkivert_vedtak.yaml` |

Keep `unit: hour`, column `opprettet`, dataset paths, and strict comparison semantics. Example:

```yaml
checks:
  - freshness:
      name: "Ingen nye kilderegistreringer er synlige i BigQuery siste 49 timer."
      column: opprettet
      threshold:
        unit: hour
        must_be_less_than: 49
```

Make Norwegian names source-specific. Creation-time freshness is an activity heuristic, not proof replication has stopped; quiet sources can legitimately trigger it.

The allowance is the existing 24-hour source activity window plus 24-hour staleness plus one hour of margin. Keep business row-count thresholds unchanged and local/deployed definitions synchronized.

## 7. Validate changes

In each changed Terraform root, initialize as necessary and run existing checks:

```sh
terraform fmt -check
terraform validate
```

For the consumer's production plan, from `flex-bigquery-terraform/prod`:

```sh
terraform init
terraform plan -var-file=../terraform.tfvars
```

Require only intended view SQL and stream freshness changes. Reject unexpected replacements, dataset deletions, proxy/connection changes, or development changes. Do not print secrets or state contents.

Use the Soda repository's existing contract validation commands. Execute bounded read-only SQL checks under the runner configuration: dry runs alone do not validate PostgreSQL inside `EXTERNAL_QUERY`.

Check cutoff/hour/day boundaries and scan overlap; older genuinely missing records must still alert. Check recently resolved old records, 24-hour business deadlines, activation dates, last-day activity, and output type compatibility. Verify external-query execution time and Cloud SQL load at the actual schedule.

## 8. Deploy in safe stages

1. Deploy updated views and their supporting locals, **without** the seven 24-hour stream overrides.
2. Deploy updated Soda contracts; they work with existing one-hour tables too.
3. Run affected contracts to verify permissions and behavior.
4. Apply the seven stream overrides through Terraform.
5. Inventory and migrate existing tables in steps 9-10.

The shared module documentation can merge independently. Split consumer changes into separate commits/PRs to enforce this order, since `main` auto-applies production.

Do not pause/restart, backfill, change write modes, or recreate tables/datasets solely for staleness changes.

## 9. Inventory and save rollback values

Use a dedicated operator working directory. Save all seven stream configurations:

```sh
for stream in \
  spinnsyn-datastream \
  arkivering-oppgave-datastream \
  flexjar-datastream \
  inntektsmelding-status-datastream \
  sykepengesoknad-datastream \
  spinnsyn-arkivering-datastream \
  flex-sykmeldinger-backend-datastream
do
  gcloud datastream streams describe "$stream" \
    --project=flex-prod-af40 --location=europe-north1 \
    --format=json > "${stream}-before.json" || break
done
```

Confirm all seven files contain successful responses. Keep operational backups internal.

For each dataset in step 3, substitute its ID in the following query and output filename:

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 \
  --format=prettyjson query --use_legacy_sql=false \
  'SELECT tables.table_name, tables.table_type, options.option_value AS max_staleness
   FROM `flex-prod-af40.spinnsyn_datastream.INFORMATION_SCHEMA.TABLES` AS tables
   LEFT JOIN `flex-prod-af40.spinnsyn_datastream.INFORMATION_SCHEMA.TABLE_OPTIONS` AS options
     ON tables.table_catalog = options.table_catalog
    AND tables.table_schema = options.table_schema
    AND tables.table_name = options.table_name
    AND options.option_name = "max_staleness"
   ORDER BY tables.table_name;' > spinnsyn_datastream-before.json
```

Review the intended CDC merge-mode tables. Separately inspect tables without `max_staleness`; do not blindly alter views, append-only tables, or manually maintained tables. Preserve actual prior values instead of assuming all tables use one hour.

## 10. Update existing tables

Single table:

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 query \
  --use_legacy_sql=false \
  'ALTER TABLE `flex-prod-af40.spinnsyn_datastream.public_vedtak_v2`
   SET OPTIONS (max_staleness = INTERVAL 24 HOUR);'
```

Dataset-wide: run once per dataset in step 3, replacing **all three** `spinnsyn_datastream` references:

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 query \
  --use_legacy_sql=false <<'SQL'
FOR target IN (
  SELECT options.table_name
  FROM `flex-prod-af40.spinnsyn_datastream.INFORMATION_SCHEMA.TABLE_OPTIONS` AS options
  JOIN `flex-prod-af40.spinnsyn_datastream.INFORMATION_SCHEMA.TABLES` AS tables
    USING (table_catalog, table_schema, table_name)
  WHERE options.option_name = 'max_staleness'
    AND tables.table_type = 'BASE TABLE'
)
DO
  EXECUTE IMMEDIATE FORMAT(
    'ALTER TABLE `flex-prod-af40.spinnsyn_datastream.%s` SET OPTIONS (max_staleness = INTERVAL 24 HOUR)',
    target.table_name
  );
END FOR;
SQL
```

This updates only base tables already configured with `max_staleness`. Compare with the inventory; handle any separately verified CDC tables missing the option individually.

The loop is not atomic across tables. On failure, inspect successful changes, correct the error, and rerun; setting the same option again is safe.

## 11. Optional gcloud stream update

Skip if Terraform already applied `86400s`. This is an alternative out-of-band stream update, not a substitute for step 10.

Example using step 9's saved stream configuration; requires `jq`:

```sh
jq '.destinationConfig.bigqueryDestinationConfig | .dataFreshness = "86400s"' \
  spinnsyn-datastream-before.json > spinnsyn-bigquery-destination.json

gcloud datastream streams update spinnsyn-datastream \
  --project=flex-prod-af40 --location=europe-north1 \
  --bigquery-destination-config=spinnsyn-bigquery-destination.json \
  --update-mask=destination_config.bigquery_destination_config.data_freshness
```

Repeat with the appropriate stream and saved file if using this alternative for the other streams. Keep Terraform's desired value aligned to avoid reversion by a later apply.

The narrow mask preserves source, dataset, destination connection, write mode, backfill, and state. Do not use `--force`.

## 12. Verify production

For each stream:

```sh
gcloud datastream streams describe spinnsyn-datastream \
  --project=flex-prod-af40 --location=europe-north1 \
  --format='yaml(state,destinationConfig.bigqueryDestinationConfig.dataFreshness)'
```

Require update completion and `86400s`; investigate unexpected stream states.

For each migrated dataset:

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 query \
  --use_legacy_sql=false \
  'SELECT table_name, option_value
   FROM `flex-prod-af40.spinnsyn_datastream.INFORMATION_SCHEMA.TABLE_OPTIONS`
   WHERE option_name = "max_staleness"
   ORDER BY table_name;'
```

Compare the intended table list with the inventory. The interval may appear as a normalized string instead of the literal SQL text.

Inspect apply progress if needed:

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 query \
  --use_legacy_sql=false \
  'SELECT table_name, upsert_stream_apply_watermark
   FROM `flex-prod-af40.spinnsyn_datastream.INFORMATION_SCHEMA.TABLES`
   WHERE table_type = "BASE TABLE"
   ORDER BY table_name;'
```

Run affected Soda checks and monitor Cloud SQL load. Verify inheritance when a naturally new CDC table becomes available; do not create production source tables just to test it.

Business checks now monitor live source state, not replication. Retain reconciliation and streamed-table freshness checks for replication monitoring.

## 13. Roll back if needed

1. Restore production stream inputs through Terraform to their previous value, normally `3600s`, so newly created tables stop inheriting 24 hours.
2. Restore each table's inventoried prior option. For a table previously using one hour:

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 query \
  --use_legacy_sql=false \
  'ALTER TABLE `flex-prod-af40.spinnsyn_datastream.public_vedtak_v2`
   SET OPTIONS (max_staleness = INTERVAL 1 HOUR);'
```

3. Repeat inventory to catch tables created during rollout and restore their intended setting.
4. Verify streams/tables before reverting tolerant contracts or cutoffs.
5. Revert view/contract changes only if desired; live business checks can remain if source load is acceptable.

Lower staleness can trigger costly runtime merges. Never delete/recreate data or overwrite entire stream configurations to change this one field.

## References

- [Datastream destination staleness](https://docs.cloud.google.com/datastream/docs/configure-bigquery-destination#specify-staleness)
- [BigQuery CDC table staleness](https://docs.cloud.google.com/bigquery/docs/change-data-capture#manage_table_staleness)
- [gcloud streams update](https://docs.cloud.google.com/sdk/gcloud/reference/datastream/streams/update)
- [Production Soda contracts](https://github.com/navikt/flex-bigquery-soda/blob/main/nais/soda-checks-prod.yaml)

## Implementation status

Replace ⬜ with ✅ when a step is done.

| Step | Task                                | Status |
|------|-------------------------------------|--------|
| 1    | Prepare repositories                | ⬜      |
| 2    | Correct module documentation        | ⬜      |
| 3    | Prepare stream overrides            | ⬜      |
| 4    | Update replication reconciliation   | ⬜      |
| 5    | Use live source reads               | ⬜      |
| 6    | Update Soda freshness contracts     | ⬜      |
| 7    | Validate changes                    | ⬜      |
| 8    | Deploy in safe stages               | ⬜      |
| 9    | Save inventory and rollback values  | ⬜      |
| 10   | Update existing tables              | ⬜      |
| 11   | Optional gcloud stream update       | ⬜      |
| 12   | Verify production                   | ⬜      |
| 13   | Roll back if needed                 | ⬜      |
