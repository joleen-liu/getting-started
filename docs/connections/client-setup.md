# Zetaris client setup

Use this page to configure a client and choose a connection. Keep passwords
and API keys in the local, gitignored `.env.local` file. Never put them in a
prompt, source file, command output, or URL.

## 1. Choose a protocol

| Use | Client and endpoint | Authentication |
|---|---|---|
| Run Lightning SQL with a Java driver | JDBC to the Zetaris query endpoint | Zetaris user ID and password, plus the driver JAR |
| Run SQL or call platform endpoints | Direct REST API at `/api/v1.0` | User-created API key, organisation ID, and request ID |
| Make a quick query from a repository helper | UI proxy at `/api/proxy/...` | API key, or UI login with user ID and password |

The direct REST API and UI proxy are separate routes. `ZETARIS_REST_URL` is
the API base address. `ZETARIS_BASE_URL` is the web UI origin. Do not put the
raw API address in `ZETARIS_BASE_URL`.

For a local instance, the defaults are `jdbc:zetaris:lightning@localhost:10000`
for JDBC, `http://localhost:8888/api/v1.0` for direct REST, and
`http://localhost:3000` for the UI proxy. For Zetaris Cloud or another remote
instance, use the URLs and organisation details supplied to you.

## 2. Set up `.env.local`

Copy the needed settings from [`.env.example`](../../.env.example) into
`.env.local` at the repository root. If the file already exists, add settings
without replacing its other values. This is the repository's only env file.
The platform bundle has a separate `.env`. Scripts also find the top-level
checkout's `.env.local` when run from a Git worktree.

| Variable | Used by | Set it to |
|---|---|---|
| `ZETARIS_JDBC_URL` | JDBC | `jdbc:zetaris:lightning@localhost:10000` locally, or the supplied JDBC URL |
| `ZETARIS_JDBC_JAR` | JDBC | Full path to the matching Zetaris driver JAR. If it is unset, provide the path when asked; the scripts do not search for it. |
| `ZETARIS_USERNAME` | JDBC, OpenAPI spec, optional UI-proxy login | Your Zetaris user ID |
| `ZETARIS_PASSWORD` | JDBC, OpenAPI spec, optional UI-proxy login | Your Zetaris password |
| `ZETARIS_REST_URL` | Direct REST | API base ending in `/api/v1.0`; local default is `http://localhost:8888/api/v1.0` |
| `ZETARIS_ORG_ID` | REST, UI proxy | Numeric **ID** shown under **Management → Organisation** in the Zetaris UI. See [Find your organisation ID](#3-find-your-organisation-id). |
| `ZETARIS_API_KEY` | REST, UI proxy | API key you create in the Zetaris GUI |
| `ZETARIS_BASE_URL` | UI-proxy helpers | Web UI origin, such as `http://localhost:3000`; no path after the origin |
| `ZETARIS_QUERY_LIMIT` | UI-proxy query helper | Positive row limit; defaults to `1000` |
| `ZETARIS_ENGINE_ID` | UI-proxy query helper | Optional compute engine ID |
| `ZETARIS_USER_AGENT` | Create scripts that call external APIs | Your app name and contact email. SEC EDGAR requires a real contact. |

If you handle local credentials in a terminal, restrict the file permissions:

```sh
chmod 600 .env.local
```

### Create an API key

Create the key yourself in the Zetaris GUI, signed in with your account. Copy
it when the GUI shows it, then save it as `ZETARIS_API_KEY` in `.env.local`.
The key has your account's permissions. Never ask an agent to create or fetch
it, and do not paste it into a prompt. If REST returns `401`, create a fresh
key in the GUI and update the local value.

The UI-proxy helpers can use that API key directly. If it is not set, they can
log in with both `ZETARIS_USERNAME` and `ZETARIS_PASSWORD`. Direct REST uses the
bearer key. JDBC uses the username and password.

## 3. Find your organisation ID

1. Sign in to the Zetaris UI.
2. In the sidebar, open **Management → Organisation**.
3. Copy the numeric value in the **ID** field into `ZETARIS_ORG_ID` in `.env.local`.

Use the **ID**, not the organisation name or login URL slug. For example, if
the ID field shows `1`, set `ZETARIS_ORG_ID=1`. Use the value shown for your
organisation.

If you cannot access this page, ask your organiser or administrator for the
numeric organisation ID. Direct REST and the UI-proxy helpers require this
setting.

## 4. Prepare JDBC

JDBC needs JDK 11 or later, the matching Zetaris driver JAR, and your Zetaris
user ID and password. A direct Java connection does not need Python. The
repository's Python JDBC runner additionally needs `jaydebeapi` in `.venv`.
On macOS,
`java` may be a stub even if a Homebrew JDK is installed; check
`/usr/libexec/java_home` and
`/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home` and set
`JAVA_HOME` to the JDK path. Do not assume or search for the driver JAR path.

The read-only preflight reports what is missing and installs nothing. Its
`--jdbc` mode also checks direct REST readiness, so a REST-only failure does
not by itself mean JDBC is unavailable. Read the JDBC-specific result
separately:

```sh
python3 scripts/preflight.py --jdbc --jar /path/to/zetaris-driver.jar
```

Preflight detects the Java path, not its version. After setting `JAVA_HOME`,
run `"$JAVA_HOME/bin/java" -version` to confirm that runtime is JDK 11 or later.

If there is no JDK, ask before installing one. If Java is not available, use
REST if your account has it.

## 5. JDBC starter prompt

Set the JDBC values in `.env.local`, then submit this as the first instruction.
Replace the goal with your task.

```text
Read AGENTS.md and follow it. Run python3 scripts/preflight.py --jdbc and show
me the result. Report JDBC and REST checks separately. If any check fails,
stop and report what is missing. If the driver JAR path is not set, ask me
for it. Once readiness checks pass, connect over JDBC to the instance
configured in .env.local and confirm with SELECT 1 and SHOW LIGHTNING DATABASES.

Goal: <for example: list NY_TAXI and TPCDS_DB tables, then read 5 rows from
NY_TAXI.taxi_zones; keep it read-only>

Dry-run before creating anything and wait for my go-ahead.
```

## 6. REST starter prompt

Create your API key and set the REST values in `.env.local` first. Then submit
this as the first instruction.

```text
Read AGENTS.md and follow it. Run python3 scripts/preflight.py and show me the
result. Then connect over REST to the instance configured in .env.local and
confirm with SELECT 1 and SHOW LIGHTNING DATABASES.

Goal: <for example: list NY_TAXI and TPCDS_DB tables, then read 5 rows from
NY_TAXI.taxi_zones; keep it read-only>

Dry-run before creating anything and wait for my go-ahead.
```

## 7. Verify access

The starter prompts run the read-only preflight first. For manual checks, run
`python3 scripts/preflight.py` for direct REST, or add `--jdbc --jar <path>`
for JDBC. A successful check returns `1` from `SELECT 1` and shows the
instance's Lightning databases. Direct REST also checks
`GET /datasource/datasources`.

`SHOW DATASOURCES` lists sources registered with `CREATE DATASOURCE`. It does
not list REST and file sources registered as Lightning databases. Use
`SHOW LIGHTNING DATABASES` to see those. On a clean instance, an empty list is
normal. REST returns values as strings, and decimals may use exponent format;
convert them before comparing.

The small `query_zetaris` and `check_zetaris` helpers use the UI proxy. For
example, each language can run one query:

```sh
./scripts/query_zetaris.ts "SELECT 1"
python3 scripts/query_zetaris.py "SELECT 1"
```

## 8. Safety and permissions

- Start with read-only commands such as `SHOW`, `DESCRIBE`, and `SELECT`.
- On a shared instance, get your team prefix and permissions first. Check that
  an existing object belongs to your team and matches the definition before
  reusing it.
- Creating, dropping, granting, or writing changes the instance. Preview
  create scripts with `--dry-run` and wait for the user's clear go-ahead before
  running them.
- Send one Lightning SQL statement per call. Keep a multi-table `COMPILE USL`
  together as one statement; do not split it at semicolons.
- Keep credentials out of prompts, logs, source files, and commits.

## 9. Fetch the OpenAPI specification

Fetch a fresh specification when you need a direct REST endpoint. The spec
route uses HTTP Basic authentication with `ZETARIS_USERNAME` and
`ZETARIS_PASSWORD`. It does not accept the bearer API key. Save the spec
outside the repository because it may contain environment-specific details.

```sh
set -a
. ./.env.local
ZETARIS_REST_URL="${ZETARIS_REST_URL:-http://localhost:8888/api/v1.0}"
set +a
curl -sS -u "$ZETARIS_USERNAME:$ZETARIS_PASSWORD" \
  "${ZETARIS_REST_URL%/api/v1.0}/redoc/docs.yaml" \
  -o /tmp/zetaris-docs.yaml
```

For a local instance this fetches `http://localhost:8888/redoc/docs.yaml`.
For a remote instance, `ZETARIS_REST_URL` must point to its `/api/v1.0` API
base. Do not print or echo the credentials.

## 10. Troubleshooting

| Symptom | Check |
|---|---|
| `Unable to locate a Java Runtime` | Find a JDK and set `JAVA_HOME`. Ask before installing one. |
| Driver JAR missing or `ClassNotFoundException` | Set the full path to the matching driver JAR. Do not guess a path. |
| `externally-managed-environment` from `pip` | Use the repository's `.venv`; do not install packages into system Python. |
| REST `401` | For direct REST, create a new GUI API key and update `ZETARIS_API_KEY`. The spec route needs Basic auth, not the bearer key. |
| REST `400 Not Allowed` | Check that `ZETARIS_ORG_ID` is the numeric ID for your account. |
| REST `400` on a valid request | Check that `X-Request-ID` and `queryId` are UUIDs. |
| A source is missing from `SHOW DATASOURCES` | Check `SHOW LIGHTNING DATABASES`; REST and file registrations appear there. |
| `Table not found` | Use the qualified name `<source>.<table>`. |
| JDBC `mismatched input ... expecting` | Send one statement per call. The `run_sql.py` runner splits scripts and keeps `COMPILE USL` intact. |

For script-specific setup and the difference between one-request helpers and
multi-statement runners, see [`scripts/HOWTO.md`](../../scripts/HOWTO.md).
