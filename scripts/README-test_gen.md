# ConceptBook Generation Guide (cb-calculus)

This repo generates concept books for **one domain family**: OpenStax
*Calculus Volume 1*, ingested chapter-by-chapter via the `concept-book-press`
Path-B pipeline. Domain IDs in `public/domains/catalog.json`:

```
calculus_ch01   Functions and Graphs             (3 application nodes)
calculus_ch02   Limits                           (4 application nodes)
calculus_ch03   Derivatives                      (4 application nodes)
calculus_ch04   Applications of Derivatives      (6 application nodes)
calculus_ch05   Integration                      (2 application nodes)
calculus_ch06   Applications of Integration      (4 application nodes)
```

All 6 are `VALID` (see `concept-book-press/docs/projects/README.md` for the
ingestion notes) and currently have `has_book: false` — no concept-book HTML
generated yet for any of them. There is no merged `calculus_full` domain
(unlike `linalg_full` in `cb-linalg`) — merge is a separate step to do later
if wanted.

## Prerequisites

```bash
conda activate spl123
cd ~/projects/digital-duck/cb-calculus/
```

The backend API must be running for the UI Generate/PDF buttons:
```bash
# terminal 1
bash scripts/start-api.sh   # uvicorn on :8200

# terminal 2
npm install
npm run dev                 # Vite on :5174 (separate terminal)
```

---

## Scripts

`scripts/test_gen.sh` was removed (it hardcoded a fixed domain list from the
template it was forked from) — run `scripts/batch_generate.py` directly
instead, as shown below.

### `scripts/batch_generate.py` — CLI with full options

```bash
python scripts/batch_generate.py [OPTIONS]

# example
python scripts/batch_generate.py --level college --language en
python scripts/batch_generate.py --level research --language en
```

| Option | Default | Description |
|--------|---------|-------------|
| `--domain` | all | Domain ID (repeatable: `--domain calculus_ch01 --domain calculus_ch02`) |
| `--n-targets` | 2 | Number of application nodes per domain |
| `--level` | domain default (`college`) | Override level: `intro / core / college / research` |
| `--language` | `en` | Output language ISO code (`en`, `zh`, `fr`, …) — friendly names also accepted (`chinese`, `french`) |
| `--llm` | `claude_cli:claude-sonnet-4-6` | LLM backend (env: `CB_LLM`) |
| `--spl-dir` | `~/projects/digital-duck/SPL.py` | SPL.py root (env: `CB_SPL_DIR`) |
| `--skip-cache` | off | Bypass spl3 LLM cache — force fresh generation |
| `--skip-existing` / `--no-skip-existing` | **on** | Skip a target already in `catalog.json` for this exact `(target, model, language)` combination |
| `--dry-run` | off | Print planned jobs without running |
| `--stop-on-error` | off | Abort batch on first failure |

**`--skip-existing` is language-aware** (ported from `cb-linalg` 2026-07-26
fix): it keys the "already generated" check on `(target, model, language)`,
not just `(target, model)` — so `--language zh` on a domain already generated
in English will not be falsely skipped.

**Recommended first run — one domain, dry-run first:**
```bash
python scripts/batch_generate.py --domain calculus_ch01 --dry-run
python scripts/batch_generate.py --domain calculus_ch01 --skip-cache
```

**Then the rest of the chapters:**
```bash
python scripts/batch_generate.py \
    --domain calculus_ch01 --domain calculus_ch02 --domain calculus_ch03 \
    --domain calculus_ch04 --domain calculus_ch05 --domain calculus_ch06 \
    --skip-cache
```

`--n-targets` defaults to 2 application nodes per domain — `calculus_ch04`
has 6 available if you want more than the default 2 for that chapter
(`--domain calculus_ch04 --n-targets 6`).

### `scripts/batch_gen_domains.py` — file-driven batch runs, one book per domain

For an unattended run across a domain list instead of repeated `--domain`
flags. Picks each domain's capstone target automatically (first application
node), is resumable (progress file tracks what's `done`), and stops the
whole batch the moment it sees a Claude CLI session/rate-limit signature
rather than burning through the rest of the list.

```bash
python scripts/batch_gen_domains.py -f scripts/domains-calculus.txt [OPTIONS]
```

You'll need to create `scripts/domains-calculus.txt` first — one domain id
per line, `#` starts a comment:
```
calculus_ch01
calculus_ch02
calculus_ch03
calculus_ch04
calculus_ch05
calculus_ch06
```

| Option | Default | Description |
|--------|---------|-------------|
| `--domains-file` / `-f` | — | Required. Path to the domain list `.txt` |
| `--model` | `sonnet` | Shorthand (`sonnet`/`haiku`/`opus`/`gemma3`/`gemma4`) or a raw spl3 `--llm` string |
| `--level` | `college` | `intro / core / college / research` |
| `--language` / `-l` | `en` | ISO code or friendly name |
| `--skip-cache` | off | Bypass spl3 LLM cache |
| `--force` | off | Regenerate even if the book already exists on disk |
| `--limit` | all | Only process the first N domains — use for a test run before a full unattended pass |
| `--progress-file` | `scripts/batch_gen_domains_progress.json` | Resume tracking, keyed `domain\|model\|level\|lang` |
| `--log-file` | none | Also write output to this file (in addition to stdout) |

```bash
conda activate spl123

# Test — 1 domain, current defaults (sonnet / college / en)
python scripts/batch_gen_domains.py -f scripts/domains-calculus.txt --limit 1

# Full run (resumable — re-running skips anything already marked done)
python scripts/batch_gen_domains.py -f scripts/domains-calculus.txt \
    --log-file scripts/batch_gen_domains.log
```

Expect several minutes per domain with Sonnet (a dozen+ LLM calls each,
scaling with node count — `calculus_ch04` at 35 nodes will take longer than
`calculus_ch05` at 27). If the Claude CLI session/rate limit is hit
mid-batch, just re-run the same command once it resets — already-`done`
domains are skipped.

### `scripts/sync_from_press.py` — pull graphs from concept-book-press

Regenerates a chapter's `input/graph.yaml` from the upstream
`concept-book-press` extraction pipeline (OpenStax Calculus Volume 1 source).
Run this if the source chapter's graph is re-extracted/edited:
```bash
python scripts/sync_from_press.py --book calculus-volume-1 --chapters 1-6 --prefix calculus_ch --subject Calculus
python scripts/sync_from_press.py --book calculus-volume-1 --chapters 4 --prefix calculus_ch --subject Calculus
```
`--subject` controls the catalog `name`/`description` label (added 2026-07-25 —
earlier revisions of this script had a hardcoded "Physics" label left over
from the template it was forked from; always pass `--subject Calculus` here).

---

## Cache behaviour

The spl3 content cache key is `(concept, language, llm)`.

- Same concept in **different languages** → separate cache entries (independent)
- Same concept with **different LLM** → separate cache entries (good for quality comparison)
- Re-running without `--skip-cache` reuses the cached version → fast (0 LLM calls)
- Re-running with `--skip-cache` regenerates everything fresh → slow but picks up prompt changes

**Rule of thumb:**
- First run for a new domain/language/model → always add `--skip-cache`
- Subsequent runs to fill missing concepts → omit `--skip-cache` (reuse hits, generate misses)

---

## Generating in other languages

Once the English pass exists, add a language on top (does not touch/remove English):
```bash
python scripts/batch_generate.py --domain calculus_ch01 --language zh --skip-cache --dry-run
python scripts/batch_generate.py --domain calculus_ch01 --language zh --skip-cache
```
Output lands in a separate directory per language, alongside the English:
```
public/domains/calculus_ch01/output/college.en/sonnet/html/
public/domains/calculus_ch01/output/college.zh/sonnet/html/
```

---

## Comparing LLM quality

```bash
# Generate with Sonnet (default)
python scripts/batch_generate.py --domain calculus_ch01 --skip-cache

# Generate same domain with Haiku for comparison
python scripts/batch_generate.py --domain calculus_ch01 --skip-cache \
    --llm claude_cli:claude-haiku-4-5-20251001
```

Both outputs are cached independently (keyed by `llm`, not overwritten). Compare the HTML files in:
```
public/domains/calculus_ch01/output/college.en/sonnet/html/
public/domains/calculus_ch01/output/college.en/haiku/html/
```

---

## Output locations

| Artifact | Path |
|----------|------|
| Concept book HTML (TOC index) | `public/domains/{id}/output/{level}.{lang}/{model}/html/book_{target}.html` |
| Individual concept HTML | `public/domains/{id}/output/{level}.{lang}/{model}/html/concept_{name}.html` |
| PDF | `public/domains/{id}/output/{level}.{lang}/{model}/pdf/book_{target}.pdf` |
| Concept graph | `public/domains/{id}/output/graph.html` |
| Generation logs | `logs/batch_gen_YYYYMMDD_HHMMSS.log` |
| SPL run logs | `~/.spl/logs/build_concept_book-*.md` |

---

## Regenerating graph.html (after color/structure changes)

```bash
bash scripts/sync_from_spl.sh
```

This copies `*_graph.yaml` from SPL.py and regenerates all `graph.html` files.
Then hard-refresh the browser (`Ctrl+Shift+R`).

---

## Completed runs

### calculus_ch01-06 ingested + synced from concept-book-press (2026-07-25)
```bash
cd ~/projects/digital-duck/concept-book-press
python -B -m pipeline.cli list-chapters --pdf input/calculus-volume-1.pdf
# ingest --chapter N / extract / validate per chapter, then:
python scripts/sync_from_press.py --book calculus-volume-1 --chapters 1-6 --prefix calculus_ch --subject Calculus
```
All 6 chapters `VALID` (one real error fixed in ch4 — `iterative_process` had
no prerequisite path, moved from a `concepts` entry with `composed_of: []`
to a tier-0 `primitives` entry). Registered in catalog with `has_book: false`
— no concept-book HTML generated yet for any chapter. This is the starting
point for the first `batch_generate.py` run.
