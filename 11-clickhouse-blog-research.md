# ClickHouse Blog & Repo Research

## Purpose

This document collects ClickHouse optimizations from official blogs, community repos, and case studies. Once complete, we'll compare against existing docs (08, 09, 10) to identify new patterns.

---

## 1. reversedns.space

**Source**: [GitHub Repo](https://github.com/ClickHouse/reversedns.space)

**Use Case**: Reverse DNS lookup visualization with IP-to-coordinate mapping.

### Schema Patterns

```sql
-- Pattern 1: Empty ORDER BY for raw ingestion (fastest inserts)
CREATE TABLE dns (
    time DateTime DEFAULT now(),
    json String
) ENGINE = MergeTree()
ORDER BY ();  -- No ordering = fastest inserts

-- Pattern 2: Native IPv4 type + Bool flags
CREATE TABLE dns_parsed (
    time DateTime,
    Status UInt8,
    TC Bool,              -- Native Bool type for flags
    RD Bool,
    RA Bool,
    AD Bool,
    CD Bool,
    ip IPv4,              -- Native IPv4 type (4 bytes, not String)
    domain String
) ENGINE = MergeTree()
ORDER BY ip;

-- Pattern 3: Composite key for search
CREATE TABLE dns_search (
    reversed_domain String,
    ip IPv4
) ENGINE = MergeTree()
ORDER BY (reversed_domain, ip);
```

### Data Type Optimizations

| Type | Size | vs String | Use Case |
|------|------|-----------|----------|
| `IPv4` | 4 bytes | vs 7-15 bytes | IP addresses |
| `IPv6` | 16 bytes | vs 39 bytes | IPv6 addresses |
| `Bool` | 1 byte | vs variable | True/false flags |
| `UInt8` | 1 byte | vs variable | Status codes, enums |

### All Functions Used

**IP Address Functions**:
```sql
-- Convert string to IPv4
IPv4StringToNum('192.168.1.1')  -- Returns UInt32

-- Convert to native IPv4 type
toIPv4('192.168.1.1')

-- Byte manipulation for reverse DNS
byteSwap(reinterpretAsUInt32(ip))  -- Swap endianness

-- Convert IP to fixed string for manipulation
reinterpretAsFixedString(ip)

-- Reverse string (for PTR record parsing)
reverse(some_string)
```

**JSON Parsing with Typed Tuple**:
```sql
-- Extract complex nested JSON into typed columns
JSONExtract(json, 'Tuple(
    Status UInt8,
    TC Bool,
    RD Bool,
    RA Bool,
    AD Bool,
    CD Bool,
    Question Array(Tuple(name String)),
    Answer Array(Tuple(data String)),
    Authority Array(Tuple(data String)),
    Comment Array(String)
)')
```

**String Operations**:
```sql
-- Regex extraction
extract(domain, '^([0-9]+)\\.([0-9]+)\\.([0-9]+)\\.([0-9]+)\\.in-addr\\.arpa$')

-- Remove trailing dots from domains
replaceRegexpOne(domain, '\\.$', '')
```

**Hashing & Spatial**:
```sql
-- Fast hash for distribution/coloring
sipHash64(domain)

-- Z-order curve: convert 1D IP to 2D coordinates
mortonDecode(2, ip_as_uint32)  -- Returns (x, y) tuple
```

### Query Optimizations

```sql
-- WITH FILL: Ensure complete grid output (no gaps)
SELECT x, y, color
FROM dns_tiles
ORDER BY x, y WITH FILL FROM 0 TO 1024
-- Fills missing x,y combinations with defaults

-- Force primary key usage (prevent full scans)
SET force_primary_key = 1;

-- Enable new query analyzer
SET allow_experimental_analyzer = 1;
```

### User Security & Quotas (Public-Facing ClickHouse)

```sql
-- Create read-only user with SHA256 password
CREATE USER website IDENTIFIED WITH sha256_hash BY 'hash...';

-- Grant minimal permissions
GRANT SELECT ON default.dns_parsed TO website;

-- User settings (all READONLY = user cannot change)
ALTER USER website SETTINGS
    readonly = 1 READONLY,                    -- No writes allowed
    add_http_cors_header = 1 READONLY,        -- Enable CORS for web
    limit = 1 READONLY,                       -- Result limit locked
    offset = 0 READONLY,                      -- Pagination locked
    max_result_rows = 1 READONLY,             -- Max rows locked
    force_primary_key = 1 READONLY;           -- Must use primary key

-- Create quota with IP-based tracking
CREATE QUOTA website
    KEYED BY ip_address                       -- Track per client IP
    FOR RANDOMIZED INTERVAL 1 minute MAX 100 queries,
    FOR RANDOMIZED INTERVAL 1 hour MAX 3000 queries,
    FOR RANDOMIZED INTERVAL 1 day MAX 30000 queries
    TO website;
```

### Key Insights

1. **Empty ORDER BY `()`** - Valid for staging tables, fastest inserts
2. **Native IPv4/IPv6 types** - 4-10x smaller than String representation
3. **Bool type** - Use for flags instead of UInt8 (semantic clarity)
4. **JSONExtract with Tuple** - Parse JSON directly into typed columns
5. **mortonDecode** - Z-order curve for spatial indexing of IPs
6. **force_primary_key** - Prevent accidental full table scans
7. **KEYED BY ip_address quotas** - Rate limit per client IP
8. **RANDOMIZED interval** - Prevents quota gaming at interval boundaries
9. **READONLY settings** - Lock settings so users can't override
10. **WITH FILL** - Ensure complete output grids without gaps

---

## 2. sql.clickhouse.com

**Source**: [GitHub Repo](https://github.com/ClickHouse/sql.clickhouse.com)

**Use Case**: Interactive SQL playground with 30+ real-world datasets (GitHub events, PyPI, UK property, Bluesky, OpenTelemetry, etc.)

### 2.1 User/Role/Quota Security (setup.sql)

```sql
-- Create user with SHA1 password hash and default role
CREATE USER demo
    IDENTIFIED WITH double_sha1_hash BY 'BE1BDEC0AA74B4DCB079943E70528096CCA985F8'
    DEFAULT ROLE demo_role;

-- Settings profile with hard limits
CREATE SETTINGS PROFILE demo SETTINGS
    readonly = 1,                          -- No writes
    max_execution_time = 60,               -- 60 second query timeout
    max_rows_to_read = 10000000000,        -- 10B rows max
    max_result_rows = 1000,                -- 1000 rows returned
    max_result_bytes = 10000000,           -- 10MB result size
    enable_http_compression = true;         -- Compress responses

-- Role with profile attached
CREATE ROLE demo_role SETTINGS PROFILE 'demo';

-- Granular GRANT across 30+ databases
GRANT SELECT ON amazon.* TO demo_role;
GRANT SELECT ON github.* TO demo_role;
GRANT SELECT ON imdb.* TO demo_role;
GRANT dictGet ON mydb.my_dict TO demo_role;  -- Dictionary access

-- Column-level permissions on system tables
GRANT SELECT(query, elapsed, read_rows) ON system.processes TO monitor;

-- Quota with multiple intervals
CREATE QUOTA demo
    KEYED BY ip_address
    FOR INTERVAL 1 hour MAX
        queries = 60,
        result_rows = 3000000000,
        execution_time = 6000
    TO demo;
```

### 2.2 Data Loading Patterns

**Atomic Table Swap (Zero-Downtime Updates)**:
```sql
-- 1. Clone schema to temp table
CREATE TABLE pypi.projects_v2 AS pypi.projects;

-- 2. Load new data into temp (parallel threads)
INSERT INTO pypi.projects_v2
SELECT * FROM s3('${path}', '${key}', '${secret}')
SETTINGS max_insert_threads = 16;

-- 3. Atomic swap (instant, no downtime)
EXCHANGE TABLES pypi.projects AND pypi.projects_v2;

-- 4. Cleanup old data
DROP TABLE pypi.projects_v2;
```

**Incremental Loading with Max Tracking**:
```sql
-- Get latest ingested timestamp
SELECT max(file_time) FROM ${TABLE_NAME};

-- Generate hourly file list for backfill
WITH (SELECT ...) as hours
SELECT toString(toDate(...)) || '-' || toString(toHour(t)) || '.json.gz'
FROM range(0, hours);

-- Stream insert with format
INSERT INTO ${TABLE_NAME} FORMAT JSONCompactEachRow
```

**Input Format Settings**:
```sql
-- Treat nulls as column defaults
SET input_format_null_as_default = 1;

-- Flexible datetime parsing
SET date_time_input_format = 'best_effort';

-- Empty CSV fields as defaults
SET input_format_csv_empty_as_default = 1;
```

### 2.3 Medallion Architecture (Bluesky Dataset)

**Bronze Layer (Raw Ingestion)**:
```sql
-- S3Queue for continuous ingestion
CREATE TABLE bluesky.bluesky_queue (
    data Nullable(String)
)
ENGINE = S3Queue('https://storage.googleapis.com/bucket/*.gz', 'CSVWithNames')
SETTINGS
    mode = 'ordered',
    s3queue_buckets = 30,
    s3queue_processing_threads_num = 10;

-- Raw storage with dedup hash
CREATE TABLE bluesky.bluesky_raw (
    data JSON(
        SKIP `commit.record.reply.root.record`,  -- Skip nested fields
        SKIP `commit.record.value.value`
    ),
    _file LowCardinality(String),
    kind LowCardinality(String) MATERIALIZED JSONExtractString(data, 'kind'),
    scrape_ts DateTime64(6) MATERIALIZED fromUnixTimestamp64Micro(...),
    bluesky_ts DateTime64(6) MATERIALIZED ...,
    dedup_hash String MATERIALIZED cityHash64(...)
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (kind, bluesky_ts)
ORDER BY (kind, bluesky_ts, dedup_hash);

-- MV with JSON validation
CREATE MATERIALIZED VIEW bluesky.bluesky_mv TO bluesky.bluesky_raw AS
SELECT * FROM bluesky.bluesky_queue
WHERE isValidJSON(data) = 1;  -- Filter invalid JSON
```

**Silver Layer (Cleaned + DLQ)**:
```sql
-- Dedup table with TTL
CREATE TABLE bluesky.bluesky_dedup (
    data JSON(SKIP ...),
    kind LowCardinality(String),
    scrape_ts DateTime64(6),
    bluesky_ts DateTime64(6),
    dedup_hash String
)
ENGINE = ReplacingMergeTree
PARTITION BY toStartOfInterval(bluesky_ts, INTERVAL 20 MINUTE)
ORDER BY dedup_hash
TTL bluesky_ts + INTERVAL 1440 MINUTE
SETTINGS ttl_only_drop_parts = 1;  -- Drop entire parts (faster)

-- Dead Letter Queue for bad records
CREATE TABLE bluesky.bluesky_dlq (...)
ENGINE = MergeTree
ORDER BY (kind, scrape_ts);

-- Route good records to dedup
CREATE MATERIALIZED VIEW bluesky.bluesky_dedup_mv TO bluesky.bluesky_dedup AS
SELECT * FROM bluesky.bluesky_raw
WHERE abs(dateDiff('second', scrape_ts, bluesky_ts)) < 1200;  -- Within 20 min

-- Route bad records to DLQ
CREATE MATERIALIZED VIEW bluesky.bluesky_dlq_mv TO bluesky.bluesky_dlq AS
SELECT * FROM bluesky.bluesky_raw
WHERE abs(dateDiff('second', scrape_ts, bluesky_ts)) >= 1200;
```

**Gold Layer (Aggregates + Lookups)**:
```sql
-- Main queryable table (monthly partitions)
CREATE TABLE bluesky.bluesky (
    data JSON(SKIP ...),
    kind LowCardinality(String),
    bluesky_ts DateTime64(6),
    _rmt_partition_id LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toStartOfInterval(bluesky_ts, toIntervalMonth(1))
ORDER BY (kind, bluesky_ts);

-- Partition tracking for incremental refresh
CREATE TABLE bluesky.latest_partition (
    partition_id SimpleAggregateFunction(max, UInt32)
)
ENGINE = AggregatingMergeTree
ORDER BY tuple();

-- Refreshable MV for deduplication
CREATE MATERIALIZED VIEW bluesky.blue_sky_dedupe_rmv
REFRESH EVERY 20 MINUTE APPEND TO bluesky.bluesky
AS WITH ...
SETTINGS do_not_merge_across_partitions_select_final = 1;
```

### 2.4 Aggregation Tables (AggregatingMergeTree Patterns)

```sql
-- Simple counter aggregation
CREATE TABLE bluesky.events_per_hour_of_day (
    event LowCardinality(String),
    hour_of_day UInt8,
    count SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY (event, hour_of_day);

-- Counter + distinct users
CREATE TABLE bluesky.top_post_types (
    collection LowCardinality(String),
    posts SimpleAggregateFunction(sum, UInt64),
    users AggregateFunction(uniq, String)  -- State-based distinct
)
ENGINE = AggregatingMergeTree
ORDER BY collection;

-- Per-entity counters
CREATE TABLE bluesky.likes_per_post (
    cid String,
    likes SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY (cid);

CREATE TABLE bluesky.likes_per_user (
    did String,
    likes SimpleAggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY (did);
```

### 2.5 Dictionary Patterns

```sql
-- Lookup table with ReplacingMergeTree
CREATE TABLE bluesky.handle_per_user (
    did String,
    handle String
)
ENGINE = ReplacingMergeTree
ORDER BY (did);

-- Dictionary from ClickHouse table
CREATE DICTIONARY bluesky.handle_per_user_dict (
    did String,
    handle String
)
PRIMARY KEY (did)
SOURCE(CLICKHOUSE(
    QUERY 'SELECT did, handle FROM bluesky.handle_per_user FINAL'
))
LIFETIME(MIN 300 MAX 360)      -- Refresh every 5-6 minutes
LAYOUT(complex_key_hashed());  -- Hash layout for string keys

-- Usage in queries
SELECT
    did,
    dictGet('bluesky.handle_per_user_dict', 'handle', did) AS handle
FROM bluesky.posts;
```

### 2.6 Separate Index Tables

```sql
-- CID-to-text lookup (avoids scanning JSON)
CREATE TABLE bluesky.cid_to_text (
    cid String,
    did String,
    text String,
    about_clickhouse Boolean
)
ENGINE = MergeTree
ORDER BY (cid);

-- Query text without touching main table
SELECT text
FROM bluesky.cid_to_text
WHERE cid = 'abc123';
```

### 2.7 Query Functions & Patterns

**Formatting Functions**:
```sql
formatReadableSize(bytes)      -- '1.23 GiB'
formatReadableQuantity(num)    -- '1.23 million'
bar(value, min, max, width)    -- ASCII bar chart
```

**Window Functions**:
```sql
lagInFrame(column)             -- Previous row without sort overhead
ROW_NUMBER() OVER (PARTITION BY x ORDER BY y)
```

**Geospatial**:
```sql
geoDistance(lon1, lat1, lon2, lat2)
pointInPolygon((x, y), [(x1,y1), (x2,y2), ...])
geohashEncode(lon, lat, precision)
geohashDecode('hash')
```

**Aggregation States**:
```sql
avgState(column)               -- Store average state
maxSimpleState(column)         -- Store max state
quantileTDigestState(0.95)(column)  -- Store quantile state
uniqState(column)              -- Store HyperLogLog state
```

### 2.8 OpenTelemetry Patterns

**Schema Tables** (inferred from script):
- otel_logs
- otel_traces
- otel_traces_1m (aggregated)
- otel_metrics_gauge
- otel_metrics_histogram
- otel_metrics_sum
- otel_metrics_summary
- otel_metrics_exponential_histogram

**Temporal Adjustment Pattern**:
```sql
-- Shift timestamps to current time (for demo data)
INSERT INTO otel_v2.otel_logs
SELECT
    Timestamp + toIntervalNanosecond(
        dateDiff('nanosecond', max_ts, now64(9))
    ) AS Timestamp,
    ...
FROM otel_v2_source.otel_logs;
```

### Key Insights

1. **S3Queue engine** - Continuous ingestion from S3/GCS with ordered processing
2. **JSON SKIP clause** - Skip nested fields to reduce storage
3. **MATERIALIZED columns** - Compute once at insert time (kind, dedup_hash)
4. **cityHash64 for dedup** - Fast hash for deduplication keys
5. **isValidJSON()** - Filter malformed JSON before processing
6. **dateDiff for validation** - Route bad records to DLQ
7. **ttl_only_drop_parts** - Faster TTL by dropping entire parts
8. **do_not_merge_across_partitions_select_final** - Optimize FINAL
9. **EXCHANGE TABLES** - Atomic swap for zero-downtime updates
10. **SimpleAggregateFunction** - Simpler than full AggregateFunction for sum/max/min
11. **complex_key_hashed** - Dictionary layout for string keys
12. **Separate index tables** - Avoid scanning large JSON for lookups
13. **REFRESH EVERY + APPEND** - Incremental MV refresh
14. **max_insert_threads** - Parallelize inserts
15. **Medallion architecture** - Bronze → Silver → Gold with MVs

### 2.9 RubyGems S3Queue Advanced Patterns

**Complex Tuple Types for Nested Data**:
```sql
CREATE TABLE rubygems.downloads_queue (
    timestamp DateTime,
    request_path String,
    request_query String,
    -- Nested Tuple for structured user agent
    user_agent Tuple(
        agent_name String,
        agent_version String,
        bundler String,
        ci String,
        command String,
        jruby String,
        options String,
        platform Tuple(cpu String, os String, version String),  -- Nested Tuple!
        ruby String,
        rubygems String,
        truffleruby String
    ),
    tls_cipher String,
    time_elapsed Int64,
    client_continent String,
    client_country String,
    client_region String,
    client_city String,
    client_latitude String,
    client_longitude String,
    client_timezone String,
    client_connection String,
    request String,
    request_host String,
    request_bytes Int64,
    http2 Bool,
    tls Bool,
    tls_version String,
    response_status Int64,
    response_text String,
    response_bytes Int64,
    response_cache String,
    cache_state String,
    cache_lastuse Float64,
    cache_hits Int64,
    server_region String,
    server_datacenter String,
    gem String,
    version String,
    platform String
)
ENGINE = S3Queue(
    '${s3_path}',
    '${S3_KEY}',
    '${S3_SECRET}',
    'JSONEachRow',
    'gzip'
)
SETTINGS
    mode = 'unordered',                        -- Process files in any order
    s3queue_polling_min_timeout_ms = 1800000,  -- 30 min minimum poll interval
    s3queue_polling_max_timeout_ms = 2400000,  -- 40 min maximum poll interval
    s3queue_tracked_files_limit = 2000;        -- Track up to 2000 files
```

**Daily Queue Rotation Pattern**:
```sql
-- Create new queue for today
CREATE TABLE rubygems.downloads_queue_${today} (...) ENGINE = S3Queue(...);

-- Create MV to route to main table
CREATE MATERIALIZED VIEW rubygems.downloads_${today}_mv
TO rubygems.downloads AS
SELECT * FROM rubygems.downloads_queue_${today};

-- Drop yesterday's queue and MV (cleanup)
DROP TABLE IF EXISTS rubygems.downloads_queue_${yesterday};
DROP TABLE IF EXISTS rubygems.downloads_${yesterday}_mv;
```

**Key S3Queue Settings**:
| Setting | Value | Purpose |
|---------|-------|---------|
| `mode` | 'unordered' | Process files in any order (faster) |
| `mode` | 'ordered' | Process files in lexicographic order |
| `s3queue_polling_min_timeout_ms` | 1800000 | Min wait between polls (30 min) |
| `s3queue_polling_max_timeout_ms` | 2400000 | Max wait between polls (40 min) |
| `s3queue_tracked_files_limit` | 2000 | Max files to track as processed |
| `s3queue_buckets` | 30 | Parallel processing buckets |
| `s3queue_processing_threads_num` | 10 | Parallel processing threads |

### 2.10 Dataset Catalog (171 Tables, 40 Databases)

**Available Datasets**:
| Database | Tables | Use Case |
|----------|--------|----------|
| amazon | 1 | Customer product reviews |
| covid | 1 | Epidemiological pandemic data |
| environmental | 1 | Global sensor network measurements |
| food | 6 | Recipes, menus, nutritional facts |
| forex | 5 | Currency exchange rates |
| geo | 2 | Cell towers, geographic locations |
| git | 8 | GitHub/Grafana repository commits |
| hackernews | 2 | News aggregation platform |
| imdb | 8 | Movie/actor/director information |
| metrica | 5 | Web analytics datasets |
| noaa | 11 | Weather measurements, ski resorts |
| nyc_taxi | 1 | 2009 taxi ride data |
| nypd | 1 | NYC crime complaint records |
| opensky | 1 | Air traffic during COVID-19 |
| reddit | 1 | Public comment discussions |
| stackoverflow | 9 | Q&A platform content |
| star_schema | 20 | Benchmark datasets |
| stock | 1 | Financial market data |
| uk | 5 | UK property prices and postal codes |
| wiki | 5 | Wikipedia statistics and traffic |
| rubygems | 1 | Ruby package download metrics |

---

## 3. ClickPy (clickgems branch)

**Source**: [GitHub Repo](https://github.com/ClickHouse/clickpy/tree/clickgems)

**Use Case**: PyPI & RubyGems package download analytics - 600+ billion rows, ~50GB compressed

### 3.1 Core Schema Pattern

```sql
-- Main downloads table (simple, optimized for inserts)
CREATE TABLE pypi.pypi (
    date Date,
    country_code LowCardinality(String),
    project String,
    type LowCardinality(String),
    installer LowCardinality(String),
    python_minor LowCardinality(String),
    system LowCardinality(String),
    version String
)
ENGINE = MergeTree
ORDER BY (project, date, version, country_code, python_minor, system)
```

### 3.2 Dimensional Aggregation Tables (SummingMergeTree)

```sql
-- Total downloads per project
CREATE TABLE pypi.pypi_downloads (
    project String,
    count Int64
)
ENGINE = SummingMergeTree
ORDER BY project

CREATE MATERIALIZED VIEW pypi.pypi_downloads_mv TO pypi.pypi_downloads AS
SELECT project, count() AS count
FROM pypi.pypi
GROUP BY project

-- Downloads by multiple dimensions
CREATE TABLE pypi.pypi_downloads_per_day_by_version_by_country (
    date Date,
    project String,
    version String,
    country_code String,
    count Int64
)
ENGINE = SummingMergeTree
ORDER BY (project, version, date, country_code)

CREATE MATERIALIZED VIEW pypi.pypi_downloads_per_day_by_version_by_country_mv
TO pypi.pypi_downloads_per_day_by_version_by_country AS
SELECT date, project, version, country_code, count() as count
FROM pypi.pypi
GROUP BY date, project, version, country_code
```

**Pattern**: Create multiple SummingMergeTree tables for different query patterns:
- `pypi_downloads` - total by project
- `pypi_downloads_by_version` - by project + version
- `pypi_downloads_per_day` - by project + date
- `pypi_downloads_per_day_by_version` - by project + version + date
- `pypi_downloads_per_day_by_version_by_country` - add country
- `pypi_downloads_per_day_by_version_by_python` - add python version
- `pypi_downloads_per_day_by_version_by_system` - add OS
- `pypi_downloads_per_day_by_version_by_installer_by_type` - add installer
- ... and more combinations

### 3.3 Min/Max Date Tracking (AggregatingMergeTree)

```sql
CREATE TABLE pypi.pypi_downloads_max_min (
    project String,
    max_date SimpleAggregateFunction(max, Date),
    min_date SimpleAggregateFunction(min, Date)
)
ENGINE = AggregatingMergeTree
ORDER BY project

CREATE MATERIALIZED VIEW pypi.pypi_downloads_max_min_mv TO pypi.pypi_downloads_max_min AS
SELECT
    project,
    maxSimpleState(date) as max_date,
    minSimpleState(date) as min_date
FROM pypi.pypi
GROUP BY project
```

### 3.4 Dictionaries for Lookups

```sql
-- Last updated dictionary (computed from query)
CREATE DICTIONARY pypi.last_updated_dict (
    name String,
    last_update DateTime64(3)
)
PRIMARY KEY name
SOURCE(CLICKHOUSE(QUERY 'SELECT name, max(upload_time) AS last_update FROM pypi.projects GROUP BY name'))
LIFETIME(MIN 0 MAX 300)
LAYOUT(COMPLEX_KEY_HASHED())

-- Country code dictionary (from table)
CREATE DICTIONARY pypi.countries_dict (
    name String,
    code String
)
PRIMARY KEY code
SOURCE(CLICKHOUSE(DATABASE 'pypi' TABLE 'countries'))
LIFETIME(MIN 0 MAX 300)
LAYOUT(COMPLEX_KEY_HASHED())
```

### 3.5 Custom Functions (UDFs)

```sql
-- Extract GitHub repo from PyPI package
CREATE OR REPLACE FUNCTION getRepoName AS package_name -> (
    WITH (
        SELECT regexpExtract(
            arrayFilter(l -> (l LIKE '%https://github.com/%'),
                arrayConcat(project_urls, [home_page]))[1],
            '.*https://github\\.com/([^/]+/[^/]+)'
        )
        FROM pypi.projects
        WHERE name = package_name
            AND length(arrayFilter(l -> (l LIKE '%https://github.com/%'),
                arrayConcat(project_urls, [home_page]))) >= 1
        ORDER BY upload_time DESC
        LIMIT 1
    ) AS repo
    SELECT repo
)

-- Get GitHub repo ID from package name
CREATE OR REPLACE FUNCTION getRepoId AS package_name -> (
    SELECT CAST(max(CAST(repo_id, 'UInt64')), 'String') AS id
    FROM github.repo_name_to_id
    WHERE repo_name = getRepoName(package_name) AND repo_id != ''
    LIMIT 1
)
```

### 3.6 GitHub Events Schema (Enum8 Pattern)

```sql
CREATE TABLE github.github_events (
    file_time DateTime,
    event_type Enum8(
        'CommitCommentEvent' = 1, 'CreateEvent' = 2, 'DeleteEvent' = 3,
        'ForkEvent' = 4, 'GollumEvent' = 5, 'IssueCommentEvent' = 6,
        'IssuesEvent' = 7, 'MemberEvent' = 8, 'PublicEvent' = 9,
        'PullRequestEvent' = 10, 'PullRequestReviewCommentEvent' = 11,
        'PushEvent' = 12, 'ReleaseEvent' = 13, 'SponsorshipEvent' = 14,
        'WatchEvent' = 15, 'GistEvent' = 16, 'FollowEvent' = 17,
        'DownloadEvent' = 18, 'PullRequestReviewEvent' = 19,
        'ForkApplyEvent' = 20, 'Event' = 21, 'TeamAddEvent' = 22
    ),
    action Enum8(
        'none' = 0, 'created' = 1, 'added' = 2, 'edited' = 3, 'deleted' = 4,
        'opened' = 5, 'closed' = 6, 'reopened' = 7, 'assigned' = 8,
        'unassigned' = 9, 'labeled' = 10, 'unlabeled' = 11,
        'review_requested' = 12, 'review_request_removed' = 13,
        'synchronize' = 14, 'started' = 15, 'published' = 16, 'update' = 17,
        'create' = 18, 'fork' = 19, 'merged' = 20
    ),
    state Enum8('none' = 0, 'open' = 1, 'closed' = 2),
    actor_login LowCardinality(String),
    repo_name LowCardinality(String),
    repo_id LowCardinality(String),
    -- ... more fields
)
ENGINE = MergeTree
ORDER BY (repo_id, event_type, created_at)
```

### 3.7 Complex MV with argMax for Latest Value

```sql
-- Map gem to GitHub repo (extract from metadata JSON)
CREATE MATERIALIZED VIEW rubygems.gem_to_repo_name_mv TO rubygems.gem_to_repo_name AS
WITH latest AS (
    SELECT
        rubygem_id,
        argMax(simpleJSONExtractString(CAST(metadata, 'String'), 'homepage_uri'), created_at) AS homepage_uri,
        argMax(simpleJSONExtractString(CAST(metadata, 'String'), 'source_code_uri'), created_at) AS source_code_uri
    FROM rubygems.versions
    GROUP BY rubygem_id
),
picked AS (
    SELECT r.id, r.name AS gem,
        multiIf(
            positionCaseInsensitive(homepage_uri, 'github.com/') > 0, homepage_uri,
            positionCaseInsensitive(source_code_uri,'github.com/') > 0, source_code_uri,
            ''
        ) AS gh_url
    FROM rubygems.rubygems r
    LEFT JOIN latest v ON r.id = v.rubygem_id
),
norm AS (
    SELECT gem, gh_url,
        substring(gh_url, positionCaseInsensitive(gh_url, 'github.com/') + length('github.com/')) AS path_after_host,
        splitByChar('/', path_after_host) AS segs,
        lower(segs[1]) AS owner_raw,
        lower(replaceRegexpAll(segs[2], '(\\.git)?$', '')) AS repo_raw,
        concat(owner_raw, '/', repo_raw) AS repo_name_norm
    FROM picked
)
SELECT gem, repo_name_norm AS repo_name
FROM norm
WHERE gh_url != ''
    AND owner_raw != '' AND repo_raw != ''
    AND match(repo_name_norm, '^[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+$')
```

### 3.8 Data Loading Settings

```sql
-- Parquet import settings
SET input_format_null_as_default = 1;           -- Treat NULL as column default
SET input_format_parquet_import_nested = 1;     -- Support nested structures
SET max_insert_threads = 16;                    -- Parallel inserts
SET min_insert_block_size_bytes = 0;            -- No byte threshold
SET min_insert_block_size_rows = 10000000;      -- 10M rows per block
```

### 3.9 Blue-Green Deployment (EXCHANGE TABLES)

```sql
-- 1. Clone schema
CREATE TABLE pypi.projects_v2 AS pypi.projects;

-- 2. Load new data with parallel threads
INSERT INTO pypi.projects_v2
SELECT * FROM s3('${path}', '${key}', '${secret}')
SETTINGS max_insert_threads = 16;

-- 3. Atomic swap
EXCHANGE TABLES pypi.projects AND pypi.projects_v2;

-- 4. Cleanup
DROP TABLE pypi.projects_v2;
```

### 3.10 Lightweight Delete for Data Fixes

```sql
-- Enable lightweight delete mode
SET lightweight_delete_mode = 'lightweight_update';

-- Delete specific date from all aggregate tables
ALTER TABLE rubygems.downloads_per_day DELETE
WHERE date = CAST('${TARGET_DATE}', 'Date');

ALTER TABLE rubygems.downloads_per_day_by_version DELETE
WHERE date = CAST('${TARGET_DATE}', 'Date');

-- Delete from main table
ALTER TABLE rubygems.downloads DELETE
WHERE toDate(timestamp) = CAST('${TARGET_DATE}', 'Date');

-- Rebuild monthly aggregate (drop MV, rebuild, exchange)
DROP VIEW rubygems.downloads_per_month_mv;

CREATE TABLE rubygems.downloads_per_month_v2 AS rubygems.downloads_per_month;

INSERT INTO rubygems.downloads_per_month_v2
SELECT toStartOfMonth(date) AS month, gem, sum(count) AS count
FROM rubygems.downloads_per_day
GROUP BY month, gem;

EXCHANGE TABLES rubygems.downloads_per_month_v2 AND rubygems.downloads_per_month;
DROP TABLE rubygems.downloads_per_month_v2;

-- Recreate MV
CREATE MATERIALIZED VIEW rubygems.downloads_per_month_mv
TO rubygems.downloads_per_month AS
SELECT toStartOfMonth(timestamp) AS month, gem, count() AS count
FROM rubygems.downloads
GROUP BY month, gem;
```

### 3.11 User & Role Security

```sql
-- Create role with settings
CREATE ROLE play SETTINGS
    readonly = 1,
    add_http_cors_header = true,
    max_execution_time = 60.0,
    max_rows_to_read = 10000000000,
    max_bytes_to_read = 1000000000000,
    max_network_bandwidth = 25000000,
    max_memory_usage = 20000000000,
    max_bytes_before_external_group_by = 10000000000,
    max_result_rows = 1000,
    max_result_bytes = 10000000,
    result_overflow_mode = 'break',
    enable_http_compression = 1;

-- Create user with role
CREATE USER play
IDENTIFIED WITH double_sha1_hash BY 'hash...'
DEFAULT ROLE play
SETTINGS enable_http_compression = 1 CHANGEABLE_IN_READONLY;

-- Grant permissions (including dictGet)
GRANT SELECT ON pypi.* TO play;
GRANT dictGet ON pypi.* TO play;
```

### 3.12 Incremental Loading Pattern

```sql
-- Get max date for incremental backfill
SELECT CAST(CAST(max(date) + toIntervalDay(1), 'DateTime'), 'Int64')
FROM pypi.pypi_downloads_per_day;

-- Get max file_time for GitHub events
SELECT max(file_time) FROM github.github_events;

-- Insert new data with format
INSERT INTO github.github_events FORMAT JSONCompactEachRow
```

### 3.13 URL Table Function for External Data

```sql
-- Load country codes from GitHub Gist
INSERT INTO pypi.countries
SELECT name, `alpha-2` AS code
FROM url('https://gist.githubusercontent.com/.../country_codes.csv');
```

### Key Insights

1. **SummingMergeTree for every dimension** - Create multiple aggregate tables covering different query patterns
2. **Enum8 for categorical data** - Saves space vs LowCardinality for known values
3. **argMax for latest value** - Get most recent value per group without window functions
4. **simpleJSONExtractString** - Fast JSON extraction without full parsing
5. **Custom UDFs** - Encapsulate complex logic (getRepoName, getRepoId)
6. **lightweight_delete_mode** - Fast deletes for data fixes
7. **Rebuild + EXCHANGE** - Atomic rebuild of aggregate tables
8. **max_insert_threads = 16** - Parallel S3 ingestion
9. **LIFETIME(MIN 0 MAX 300)** - Dictionary refresh every 5 minutes
10. **CHANGEABLE_IN_READONLY** - Allow users to modify specific settings

---

## Summary: New Patterns Found

### From reversedns.space

| Pattern | Already in Docs? |
|---------|------------------|
| Empty ORDER BY `()` | ❌ No |
| Native IPv4/IPv6 types | ❌ No |
| Bool type for flags | ❌ No |
| JSONExtract with typed Tuple | ❌ No |
| mortonDecode for spatial | ❌ No |
| WITH FILL clause | ❌ No |
| force_primary_key setting | ❌ No |
| KEYED BY ip_address quotas | ❌ No |
| RANDOMIZED interval quotas | ❌ No |
| READONLY user settings | ❌ No |
| add_http_cors_header | ❌ No |
| allow_experimental_analyzer | ❌ No |
| byteSwap for endianness | ❌ No |
| reinterpretAsFixedString | ❌ No |
| sipHash64 for distribution | ⚠️ Partial (sharding) |

### From sql.clickhouse.com

| Pattern | Already in Docs? |
|---------|------------------|
| SETTINGS PROFILE with limits | ❌ No |
| DEFAULT ROLE for users | ❌ No |
| Column-level GRANT | ❌ No |
| GRANT dictGet permissions | ❌ No |
| EXCHANGE TABLES (atomic swap) | ⚠️ Partial (mentioned) |
| max_insert_threads setting | ❌ No |
| S3Queue engine | ❌ No |
| S3Queue mode (ordered/unordered) | ❌ No |
| S3Queue polling timeout settings | ❌ No |
| S3Queue tracked_files_limit | ❌ No |
| Nested Tuple types | ❌ No |
| Daily queue rotation pattern | ❌ No |
| JSON SKIP clause | ❌ No |
| MATERIALIZED columns | ❌ No |
| cityHash64 for dedup | ❌ No |
| isValidJSON() function | ❌ No |
| dateDiff for record routing | ❌ No |
| ttl_only_drop_parts setting | ❌ No |
| do_not_merge_across_partitions_select_final | ❌ No |
| SimpleAggregateFunction | ⚠️ Partial (OHLCV) |
| complex_key_hashed dictionary layout | ❌ No |
| Separate index tables pattern | ❌ No |
| REFRESH EVERY + APPEND | ❌ No |
| Medallion architecture (Bronze/Silver/Gold) | ❌ No |
| DLQ (Dead Letter Queue) pattern | ❌ No |
| input_format_null_as_default | ❌ No |
| date_time_input_format best_effort | ❌ No |
| formatReadableSize/Quantity | ❌ No |
| lagInFrame window function | ❌ No |
| Geospatial functions (geoDistance, pointInPolygon) | ❌ No |
| Aggregation state functions (*State) | ⚠️ Partial |
| parseDateTimeBestEffort | ❌ No |
| arrayJoin + range for sequences | ❌ No |

### From CryptoHouse (Solana-specific) → See `12-cryptohouse-solana-reference.md`

| Pattern | Already in Docs? |
|---------|------------------|
| Null engine staging tables | ❌ No |
| CAST for string → Array(Tuple) | ❌ No |
| DateTime64(6) microsecond precision | ❌ No |
| Decimal(38,9) for lamports | ❌ No |
| Decimal(76,38) for token amounts | ❌ No |
| generateULID() for unique IDs | ❌ No |
| arrayExists with lambda filtering | ❌ No |
| Vote transaction filtering pattern | ❌ No |
| uniqCombinedState(14) HyperLogLog | ⚠️ Partial |
| base58Decode for Solana data | ❌ No |
| ARRAY JOIN for log parsing | ❌ No |
| splitByString for log extraction | ❌ No |
| ROW_NUMBER() OVER deduplication | ❌ No |
| anyLast() OVER for OHLC close price | ❌ No |
| use_query_cache setting | ❌ No |
| max_bytes_before_external_group_by | ❌ No |
| Polymorphic account schema | ❌ No |
| Array(Tuple) for nested blockchain data | ❌ No |

### From ClickPy (clickgems branch)

| Pattern | Already in Docs? |
|---------|------------------|
| Multi-dimensional SummingMergeTree aggregates | ❌ No |
| SimpleAggregateFunction(max/min, Date) | ❌ No |
| maxSimpleState / minSimpleState | ❌ No |
| Custom UDFs (CREATE FUNCTION AS) | ❌ No |
| Enum8 for categorical event types | ❌ No |
| argMax for latest value per group | ❌ No |
| simpleJSONExtractString (fast JSON) | ❌ No |
| lightweight_delete_mode setting | ❌ No |
| DROP VIEW + rebuild + EXCHANGE pattern | ❌ No |
| input_format_parquet_import_nested | ❌ No |
| min_insert_block_size_rows setting | ❌ No |
| GRANT dictGet permission | ❌ No |
| CHANGEABLE_IN_READONLY setting | ❌ No |
| regexpExtract function | ❌ No |
| multiIf for conditional logic | ❌ No |
| positionCaseInsensitive function | ❌ No |
| match() for regex validation | ❌ No |
| url() table function | ⚠️ Partial (s3) |
| max_insert_threads = 16 | ⚠️ Partial |
| EXCHANGE TABLES | ⚠️ Partial |

### From StockHouse (Real-Time Trading) → Added to `12-cryptohouse-solana-reference.md`

| Pattern | Already in Docs? |
|---------|------------------|
| Time-bucketed ORDER BY `(sym, t - (t % 60000))` | ❌ No |
| AggregateFunction(argMin, Float64, UInt64) | ❌ No |
| AggregateFunction(argMax, Float64, UInt64) | ❌ No |
| argMinState/argMaxState for OHLC | ❌ No |
| argMinMerge/argMaxMerge queries | ❌ No |
| toUnixTimestamp64Milli(now64()) | ❌ No |
| fromUnixTimestamp64Milli(t, 'UTC') | ❌ No |
| toStartOfInterval for candlesticks | ❌ No |
| toDateTime64(..., 3) precision | ❌ No |
| PARTITION BY toYYYYMM(event_date) | ⚠️ Partial |
| LZ4 compression (Go client) | ❌ No |
| 50,000 batch size optimization | ❌ No |
| Backpressure/load shedding (95%) | ❌ No |
| Arrow output format | ❌ No |
| JSONCompact output format | ❌ No |
| inserted_at DEFAULT pattern | ❌ No |

---

## Blogs/Repos to Research

- [x] reversedns.space ✅ (15 patterns)
- [x] sql.clickhouse.com ✅ (35+ patterns)
  - [x] setup.sql (users, roles, quotas)
  - [x] queries.json (query patterns)
  - [x] load_scripts/github_events
  - [x] load_scripts/pypi
  - [x] load_scripts/ontime
  - [x] load_scripts/hackernews
  - [x] load_scripts/bluesky (medallion architecture)
  - [x] load_scripts/otel_v2
  - [x] load_scripts/rubygems (S3Queue advanced)
  - [x] load_scripts/hyperdx (no SQL - Python only)
  - [x] table_comments.json (171 tables catalog)
- [x] CryptoHouse ✅ → **Separate doc: `12-cryptohouse-solana-reference.md`**
  - [x] setup.sql (Solana schemas, staging pattern, aggregation tables)
  - [x] queries.json (32 queries: OHLC, compute units, balance changes, sentiment)
  - Solana-specific: blocks, transactions, accounts, tokens, token_transfers, instructions
  - Key patterns: Null engine staging, arrayExists vote filtering, Decimal precision
- [x] ClickPy (clickgems branch) ✅ (20+ patterns)
  - [x] ClickHouse.md (full PyPI schema, MVs, dictionaries)
  - [x] scripts/create_tables.sh (RubyGems + PyPI + GitHub schemas)
  - [x] scripts/create_user.sh (roles, permissions, dictGet)
  - [x] scripts/load.sh (Parquet import settings)
  - [x] scripts/load_projects_previous.sh (EXCHANGE TABLES pattern)
  - [x] scripts/clickgems-day-fix.sh (lightweight delete, rebuild pattern)
  - [x] scripts/update_github.sh (incremental loading)
  - [x] scripts/populate.sh (url() table function)
  - [x] src/utils/clickhouse.js (client config, allow_experimental_analyzer)
  - Key patterns: Multi-dim aggregates, UDFs, Enum8, argMax, lightweight deletes
- [x] StockHouse ✅ (15+ patterns) → **Added to `12-cryptohouse-solana-reference.md`**
  - [x] scripts/schema.sql (quotes, trades, crypto tables, daily aggregations)
  - [x] frontend/src/queries/index.js (OHLC candlestick queries)
  - [x] ingester-go/ingest.go (batch settings, LZ4 compression, backpressure)
  - [x] api/query.js (Arrow/JSONCompact format)
  - [x] frontend/src/composables/useClickhouse.js (health check)
  - Key patterns: Time-bucketed ORDER BY, argMinState/argMaxState OHLC, 50k batch inserts
- [ ] [Next blog/repo URL]

---

## Sources

- [reversedns.space GitHub](https://github.com/ClickHouse/reversedns.space)
- [sql.clickhouse.com GitHub](https://github.com/ClickHouse/sql.clickhouse.com)
- [Bluesky Medallion DDL](https://github.com/ClickHouse/sql.clickhouse.com/tree/main/load_scripts/bluesky/clickhouse_ddl/medallion)
- [RubyGems Ingest Script](https://github.com/ClickHouse/sql.clickhouse.com/blob/main/load_scripts/rubygems/ingest.sh)
- [CryptoHouse GitHub](https://github.com/ClickHouse/CryptoHouse) → See `12-cryptohouse-solana-reference.md`
- [ClickPy GitHub (clickgems)](https://github.com/ClickHouse/clickpy/tree/clickgems)
- [StockHouse GitHub](https://github.com/ClickHouse/stockhouse) → See `12-cryptohouse-solana-reference.md`
