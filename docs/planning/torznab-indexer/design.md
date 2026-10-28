# Torznab Indexer and mediactl Refactor

Status: approved design, not yet implemented.
Date: 2026-09-30

## Overview

Two related changes, delivered as one specification in two independent phases.

Phase 1 splits the `media` CLI into an importable `mediactl` package covered by
pytest, without changing behaviour.

Phase 2 adds a self-hosted Torznab indexer service: it crawls configured
sources into a local database and exposes the Torznab API that Prowlarr,
Sonarr and Radarr already consume.

Phase 1 comes first because Phase 2 adds `./media index` subcommands, and
writing them into the current single-file script means moving them again.

## Goals

- `media` logic lives in small, importable modules with automated tests.
- Prowlarr can add the new service as a Generic Torznab indexer and receive
  results from sources described entirely in configuration.
- Adding a source requires writing a YAML file, not Python code.
- The host keeps its current requirements: Python 3.10 or newer, no package
  installation needed to run `./media`.

## Non-goals

- Replacing Prowlarr. It keeps aggregating indexers and syncing them to Sonarr
  and Radarr.
- Live proxying. Searches read the local database only.
- A scheduler. Crawls run when a command runs.
- A web interface for the indexer.
- Shipping source definitions for sites that distribute unauthorised copies of
  copyrighted works. The engine is source-agnostic; definitions maintained in
  this repository target sources that publish their own torrents.

## Decisions

| Decision | Rationale |
| --- | --- |
| Torznab behind Prowlarr | Prowlarr already handles aggregation, history and app sync. |
| Crawl into a local database | Search latency and availability stop depending on the source. |
| TinyDB with JSON storage | Inspectable by hand, no service to run. Replaceable behind one module. |
| Command-driven crawl | No daemon, no scheduler, no partial-failure recovery logic. |
| Declarative YAML definitions | A new source is configuration. Matches the repository's declarative preference. |
| Container in `compose.yml` | Consistent with every other service in the stack. |
| Fake executables on `PATH` for tests | No test seam in production code; assertions cover the real argv. |
| uv for development and image builds only | `./media` keeps running on the system interpreter. |

## Phase 1: mediactl package and tests

### Module layout

```text
media                    # entry point: shebang + from mediactl.cli import main
mediactl/
  __init__.py
  process.py             # MediaError, run, require_commands
  env.py                 # read_env, validate_env, validate_number, key tuples
  compose.py             # detect_hardware, selected_hardware, validate_hardware,
                         # validate_vpn, compose_command, validated_context
  qbittorrent.py         # wait_for, login, session, temporary password, configure
  commands.py            # command_* functions, print_urls
  cli.py                 # build_parser, main
```

`media` stays executable and keeps its current path, so the documented
`./media setup` continues to work. Running it puts the repository root on
`sys.path`, so `mediactl` imports without installation.

`mediactl` must import only the standard library, and must remain compatible
with Python 3.10. A test asserts the standard-library constraint.

### Test harness

```text
tests/
  conftest.py
  fakes/docker
  fakes/git
  fixtures/
  test_cli_*.py          # characterisation, through a subprocess
  test_env.py
  test_compose.py
  test_qbittorrent.py
```

A fixture copies `media`, `mediactl/`, `compose*.yml` and `.env.example` into
`tmp_path` and runs from there. This is required because `ROOT` is the script's
own directory, and it keeps `create_directories` away from real storage.

A `bin/` directory inside `tmp_path` is prepended to `PATH` and holds fake
`docker` and `git` executables. Each appends its received argv to the file named
by `MEDIA_TEST_CALLS` and writes canned stdout driven by environment variables.
Assertions read that log.

`temporary_qbittorrent_password` runs `compose logs` through `run`, so the fake
`docker` prints a log line containing a temporary password and the existing
regex extracts it.

The qBittorrent Web API is HTTP, not an executable. A `ThreadingHTTPServer`
from the standard library binds port 0, and the assigned port is written into
the test `.env`. `qbittorrent_url` builds its URL from `ADMIN_BIND_ADDRESS` and
`QBITTORRENT_PORT`, so no patching is needed.

Tests invoke `media` through `sys.executable` rather than its shebang, because
the harness manipulates `PATH`.

### Known gap

`detect_hardware` checks `nvidia-smi` on `PATH`, which the fakes cover, and the
existence of `/dev/dri/renderD128`, which they cannot. Command-level tests pass
`--hardware` explicitly, which bypasses detection. `detect_hardware` itself gets
a direct unit test once it is importable.

### Order of work

1. Write characterisation tests against the current `media`. They must pass
   before anything moves.
2. Move code into `mediactl/` with no signature changes.
3. The same tests, unchanged, must still pass.
4. Add unit tests for the pure functions.

## Phase 2: Torznab indexer

### Layout

```text
indexer/
  Dockerfile
  __init__.py
  app.py                 # Flask app, /api routing, apikey check
  torznab.py             # caps and RSS rendering, category matching
  storage.py             # search, upsert
  definitions.py         # loading and validation
  extract.py             # html and json extractors, parse helpers
  crawl.py               # pipeline
  cli.py                 # sources, crawl, check
  definitions/
```

### Service

A Flask application listening on `0.0.0.0:9117` inside the container, serving a
single `/api` endpoint. Every request requires the `apikey` parameter to match
`INDEXER_API_KEY`; a mismatch returns the Torznab error document, not a bare
HTTP status.

### Torznab contract

| `t` | Behaviour |
| --- | --- |
| `caps` | Capabilities document, derived from the loaded definitions. |
| `search` | Free text through `q`, filtered by `cat`, paged by `limit` and `offset`. |
| `movie` | Same as `search`. |
| `tvsearch` | Same as `search`. |

`caps` advertises only `q` as a supported search parameter, and only the
categories that some loaded definition declares. This is deliberate: releases
are not indexed with `imdbid`, `tmdbid`, `tvdbid`, `season` or `ep`, and a client
that believes those are supported silently receives wrong results. Advertising
only `q` makes Sonarr and Radarr compose the query themselves and filter the
returned titles, which they already do.

Each result item carries `title`, `guid`, `link`, `pubDate`, `size`, and
`torznab:attr` entries for `category`, `seeders`, `peers`, `downloadvolumefactor`,
`uploadvolumefactor`, and `magneturl` with `infohash` when a magnet is known.

Three constraints that decide whether clients accept the results:

- `size` in bytes is mandatory in practice. Sonarr and Radarr discard releases
  without a plausible size.
- `peers` is seeders plus leechers, not leechers alone.
- Categories are Newznab numeric ids. 2000 covers movies (2040 HD, 2045 UHD,
  2050 BluRay), 5000 covers TV (5040 HD, 5070 anime).

### Search semantics

`q` is normalised the same way as the stored `search_title`: lowercased,
punctuation replaced by spaces, runs of whitespace collapsed. A release matches
when every query token appears in its `search_title`.

A requested category matches a release category when the two are equal, or when
the requested id is a parent: `requested % 1000 == 0` and
`release // 1000 == requested // 1000`. Without the parent rule, a client asking
for 2000 receives nothing.

Results are ordered by seeders descending, then `published_at` descending.

### Data model

Table `releases`, one document per release:

```json
{
  "id": "0123456789abcdef0123456789abcdef01234567",
  "title": "Some Release Name 2024 1080p",
  "search_title": "some release name 2024 1080p",
  "magnet": "magnet:?xt=urn:btih:0123...",
  "torrent_url": null,
  "size": 8589934592,
  "seeders": 24,
  "leechers": 3,
  "categories": [2040],
  "published_at": "2026-09-29T18:00:00Z",
  "source": "example-source",
  "source_url": "https://example.org/item/123",
  "indexed_at": "2026-09-30T08:14:52Z"
}
```

At least one of `magnet` and `torrent_url` is required. Many sources that
publish their own torrents expose a `.torrent` file and no magnet, and deriving
an infohash from a `.torrent` means downloading and hashing it, which is too
expensive at index time.

`id` is the deduplication key: the infohash when a magnet exists, otherwise
`sha1(source + source_url)`. Infohashes arriving as base32 are normalised to
lowercase hex during ingestion, because the same release otherwise lands twice.

`search_title` is precomputed. TinyDB has no full-text index, and this is the
field an SQLite FTS5 column would later replace.

Ingestion upserts by `id`, updating `title`, `size`, `seeders`, `leechers`,
`categories` and `indexed_at`. Crawls are incremental and paged, so replacing a
source's rows wholesale would delete what the current run did not reach.

### Storage and concurrency

`mediactl` never touches this database; only the container does.

TinyDB's JSON storage rewrites the whole file per write. The crawl writes to a
temporary file in the same directory and finishes with `os.replace`, which is
atomic on POSIX, so a concurrent reader never observes partial JSON. The read
path reloads when the file's mtime changes.

One module exposes two functions, `search` and `upsert`. No abstract base class,
protocol or factory: the point is that the storage choice is confined to one
file, not that it is pluggable.

The database file lives at `/config/releases.json`, inside
`${APPDATA_DIR}/indexer`, which `./media backup` already archives because it
tars `APPDATA_DIR` whole.

### Definition format

```yaml
id: example-source
name: Example Source
base_url: https://example.org
kind: html                      # html | json

request:
  delay_seconds: 1.5
  timeout_seconds: 20

listing:
  url: "/releases?page={page}"
  page_start: 1
  page_limit: 5
  rows: "div.release-list > article"

fields:
  title:        { selector: "h2 a", attribute: text }
  source_url:   { selector: "h2 a", attribute: href }
  published_at: { selector: "time", attribute: datetime, parse: datetime }
  category:     { selector: "span.cat", attribute: text }

detail:
  url_from: source_url
  fields:
    magnet:  { selector: "a[href^='magnet:']", attribute: href }
    size:    { selector: "li.size", attribute: text, parse: size }
    seeders: { selector: "li.seeders", attribute: text, parse: int }

categories:
  "Movies HD": [2040]
  "Movies 4K": [2045]
default_categories: [2000]
```

Available `parse` values are `text` (default), `int`, `size`, `datetime`, and
`regex` with a single capture group for values embedded in surrounding text.

A field may carry `template` instead of `selector`, interpolating fields already
extracted for that row:

```yaml
  torrent_url:
    template: "/download/{identifier}/{identifier}_archive.torrent"
```

This exists because some sources never expose a download link at all: the URL is
constructed from an item identifier. Fields are extracted in declaration order,
so a template may only reference fields declared before it.

With `kind: json`, `rows` and `selector` are dotted paths into the decoded
document instead of CSS selectors. The pipeline is identical; only the extractor
changes. This is not speculative: the reference definition uses it.

The `detail` block exists because listings rarely expose a magnet or a size.
Without it the format would cover only the easy case, which is what the rejected
Python-plugin approach was for.

Robots.txt is always honoured and the request delay always applies. Neither is
configurable, to avoid a second code path.

Definitions load from two directories: `indexer/definitions/` in this
repository, and the host directory mounted at `/definitions`. On an `id`
collision the mounted directory wins, so `git pull` never overwrites locally
maintained definitions.

Definitions are validated on load. An invalid definition fails loudly, naming
the file and the offending field.

`indexer/definitions/` ships one reference definition, targeting the Internet
Archive `advancedsearch` JSON API over a public-domain film collection. It was
chosen because it exercises the three least obvious paths at once: `kind: json`,
a `torrent_url` built by `template` with no magnet anywhere, and category
mapping. It also serves as the worked example in `docs/indexer.md`.

Its exact field paths are confirmed during implementation with
`./media index check`, not assumed from this document.

### Crawl pipeline

```text
load definitions
for each source:
  for each page:
    fetch listing, extract rows
    for each row:
      extract fields
      fetch and merge detail when configured
      normalise size, id, search_title, timestamps
      validate required fields, otherwise skip and count
  upsert collected releases
report per-source counts
```

Paging stops at `page_limit`, or when a page yields zero rows. The zero-row stop
prevents unbounded paging when a source changes its layout.

### Error isolation

A malformed row is skipped and counted; it never aborts the page. A failing
source never aborts the other sources. The command persists what succeeded and
exits non-zero if any source failed.

Declarative scrapers break whenever a source changes, so the report names the
definition file and the field whose selector stopped matching.

### CLI subcommands

```text
./media index sources              # loaded definitions and validation state
./media index crawl [source ...]   # crawl all, or the named sources
./media index check <file.yml>     # dry run: print extracted rows, write nothing
```

Each wraps `docker compose exec indexer python -m indexer.cli <subcommand>`,
so TinyDB never becomes a host dependency.

`check` matters more than it appears: writing selectors is trial and error, and
without it the feedback loop becomes crawling and inspecting JSON.

### Packaging

`indexer/Dockerfile`, with the repository root as build context. A build stage
on the uv image runs `uv sync --frozen --no-dev`; the final stage is
`python:3.12-slim` with only the virtual environment copied. uv is absent from
the runtime image.

```yaml
  indexer:
    build:
      context: .
      dockerfile: indexer/Dockerfile
    container_name: daconnas-indexer
    restart: unless-stopped
    user: "${PUID}:${PGID}"
    environment:
      TZ: ${TZ}
      INDEXER_API_KEY: ${INDEXER_API_KEY}
    volumes:
      - "${APPDATA_DIR}/indexer:/config"
      - "${INDEXER_DEFINITIONS_DIR}:/definitions:ro"
    networks:
      - media
```

`user: "${PUID}:${PGID}"` mirrors Jellyfin and settles ownership of the database
file on the volume.

No `ports` entry. This departs from the other admin services on purpose: Torznab
has no interface to open, and its only client is Prowlarr on the same network at
`http://indexer:9117/api`. `print_urls` and `PORT_KEYS` are therefore unchanged.

### Configuration

`.env.example` gains two keys:

```dotenv
INDEXER_API_KEY=
INDEXER_DEFINITIONS_DIR=/home/your-user/daconnas-media/definitions
```

`INDEXER_DEFINITIONS_DIR` joins `DIRECTORY_KEYS`, inheriting directory creation
and absolute-path validation. `INDEXER_API_KEY` joins `REQUIRED_KEYS`.

### Migration

`validate_env` fails on a missing key, so adding `INDEXER_API_KEY` to
`REQUIRED_KEYS` makes an existing `.env` stop validating until it is edited.
This is intended, because an empty API key on a service that requires one is
worse. The error message must name the missing key and point at `.env.example`,
and `docs/installation.md` must document the step.

### Tests

Recorded HTML and JSON fixtures under `tests/fixtures/`, paired with test
definitions. `httpx.MockTransport` serves the fixtures, so no test touches the
network.

Coverage targets the parts that break: size parsing, date parsing, category
mapping, parent-category matching, base32 infohash normalisation, a row missing
its magnet, a page yielding zero rows, a source failing while others succeed,
and `caps` reflecting the loaded definitions.

The Torznab endpoint is tested through Flask's test client against a seeded
database, asserting on the parsed XML rather than on strings.

## Dependencies

Root `pyproject.toml` with `uv.lock` committed.

- Runtime, indexer only: `flask`, `httpx`, `beautifulsoup4`, `lxml`, `pyyaml`,
  `tinydb`.
- Development: `pytest`.

`httpx` earns its place over `urllib` through `MockTransport`, which is what
makes network-free tests straightforward.

`.venv/` is added to `.gitignore`. Pytest configuration lives in
`[tool.pytest.ini_options]`.

## Documentation changes

- `docs/indexer.md`, new: the definition format reference.
- `docs/services.md`: internal address table, and the step for adding the
  Generic Torznab indexer in Prowlarr.
- `docs/installation.md`: the two new keys.
- `README.md`: uv as a development requirement, `./media index` in the command
  list, the indexer in the service list.
