# HOWTO: onboard these REST API sources into Zetaris

Start with [Start here](../../START-HERE.md) and [your first dataset](../../docs/guides/first-dataset.md). PUDL is currently known to fail. On the shared event instance, use your assigned database and container names consistently.


**File naming:** each source has a numbered `_create.sql` file for DDL and a matching `_select.sql` file for verification and examples. Run the create script first. Verification and optional queries in the select files are commented out by default.

**Source folders:** pairs under `sql/` are grouped by rate limits and current readiness:

- `sql/rate_limited/` — space out queries; use a registered API key when available. Caching may reduce repeated requests.
- `sql/non_rate_limited/` — no hard limit was observed; follow the source's fair-use terms.
- `sql/known_to_fail/` — Singapore PM2.5 registration returned HTTP 502. See its [issue](sql/known_to_fail/ISSUE.md).

`rest-api-sources.md`'s quick-reference table lists which folder each source is in.

---

## 0. Fast start: `company_dns` end to end in about 5 minutes

This walkthrough registers, caches, flattens, and verifies the live `company_dns` SIC hierarchy. Its scripts are `sql/non_rate_limited/10_company_dns_sic_create.sql` and `_select.sql`.

**Note:** after idle time, the hosted service may return an empty response or HTTP 502. Warm it with the optional script in step 1, then retry once if needed.

1. *(Optional, see note above)* From the repository root: `deno run --allow-net --allow-env scripts/warmup_company_dns.ts`.
2. Open the Zetaris **SQL Editor** and register the logical database:
   ```sql
   CREATE LIGHTNING DATABASE COMPANY_DNS DESCRIBE BY "company_dns SIC reference data - division, major group, industry group, SIC code";
   ```
3. Register and cache the REST table:
   ```sql
   CREATE LIGHTNING REST TABLE sic_codes_raw FROM COMPANY_DNS REQUEST(
       endpoint "https://company-dns.mediumroast.io/V3.0/na/sic/code/%25",
       method "get",
       response_type "json",
       http_encoding "URLENCODED"
   ) HEADER (
       user-agent "YOUR_APP_NAME YOUR_CONTACT_EMAIL"
   ) BODY ();

   CACHE TABLE company_dns.sic_codes_raw;
   ```
4. Flatten the dynamic-key JSON response into a view:
   ```sql
   CREATE SCHEMASTORE CONTAINER company_dns;

   CREATE SCHEMASTORE VIEW sic_codes_table WITH CONTAINER company_dns AS
   SELECT
       sic_code,
       sic_val.description         AS description,
       sic_val.division             AS division,
       sic_val.division_desc        AS division_desc,
       sic_val.major_group          AS major_group,
       sic_val.major_group_desc     AS major_group_desc,
       sic_val.industry_group       AS industry_group,
       sic_val.industry_group_desc  AS industry_group_desc
   FROM company_dns.sic_codes_raw
   LATERAL VIEW explode(
       from_json(to_json(data.sics), 'map<string, struct<description:string, division:string, division_desc:string, major_group:string, major_group_desc:string, industry_group:string, industry_group_desc:string>>')
   ) AS sic_code, sic_val;
   ```
5. Verify it worked:
   ```sql
   SELECT COUNT(*) FROM company_dns.sic_codes_table;   -- expect 1005
   ```
6. Try a real query — every SIC code in the "Manufacturing" division, for example:
   ```sql
   SELECT sic_code, description, major_group_desc
   FROM company_dns.sic_codes_table
   WHERE division_desc LIKE '%Manufacturing%'
   ORDER BY sic_code;
   ```

The matching select file has eight more queries. The sections below cover other sources, verification, and troubleshooting.

---

## 1. Running the scripts

1. Open the Zetaris **SQL Editor**.
2. For each source in `sql/`, in order:
   - Read the `_create.sql` header and its caveats.
   - Run the `CREATE LIGHTNING DATABASE` statement (Step 0).
   - Create the SchemaStore container once; skip this statement if it already exists.
   - Fill required `HEADER` values. SEC EDGAR requires a real `User-Agent`.
   - Run each `CREATE LIGHTNING REST TABLE` and `CREATE SCHEMASTORE VIEW` statement.
   - Run the matching verification query in `_select.sql` before trusting the result.

Suggested order, from simpler verified patterns to higher-risk shapes:

| Order | Script | Why here |
|---|---|---|
| 1 | `non_rate_limited/10_company_dns_sic_create.sql` | Verified fast start. |
| 2 | `rate_limited/01_edgar_company_facts_create.sql` | Verified array-of-structs pattern. |
| 3 | `rate_limited/08_statcan_wds_create.sql` | Flat array-of-structs; GET only. |
| 4 | `non_rate_limited/02_pokeapi_create.sql` | Verified nested structs. |
| 5 | `rate_limited/03_open_food_facts_live_create.sql` | Verified sparse array fields. |
| 6 | `rate_limited/05_nasa_neows_create.sql` | Verified deep nesting and nested arrays. |
| 7 | `rate_limited/06_nasa_donki_create.sql` | Unverified top-level JSON array. |
| 8 | `non_rate_limited/07_eurostat_create.sql` | Verified metadata; dynamic-key values need decoding. |
| 9 | `non_rate_limited/09_abs_data_api_create.sql` | Doubly dynamic SDMX-JSON keys; higher risk. |

**Excluded:** Singapore PM2.5 registration returned HTTP 502. See the [issue write-up](sql/known_to_fail/ISSUE.md).

---

## 2. Verifying data

For every source, run the verification query in its `_select.sql` (commented out by default):

1. Count rows in the Zetaris view.
2. Count the matching array directly from the API with `curl` and `jq`, for example:
   ```bash
   curl -s -A "YOUR_APP_NAME YOUR_CONTACT_EMAIL" \
     https://data.sec.gov/api/xbrl/companyconcept/CIK0000320193/us-gaap/Revenues.json \
     | jq '.units.USD | length'
   ```
3. Compare counts before analysis. A lower Zetaris count may mean truncation; check the source response first. On EDGAR, the direct API and Zetaris both returned 11 rows for Apple's `us-gaap:Revenues` tag.

---

## 3. Removing a source

On 12 October 2026, Zetaris 2.4.3.1 accepted these commands against disposable public-S3 file tables and a PokeAPI REST table:

- `DROP LIGHTNING DATABASE TABLE sqltest_20261012_files.reason` (file table)
- `DROP LIGHTNING DATABASE TABLE sqltest_20261012_files.ability` (PokeAPI REST table)
- `DROP LIGHTNING DATABASE sqltest_20261012_files`

The database drop succeeded after its test tables were removed. This does not establish behavior for every connector. Use the exact `DROP LIGHTNING DATABASE TABLE` form; the check does not establish that generic `DROP TABLE` works. `DROP DATASOURCE` was not validated for these Lightning registrations.

`DROP VIEW <container>.<view>` removes a SchemaStore view. Container removal was performed through the documented API after confirming `viewCount` was zero; this validation did not establish a SQL command for dropping a container. Script `TEARDOWN` blocks therefore keep their `DROP VIEW` statements and direct readers here for source cleanup.

---

## 4. Troubleshooting / FAQ

### My query failed with an unresolved-column error mentioning a mixed-case, reserved-word, or hyphenated field name

A JSON field may need backticks when it uses mixed case, a reserved word, spaces, or punctuation such as a hyphen. For example, use `` `entityName` ``, `` fact.`start` ``, or `` product.nutriments.`energy-kcal_100g` ``. Check the source keys before writing a view.

The unquoted forms may fail or parse incorrectly because Zetaris treats the name as a different identifier or as an operator.

### My query failed with `UNRESOLVED_COLUMN`, referencing a column that "doesn't exist" on a view I expected to have it

Check the view's own `CREATE SCHEMASTORE VIEW ... AS SELECT` list. A comment or a similar view may name columns this view does not expose.

Check the raw response too. Singapore PM2.5's documented `national` field was absent from the JSON; see the [issue write-up](sql/known_to_fail/ISSUE.md).

### My query failed with `MISSING_ATTRIBUTES.RESOLVED_ATTRIBUTE_APPEAR_IN_OPERATION`

This can occur when one query reads the same `UNION ALL` view built with `LATERAL VIEW explode(...)` more than once, including through a self-join or subquery. For example:

```sql
-- Avoid this pattern:
SELECT a.*
FROM some_view a
JOIN (
    SELECT key, MAX(val) AS max_val
    FROM some_view
    GROUP BY key
) top ON top.key = a.key AND top.max_val = a.val;
```

The analyzer can see conflicting internal attributes from the repeated exploded view. This was reproduced with StatCan `IN` and scalar subqueries and an Open Food Facts self-join. Joins between distinct views did not show the issue.

**Fix:** use a window function so the query reads the view once:

```sql
SELECT key, val
FROM (
    SELECT key, val, ROW_NUMBER() OVER (PARTITION BY key ORDER BY val DESC) AS rn
    FROM some_view
) ranked
WHERE rn = 1;
```

Use `COUNT(*) OVER (PARTITION BY key)` for group counts or `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` to select a top row. See the StatCan and Open Food Facts select scripts for examples.

### A numeric-looking field sorts or compares wrong (e.g. "closest" or "smallest" picks the wrong row)

JSON may encode numeric-looking values as strings. NeoWs distance and velocity fields are quoted strings, so Zetaris can sort them lexicographically. Cast them to `DOUBLE`, `INT`, or `BIGINT` in the view before numeric comparisons or aggregation.

### A JSON-stat / SDMX source returns an empty result even though the dataset should have data

Eurostat silently returns an empty result for an invalid dimension code such as `age=Y15-74`. Check `dimension.<name>.category.index` for valid codes before concluding the dataset has no matching rows.

### A table's row count looks lower than I expect

Compare the Zetaris count with a direct API count first. Apple's EDGAR `us-gaap:Revenues` endpoint returned 11 rows in both checks; a low count can reflect the source's selected tag, not connector truncation.

### `CREATE LIGHTNING REST TABLE` (or the Data Explorer previewing a new source) returned an HTTP 502

Check the endpoint directly with `curl` before changing headers or authentication. The Singapore PM2.5 endpoint returned HTTP 200 to a plain GET, but rapid testing hit 429 after about 5–6 requests in 5–8 seconds and recovered after about 30 seconds. Its first Zetaris registration returned 502, not the upstream 429; the cause of that difference remains unresolved. See the [issue write-up](sql/known_to_fail/ISSUE.md).

Wait about 30 seconds before retrying a rate-limited endpoint. Avoid probing it with repeated `curl` calls immediately beforehand. If 502s persist, check the source API again at a slower interval.

### A REST source's HTTP 502 is a cold-start problem, not rate limiting — and can surface as a client-side `TTransportException`

The hosted `company_dns` service previously returned 502 after idle and succeeded on an immediate retry. Warm it with `scripts/warmup_company_dns.ts` before registration or query. A later SELECT may report `TTransportException`; REST tables fetch the endpoint again at query time. If `DESCRIBE` and direct `curl` succeed, retry before changing the payload or schema.

### A query fails with a rate-limit error well after `CREATE LIGHTNING REST TABLE` already succeeded

On NASA NeoWs, registration and an early SELECT succeeded; a later SELECT returned:

```
REST API call failed: Server returned HTTP response code: 429 for URL: https://api.nasa.gov/neo/rest/v1/neo/browse?api_key=DEMO_KEY
...
Response body: <html><body><h1>OVER_RATE_LIMIT</h1><p>You have exceeded your rate limit...
```

The error URL was the source endpoint. A REST table is registered at CREATE time, but an uncached query fetches the endpoint again. Repeated SELECTs can therefore exhaust rate limits. This does not explain the distinct PM2.5 502; its cause remains unresolved in the [issue write-up](sql/known_to_fail/ISSUE.md).

**Workaround:** cache the raw REST table right after creating it, before running further queries, using the `CACHE TABLE` statement:
```sql
CACHE TABLE <logical_datasource_name>.<raw_table_name>;
```
Caching `company_dns.sic_codes_raw` reduced the UI-reported count query from 49.685 seconds to about 1.2 seconds. Whether that cache also speeds up dependent SchemaStore views has not been tested. Use `UNCACHE TABLE <same_name>;` to release it.

`CACHE TABLE` is explicit in-memory caching. It is distinct from Adaptive Cache; its persistence and status reporting are limited. `SHOW CACHE TABLES` does not prove explicit-cache membership. See the [SQL companion](../../docs/guides/zetaris-lightning-sql-companion.md#5-operational-limitations-confirmed-live-not-documentation-guesses) for details and materialization options.

If caching does not help, reduce or space out queries and use a registered key where available. NASA's `DEMO_KEY` allows 30 requests/hour and 50/day per IP; a free key allows 1,000/hour. The shared quota resets on a rolling basis.

### Can I `DROP` a REST table or its Lightning database?

On 12 October 2026, `DROP LIGHTNING DATABASE TABLE` removed one PokeAPI REST table on Zetaris 2.4.3.1. See "Removing a source" above for the exact command and limits. This does not establish that generic `DROP TABLE` works for every connector.

### `CREATE SCHEMASTORE CONTAINER` failed with "A container named ... already exists"

`CREATE SCHEMASTORE CONTAINER` has no `IF NOT EXISTS` form. If it already exists, skip that statement or use `python3 scripts/run_sql.py --skip-exists FILE`; see the [SQL companion](../../docs/guides/zetaris-lightning-sql-companion.md#5-operational-limitations-confirmed-live-not-documentation-guesses).

### `CREATE LIGHTNING DATABASE ... DESCRIBE BY "..."` failed with "Description is invalid"

`DESCRIBE BY` accepts letters, digits, spaces, `_`, `.`, `-`, and `,`. Parentheses, slashes, and colons fail. For example, replace `"... (division/major group)"` with `"... - division, major group"`. This validation runs before Zetaris contacts the endpoint.

### How do I query a table once it's inside a Virtual Data Mart?

Use `<mart_name>.<table_name>`. The mart omits the original `SCHEMASTORE CONTAINER` prefix and uses the VDM node's "Virtual Table" alias. For example, a mart named `companies_mart` containing `edgar.all_companies_profile_table`:
```sql
SELECT * FROM companies_mart.all_companies_profile_table;
```
If the VDM renamed the table, use its alias instead of the source name.

---

## 5. Basic syntax

`CREATE LIGHTNING REST TABLE` registers the endpoint response; `CREATE SCHEMASTORE VIEW` flattens it into columns. REST requests use `endpoint`, `HEADER`, and `BODY`, not a file `PATH`. See the [Parquet/CSV guide](../parquet_csv/HOWTO.md) for file tables.

The general shape (the Fast Start above is a filled-in, real version of this):

```sql
-- Step 0: register the logical database (same prerequisite as filestore
-- tables -- see ../parquet_csv/HOWTO.md, "Registering a datasource").
CREATE LIGHTNING DATABASE <logical_datasource_name> DESCRIBE BY "<short description>";

-- Step 1: register the raw REST response as a table.
CREATE LIGHTNING REST TABLE <raw_table_name> FROM <logical_datasource_name> REQUEST(
    endpoint "https://api.example.com/path",
    method "get",
    response_type "json",
    http_encoding "URLENCODED"
) HEADER (
    user-agent "YOUR_APP_NAME YOUR_CONTACT_EMAIL"   -- required by some APIs, e.g. SEC EDGAR
) BODY ();

-- Step 2: a SCHEMASTORE container to hold flattened views (see
-- companion guide sec 5 (docs/guides/zetaris-lightning-sql-companion.md) -- this can only be created ONCE per name).
CREATE SCHEMASTORE CONTAINER <container_name>;

-- Step 3: flatten the raw JSON into a queryable view.
CREATE SCHEMASTORE VIEW <view_name> WITH CONTAINER <container_name> AS
SELECT
    <top_level_field>,
    `<mixedCaseField>` AS <alias>,          -- backtick-quote mixed-case JSON keys
    fact.<nested_field> AS <alias>
FROM <logical_datasource_name>.<raw_table_name>
LATERAL VIEW explode(<array_field>) AS fact;
```

Register `<logical_datasource_name>` with `CREATE LIGHTNING DATABASE` before referencing it in `FROM`.

---

## 6. Known limitations

See the [SQL companion](../../docs/guides/zetaris-lightning-sql-companion.md#5-operational-limitations-confirmed-live-not-documentation-guesses) for platform limits on containers, teardown, request re-fetching, cache behavior, JSON bodies, `DESCRIBE BY`, and row counts.

Two issues are specific to these sources:

- Singapore PM2.5 registration returned HTTP 502; see its [issue](sql/known_to_fail/ISSUE.md).
- StatCan's `getCubeMetadata` requires `Content-Type: application/json`; the form-encoded request returned 415 in direct `curl` testing. This is the package's example of the JSON-body limitation.

---

## 7. Understanding JSON response shapes

See the [SQL companion's REST-shape sections](../../docs/guides/zetaris-lightning-sql-companion.md#2-rest-tables-the-shape-you-must-plan-for-before-writing-sql) for `explode()`, dot-access, dynamic keys, and field types.

Source shapes in this package:

1. **Object with array fields:** EDGAR, PokéAPI, Open Food Facts, and NeoWs. PokéAPI has nested structs; Open Food Facts ingredients can contain a second array, which this script does not explode.
2. **Array under a wrapper:** Singapore PM2.5 uses `data.items`, with a struct for each item's readings; registration is known to fail.
3. **Top-level array:** NASA DONKI. Registration of this shape is unverified.
4. **Dynamic-key objects:** Eurostat, ABS, and `company_dns` use `from_json(to_json(...), 'map<...>')`. Eurostat's full decode must cover every dynamic-key object exploded in a query; ABS needs two key-decoding levels and remains untested.
5. **Nested struct without an array:** Open Food Facts `product.nutriments` uses dot access, not `explode()`.

No source here uses parallel arrays such as `times: [...]` and `values: [...]`. That shape needs `posexplode()` and positional indexing; inspect the response before writing a query.

---

## 8. Everything else

For licenses and source links, see [`rest-api-sources.md`](rest-api-sources.md). The [SQL companion](../../docs/guides/zetaris-lightning-sql-companion.md#1-the-two-onboarding-patterns-this-repo-has-proven) compares REST, file, and JDBC onboarding. Other categories are tracked in [`docs/plans/FUTURES.md`](../../docs/plans/FUTURES.md).
