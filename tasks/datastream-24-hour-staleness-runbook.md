# Runbook: 24-hour Datastream staleness

## Scope

Change BigQuery CDC staleness from one hour to 24 hours for the seven
production streams in `flex-prod-af40`, region `europe-north1`.

- Keep the module default (`3600s`) in `terraform-google-bigquery-datastream`.
  This change does not touch the module repository.
- Keep `dev/` at one hour. `dev/` has no Soda views.
- Update the views in `soda_dataset` and the Soda contracts so that they
  give correct results with 24-hour staleness.

## Decisions

| Topic | Decision | Reason |
| --- | --- | --- |
| Stream staleness | Set `bigquery_table_freshness` in `prod/datastreams.tf`. | CI applies `main` to prod. Terraform reverts a change made only with gcloud. |
| Existing tables | Use `bq` (Google Cloud SDK) to run `ALTER TABLE`. | The stream setting applies only to new tables. Terraform does not manage the streamed tables. |
| Business views | Add 24 hours to each time window. | Small SQL changes. No new permissions and no Cloud SQL load. Alerts can come one run later. |
| Soda schedule | Run every day, not only on weekdays. | With a daily run, the two-day lookback leaves no gaps. |
| Module description | Do not change. | It is not necessary for this change. |

## Glossary

- **Staleness**: The maximum age of the data that a BigQuery query on a CDC
  table can return. It is the table option `max_staleness`. In Datastream,
  the stream setting is `data_freshness`.
- **Reconciliation view**: A view that compares IDs in Cloud SQL (with
  `EXTERNAL_QUERY`) with IDs in the streamed BigQuery table.
- **Business view**: A view that checks the state of the data, for example
  records with status `NY` that are too old.

## 1. Create branches

Create one branch in `flex-bigquery-terraform` and one in
`flex-bigquery-soda`. Do not use the prefix `dev-` in `flex-bigquery-terraform`.
The prefix starts an apply to `dev`.

## 2. Update the reconciliation views

File: `flex-bigquery-terraform/prod/soda-dataset.tf`.

Update these six views:

- `spinnsyn_vedtak_datastream_avstemming`
- `sykepengesoknad_sykepengesoknad_avstemming`
- `flex_sykmeldinger_backend_avstemming`
- `arkivering_oppgave_oppgavestyring_avstemming`
- `flexjar_feedback_avstemming`
- `flex_inntektsmelding_status_datastream_avstemming`

In each view, change the cutoff from 2 to 25 hours. Change both the
PostgreSQL side and the BigQuery side:

```sql
-- PostgreSQL, inside EXTERNAL_QUERY
WHERE opprettet < date_trunc('hour', current_timestamp) - INTERVAL '25 hours'

-- BigQuery
WHERE opprettet < timestamp_add(timestamp_trunc(current_timestamp, HOUR), INTERVAL -25 HOUR)
```

Keep the two-day lookback. Step 5 makes the schedule daily, so the windows of
two runs that follow each other overlap.

`flex_sykmeldinger_backend_avstemming` has no Soda contract. Update it
anyway, so that it stays the same as the other reconciliation views.

## 3. Update the business views

File: `flex-bigquery-terraform/prod/soda-dataset.tf`.

Make these changes. Keep view IDs, schemas and output columns.

| View | Change |
| --- | --- |
| `sendt_sykepengesoknad_arkivering_oppgave_avstemming` | `sendt < ... INTERVAL 2 HOUR` to `INTERVAL 26 HOUR`. |
| `spinnsyn_utbetaling_spinnsyn_arkivering_avstemming` | `INTERVAL 24 HOUR` to `INTERVAL 48 HOUR`. Change both occurrences. |
| `frisk_til_arbeid_vedtak_ubehandlet` | `INTERVAL 24 HOUR` to `INTERVAL 48 HOUR`. |
| `flex_inntektsmelding_status_forelagte_opplysninger_forsinket` | `forsinket_dager > 5` to `forsinket_dager > 6`. |
| `sykepengesoknad_aktivering_forsinket` | `forventet_aktivering_dato < CURRENT_DATE()` to `forventet_aktivering_dato < DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)`. |
| `forsinket_saksbehandling_varslinger_sendt_nylig` | `INTERVAL 96 HOUR` to `INTERVAL 120 HOUR`. |
| `sykepengesoknad_vedtaksperiode_oppdatert_siste_dogn` | `INTERVAL 24 HOUR` to `INTERVAL 48 HOUR`. |
| `innteksmelding_status_vedtaksperiode_oppdatert_siste_dogn` | `INTERVAL 24 HOUR` to `INTERVAL 48 HOUR`. |

Do not change these views:

- `arkivering_oppgave_venter_pa_bomlo`
- `frisk_til_arbeid_vedtak_overlapp`
- `frisk_til_arbeid_vedtak_arbeidssokerperiode`

These views check only the current status, with no time window. Staleness
only makes an alert come up to 24 hours later, or stay up to 24 hours longer
after a fix.

Do not change `sykmeldinghendelser_korrelerer_med_syfosmregister` and
`sykepengesoknad_hag_soknadsperioder_view`. They have no Soda contract.

## 4. Set staleness on the streams

File: `flex-bigquery-terraform/prod/datastreams.tf`.

1. Add this value to the existing `locals` block:

   ```hcl
   bigquery_table_freshness = "86400s"
   ```

2. Add this input to all seven module calls:

   ```hcl
   bigquery_table_freshness = local.bigquery_table_freshness
   ```

Do not change the module `ref`. `v1.0.10` already has the input.

This change applies only to tables that Datastream creates in the future.
Step 9 changes the existing tables.

## 5. Update the Soda contracts and schedule

Repository: `flex-bigquery-soda`. Make each change in both
`nais/soda-checks-prod.yaml` and `local/soda-checks/<contract>.yaml`.

1. Change the freshness threshold from `must_be_less_than: 24` to
   `must_be_less_than: 49` in these contracts:
   - `public_sykepengesoknad.yaml`
   - `public_sykmelding.yaml`
   - `public_vedtak_v2.yaml`
   - `public_oppgavestyring.yaml`
   - `public_feedback.yaml`
   - `public_arkivert_vedtak.yaml`

   49 hours is the 24-hour activity window, plus 24 hours staleness, plus
   one hour margin.
2. In the same six contracts, change "siste 24 timer" to "siste 49 timer" in
   `name`.
3. In `frisk_til_arbeid_vedtak_ubehandlet.yaml`, change "24 timer" to
   "48 timer" in `name`.
4. In `sykepengesoknad_vedtaksperiode_oppdatert_siste_dogn.yaml` and
   `innteksmelding_status_vedtaksperiode_oppdatert_siste_dogn.yaml`, change
   "de siste 24 timene" to "de siste 48 timene" in `name`.
5. In `nais/prod.yaml`, change `schedule: '00 4 * * 1-5'` to
   `schedule: '00 4 * * *'`.

The job sends Slack messages also on weekends, because
`NOTIFY_OK_SCAN_STATUS` is `true`.

## 6. Validate the changes

1. In `flex-bigquery-terraform/prod`, run:

   ```sh
   terraform init
   terraform fmt -check
   terraform validate
   terraform plan -var-file=../terraform.tfvars
   ```

2. Make sure that the plan has only these changes:
   - In-place updates of the 14 views from steps 2 and 3.
   - In-place updates of `data_freshness` on the seven streams.
3. Stop if the plan replaces or deletes a resource.
4. In BigQuery, run the SQL of each changed view as a query. Make sure that
   it runs and gives the expected columns.
5. In `flex-bigquery-soda/local`, run `make run-soda-checks`. The local views
   are still the old ones, so this step only checks the contract syntax.

## 7. Deploy

The new views and contracts also work with one-hour staleness. The real
switch is step 9.

1. Merge the `flex-bigquery-terraform` branch. CI applies it to prod.
2. Merge the `flex-bigquery-soda` branch. CI deploys the contracts and the
   new schedule, and runs the checks once.
3. Make sure that the run-once job passes.

## 8. Save the current table staleness

Save the current values before you change them. You need them for rollback.

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 --format=prettyjson \
  query --use_legacy_sql=false --max_rows=1000 '
SELECT o.table_schema, o.table_name, o.option_value
FROM `flex-prod-af40.region-europe-north1.INFORMATION_SCHEMA.TABLE_OPTIONS` AS o
JOIN `flex-prod-af40.region-europe-north1.INFORMATION_SCHEMA.TABLES` AS t
  USING (table_catalog, table_schema, table_name)
WHERE o.option_name = "max_staleness"
  AND t.table_type = "BASE TABLE"
  AND o.table_schema IN (
    "spinnsyn_datastream",
    "arkivering_oppgave_datastream",
    "flexjar_datastream",
    "inntektsmelding_status_datastream",
    "sykepengesoknad_datastream",
    "spinnsyn_arkivering_datastream",
    "flex_sykmeldinger_backend_datastream")
ORDER BY 1, 2' > max-staleness-before.json
```

Make sure that the file has tables from all seven datasets.

## 9. Change staleness on the existing tables

Run this script once. It changes all base tables in the seven datasets that
already have `max_staleness`.

```sh
bq --project_id=flex-prod-af40 --location=europe-north1 \
  query --use_legacy_sql=false <<'SQL'
FOR target IN (
  SELECT o.table_schema, o.table_name
  FROM `flex-prod-af40.region-europe-north1.INFORMATION_SCHEMA.TABLE_OPTIONS` AS o
  JOIN `flex-prod-af40.region-europe-north1.INFORMATION_SCHEMA.TABLES` AS t
    USING (table_catalog, table_schema, table_name)
  WHERE o.option_name = 'max_staleness'
    AND t.table_type = 'BASE TABLE'
    AND o.table_schema IN (
      'spinnsyn_datastream',
      'arkivering_oppgave_datastream',
      'flexjar_datastream',
      'inntektsmelding_status_datastream',
      'sykepengesoknad_datastream',
      'spinnsyn_arkivering_datastream',
      'flex_sykmeldinger_backend_datastream')
)
DO
  EXECUTE IMMEDIATE FORMAT(
    'ALTER TABLE `flex-prod-af40.%s.%s` SET OPTIONS (max_staleness = INTERVAL 24 HOUR)',
    target.table_schema, target.table_name);
END FOR;
SQL
```

The script is not atomic. If it fails, correct the error and run it again.
It is safe to set the same value two times.

## 10. Verify production

1. For each of the seven streams, run:

   ```sh
   gcloud datastream streams describe <stream-id> \
     --project=flex-prod-af40 --location=europe-north1 \
     --format='yaml(state,destinationConfig.bigqueryDestinationConfig.dataFreshness)'
   ```

   Make sure that `state` is `RUNNING` and `dataFreshness` is `86400s`.
2. Run the query from step 8 again, without the output file. Make sure that
   all tables show 24 hours.
3. Read the Slack results of the next scheduled Soda run.

## 11. Roll back

1. Revert the change from step 4 in `prod/datastreams.tf` and merge it.
2. For each table in `max-staleness-before.json`, set the saved value. For
   example:

   ```sh
   bq --project_id=flex-prod-af40 --location=europe-north1 \
     query --use_legacy_sql=false \
     'ALTER TABLE `flex-prod-af40.spinnsyn_datastream.public_vedtak_v2`
      SET OPTIONS (max_staleness = INTERVAL 1 HOUR);'
   ```

3. Keep the view and contract changes. They also work with one-hour
   staleness.

## References

- [Datastream destination staleness](https://docs.cloud.google.com/datastream/docs/configure-bigquery-destination#specify-staleness)
- [BigQuery CDC table staleness](https://docs.cloud.google.com/bigquery/docs/change-data-capture#manage_table_staleness)
- [gcloud datastream streams describe](https://docs.cloud.google.com/sdk/gcloud/reference/datastream/streams/describe)

## Implementation status

Replace ⬜ with ✅ when a step is done.

| Step | Task | Status |
| --- | --- | --- |
| 1 | Create branches | ⬜ |
| 2 | Update the reconciliation views | ⬜ |
| 3 | Update the business views | ⬜ |
| 4 | Set staleness on the streams | ⬜ |
| 5 | Update the Soda contracts and schedule | ⬜ |
| 6 | Validate the changes | ⬜ |
| 7 | Deploy | ⬜ |
| 8 | Save the current table staleness | ⬜ |
| 9 | Change staleness on the existing tables | ⬜ |
| 10 | Verify production | ⬜ |
| 11 | Roll back | ⬜ |
