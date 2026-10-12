# USL Models for Zetaris Quick-Starts

A catalog of USL models that reuse data onboarded elsewhere in this repo. See [`HOWTO.md`](HOWTO.md) for their lifecycle and SQL patterns.

**Script location:** each model has a numbered `_create.sql` and `_select.sql` pair in `sql/`. Both reuse raw tables registered by `open_data/rest_apis/`.

---

## Quick-reference table

| # | Model | Tables | What it tests | Verification | Script |
|---|---|---|---|---|---|
| 1 | `sic_usl` | One: `sic_code` | Baseline USL lifecycle — namespace, compile, activate, a self-consistency check, deferred materialization | Lifecycle verified; self-consistency check confirmed not expressible as `REGISTER DQ`, runs as a plain query instead | `sql/01_sic_usl_create.sql` |
| 2 | `sic_edgar_usl` | Two, FK-related: `company`, `sic_code` | FK relationships, an FK-generated DQ rule, a description-agreement check, deferred `Valid Records Only` materialization | Lifecycle verified (rebuilt clean 2026-10-01 over JDBC); FK rule verified (found 1 real invalid record, 6/7, IBM); description-agreement check runs as a plain query instead of `REGISTER DQ`; FK-vs-check comparison run: complementary | `sql/02_sic_edgar_usl_create.sql` |

**Verified** means the statement ran successfully on a live instance. **Not yet run** means it has not been tested live. See the [SQL companion](../../docs/guides/zetaris-lightning-sql-companion.md#86-register-dq-confirmed-limitations-live-tested) for `REGISTER DQ` limits.

---

## 1. `sic_usl` — single-table USL

- **What it is:** a USL version of the `company_dns` SIC hierarchy, already registered through REST. It is the single-table lifecycle example.
- **Source data:** reuses `company_dns.sic_codes_raw` directly — no new REST registration.
- **Status:** namespace, compile, and activation succeeded with 1,005 rows. The self-consistency check runs as a plain query because it is not supported as `REGISTER DQ`. The primary-key DQ result is unconfirmed. Materialization is deferred because no cloud storage target is configured.

## 2. `sic_edgar_usl` — two-table, FK-related USL

- **What it is:** a USL version of the EDGAR/SIC company profile built by the REST package. It adds a foreign key and its generated DQ rule.
- **Source data:** reuses `company_dns.sic_codes_raw` and the seven `*_submissions_raw` REST tables from `sql/11` directly — no new REST registration.
- **Status:** both activations succeeded. The `cik` primary-key rule returned 7/7 valid; the `sic` foreign-key rule returned 6/7 valid and flagged IBM (`cik` `0000051143`, `sic` `3570`). A clean rebuild on 2026-10-01 confirmed that code `3570` is absent from the 1,005-row SIC reference. The plain description join excludes IBM and finds five wording differences, so it complements the FK rule. Description equality runs as a query, not a `REGISTER DQ` rule. Materialization is deferred because no cloud storage target is configured.

---

**See also:** `HOWTO.md` for the Zetaris USL onboarding walkthrough and the lifecycle this package exercises, `docs/guides/zetaris-lightning-sql-companion.md` section 8 for the general USL reference material, and `docs/plans/archive/usl-build-plan.md` for the live-verification checklist this catalog's status detail is drawn from.
