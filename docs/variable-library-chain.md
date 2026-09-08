---
artifact:
  url: https://claude.ai/code/artifact/b7230f16-43dd-4fb0-ba91-53e4b16f1c85
  favicon: "🧬"
  description: >-
    A left-to-right schematic of how dbGaP digests and LinkML-Map transformation specs
    converge into BDC variable library entries, with the script behind each stage annotated.
  build: /private/tmp/claude-501/-Users-ginniehench-Developer-dm-bip/08c4da51-3e4a-4d9e-89aa-6896662a6680/scratchpad/variable-library-chain.html
palette:
  spec:   { light: "#0f6a60", dark: "#52bfb1" }   # the transformation-spec stream
  dbgap:  { light: "#8f5c0c", dark: "#d9a44e" }   # the dbGaP digest stream
  typing: { light: "#74406b", dark: "#c489b6" }   # the inferred-schema stream
  held:   { light: "#8d5a52", dark: "#c08d84" }   # variables not emitted
type:
  display: IBM Plex Sans Condensed
  body:    Source Serif 4
  mono:    IBM Plex Mono
---

<!--
  SOURCE FILE. The published artifact is a build output, not the original.

  To change the page: edit this file, then ask Claude to rebuild the chain artifact.
  It recompiles to the HTML at `artifact.build` and republishes to `artifact.url`,
  so the link stays the same.

  How the compiler reads this file:
    H1                          -> page title
    the *italic* line under it  -> eyebrow
    first paragraph             -> standfirst
    the `code · code` line      -> the run line under the standfirst
    ```chain fence              -> the SVG schematic (see the block's own notes)
    *italic* para after it      -> figcaption
    "### NN · Title" + comment  -> a numbered stage row; its trailing bullet list of
                                   backticked paths becomes the right-hand script column
    everything else             -> ordinary sections, in document order

  Module paths appear twice on purpose: once short, inside a diagram box, and once in
  full in the stage list. Change both, or say which one is right and I will reconcile.
-->

# Variable Library Chain

*dm-bip · variable library extractor*

Two provenance streams converge on one file. The transformation specs say *which* source
variables exist and what they feed; the dbGaP digests say what those variables *mean*.
Everything below happens in a single process — nothing is written between stages except the
digest cache.

`driver: dm-bip extract-variable-library` · `make: variable-library` · `out: $(DM_OUTPUT_DIR)/variable-library.yaml`

## The chain

```chain
# Diagram source. Geometry is derived, not written: the compiler places a node from its
# (lane, row) cell and routes edges between cell edges. Reorder lanes, move a node to a
# different row, retitle a box, or add an edge — nothing here is a coordinate.
#
#   kind:    source | stage | output | held   (controls the box treatment)
#   row:     names the stream, and therefore the colour, from `palette` in the front matter
#   body:    the short text inside the box; keep lines under ~26 characters
#   scripts: the mono annotation at the foot of the box, at most two lines
#   example: an optional band under a hairline inside the box, carrying the concrete
#            arguments one real run passes in. Only lanes marked `width: wide` have room
#            for it. Two lines, ~43 characters each.
#
# Edge fields: from, to (node id, optionally `id:top|bottom|left|right`), label, note
# (a second, quieter line), style (solid | dashed | emphasis), stream (a palette key or
# `neutral`), offset (nudges the entry point along the shared edge, in diagram units).

# `width: wide` gives a lane room for an `example` band; the rest stay at the base width.
lanes:
  - { name: Inputs,  width: wide }
  - { name: Read,    width: wide }
  - { name: Resolve }
  - { name: Express }
  - { name: Output }

rows: [spec, dbgap, typing]

nodes:
  - id: specs
    kind: source
    lane: Inputs
    row: spec
    eyebrow: SOURCE A
    title: Transformation specs
    body: |
      LinkML-Map YAML in the study repo,
      plus its researchstudy.yaml
    scripts: [DM_TRANS_SPEC_DIR]
    example:
      label: ARIC RUN
      lines:
        - ~/Developer/NHLBI-BDC-DMC-HV/
        - priority_variables_transform/ARIC-ingest

  - id: digests
    kind: source
    lane: Inputs
    row: dbgap
    eyebrow: SOURCE B
    title: dbGaP digest XML
    body: |
      data_dict declares the column;
      var_report measures it
    scripts: [ftp.ncbi.nlm.nih.gov/dbgap/studies]
    example:
      label: ARIC RUN
      lines:
        - --dbgap-cache /tmp/dbgap-live-check
        - already populated, so nothing is downloaded

  - id: schema
    kind: source
    lane: Inputs
    row: typing
    eyebrow: SOURCE C
    title: Inferred schema
    body: |
      schema-automator over the prepared
      TSVs — a pipeline product
    scripts: [make schema-create]
    example:
      label: ARIC RUN
      lines:
        - -s /tmp/aric-mini-schema.yaml
        - hand-cut, types five variables

  - id: extract
    kind: stage
    stage: 1
    lane: Read
    row: spec
    title: Index the variables
    body: |
      One record per phv, holding every
      target slot it feeds
    scripts: [variable_lib/extract.py, collect_variables()]
    example:
      label: ARIC RUN
      lines:
        - every *.yaml under …/ARIC-ingest,
        - searched recursively

  - id: fetch
    kind: stage
    stage: 2
    lane: Read
    row: dbgap
    title: Pin, filter, fetch
    body: |
      Match the study to a cohort, then
      pull only the named datasets
    scripts: [prepare_study/fetch_digests.py]
    example:
      label: ARIC RUN
      lines:
        - --cohort aric --no-fetch
        - still resolves and selects by pht

  - id: read
    kind: stage
    stage: 3
    lane: Resolve
    row: dbgap
    title: Read the digests
    body: |
      Merge what is declared
      with what is observed,
      one index per table
    scripts: [variable_lib/dbgap.py, load_tables()]

  - id: classify
    kind: stage
    stage: 4
    lane: Resolve
    row: typing
    title: Type each variable
    body: |
      Continuous or
      categorical, from the
      declared range alone
    scripts: [variable_lib/classify.py, classifier_for()]

  - id: map
    kind: stage
    stage: 5
    lane: Express
    row: dbgap
    title: Map onto BDC slots
    body: |
      dbGaP fields become the
      slots the chosen class
      actually accepts
    scripts: [dbgap_metadata.py, DbgapMetadata.lookup()]

  - id: emit
    kind: stage
    stage: 6
    lane: Express
    row: spec
    title: Emit the entries
    body: |
      Identity from the specs,
      description from dbGaP,
      grouped by class
    scripts: [variable_lib/emit.py, "to_entries() · to_yaml()"]

  - id: output
    kind: output
    lane: Output
    row: spec
    eyebrow: DELIVERABLE
    title: variable-library.yaml
    body: |
      Two keyed lists, always
      both present, unset
      slots omitted
    scripts: [single_continuous_…, single_categorical_…]

  - id: held
    kind: held
    lane: Output
    row: dbgap
    eyebrow: HELD BACK
    title: Untyped variables
    body: |
      Neither class defines a
      home for an untyped
      identity, so it is
      counted, not guessed
    scripts: [VariableEntries, .unclassified]

edges:
  - { from: specs,   to: extract,      stream: spec }
  - { from: digests, to: fetch,        stream: dbgap,  label: listing }
  - { from: schema,  to: classify,     stream: typing, style: dashed,
      label: "declared ranges, keyed (pht, phv)" }

  # The load-bearing edge: what stage 1 found is what stage 2 is allowed to download.
  - { from: extract:bottom, to: fetch:top, stream: spec, style: emphasis,
      label: "the pht set", note: "only these are fetched" }

  - { from: fetch,   to: read,         stream: dbgap,  label: cache }
  - { from: read,    to: map,          stream: dbgap,  label: index }

  # The classifier is built once in stage 4 and consulted twice.
  - { from: classify, to: map:bottom,  stream: typing, style: dashed, label: kind }
  - { from: classify, to: emit:left,   stream: typing, style: dashed, label: kind }

  - { from: extract, to: emit,         stream: spec,   offset: -20,
      label: "identity triple + every usage" }
  - { from: map:top, to: emit:bottom,  stream: dbgap,  label: "slot values" }

  - { from: emit,       to: output,    stream: neutral }
  - { from: emit:right, to: held:left, stream: held,   style: dashed, label: untyped }

footnote: >-
  One process. The only thing written between stages is the dbGaP cache, which the command
  manages itself. The ARIC RUN lines are the concrete arguments from the cached check run.
```

*The two crossing edges are the ones worth reading. Stage 1's set of `pht` accessions becomes
the filter on stage 2's download, so a cohort's full published listing is never fetched; and
the classifier built in stage 4 is consulted twice — once to pick the entry class, once to
decide which slot set stage 5 is even allowed to return.*

## The six stages

In general terms, with the module and entry point that carries each one.

### 01 · Index the source variables
<!-- stream: spec -->

Walk every derivation block in the specs, including nested class derivations, and re-key what
they yield by `phv` accession. A mapping-provenance record is organized around the derived
thing; a variable library record is organized around the source thing, so each entry
accumulates *all* of its uses rather than keeping the first one seen. A slot populated straight
across is kept distinct from one referenced inside an expression.

- `variable_lib/extract.py` — `collect_variables()`
- `mapping_prov/extract.py` — `iter_spec_blocks()`, `read_study()`

### 02 · Pin the cohort, then fetch only what was named
<!-- stream: dbgap -->

The study accession found in the specs is matched against the upstream cohort manifests, which
pin the dbGaP version. The FTP directory listing is scraped, and because the `pht` accession
sits in every digest filename, the listing is filtered before anything is downloaded — at no
extra request cost.

- `prepare_study/fetch_digests.py` — `load_cohorts()`, `cohort_for_study()`
- `fetch_digests(datasets=…, kinds=…)`
- `cli.py` — `_dbgap_metadata()`

### 03 · Read the digests
<!-- stream: dbgap -->

Parse both XML kinds into one index per table, keyed on the `pht` the file declares rather than
the one in its name — a cohort's listing includes tables contributed by other studies. Only the
total-set row of a var_report contributes; the per-consent-group rows restate it. Parsing is
hardened: no entity resolution, no DTD, no network.

- `variable_lib/dbgap.py` — `read_data_dict()`
- `merge_var_report()`, `load_tables()`

### 04 · Type each variable
<!-- stream: typing -->

The BDC model splits single variables into a continuous class and a categorical one, and a
transformation spec never says which a variable is. The signal comes from the inferred schema,
where classes are named by `pht` and slots by `phv` — so the pair is an exact lookup, not a name
match. The rule reads the declared range and nothing else; no distinct-value threshold is
applied, because a threshold is a judgement rather than a fact.

- `variable_lib/classify.py` — `classifier_for()`
- `classify_from_source_schema()`

### 05 · Map dbGaP fields onto BDC slots
<!-- stream: dbgap -->

The only layer that knows target slot names. It is handed the same classifier stage 4 built, so
it returns exactly the slot set the chosen class accepts — bounds and unit for a continuous
variable, coded values for a categorical one. Returning the union instead would work, and would
emit one warning per dropped key per variable: thousands of lines on a real study.

- `variable_lib/dbgap_metadata.py` — `metadata_for()`
- `DbgapMetadata.lookup()`

### 06 · Emit the entries
<!-- stream: spec -->

Identity fields are laid down first and the metadata is merged over them — never the other way,
or a table contributed by another study would overwrite the study the spec belongs to. Entries
are grouped by the class they took, both keys always present even when empty, unset slots
omitted rather than written as nulls. Repeated runs over unchanged inputs are byte-identical.

- `variable_lib/emit.py` — `to_entries()`, `to_yaml()`
- `variable_lib/datamodel/` — gen-pydantic classes

## What fills each slot

The split is the whole reason there are two input streams: the specs can only ever supply the
join.

### From the transformation specs
<!-- stream: spec -->

| Slot | Value |
|---|---|
| `id` | `dbgap:phv10111300` |
| `source_id` | the `phv` accession |
| `file_id` | the `pht` it was seen under |
| `associated_study` | the `phs` — a placeholder until study identity is settled |
| `variable_description` | every target slot this variable feeds, rendered |

### From the dbGaP digests
<!-- stream: dbgap -->

| Slot | Value |
|---|---|
| `variable_name` | VARNAME |
| `source_variable_description` | VARDESC |
| `file_name` | the table name, which only var_report carries |
| `data_type` | calculated where available, declared otherwise |
| `comment` | as published |
| `minimum_value`, `maximum_value`, `unit` | continuous only; unit normalized to UCUM, unmapped units passed through unchanged |
| `coded_values` | categorical only, in document order — dbGaP orders values meaningfully |
| `missing_value` | a coded value on a numeric variable is a sentinel, not a domain |
| `resolution`, `alert_values` | always empty; dbGaP states neither, and deriving them would be inference |

## Why both digest files

They are not interchangeable, and the disagreement between them is the point. ARIC declares a
standing height in centimetres as `<type>string</type>`.

| Across 1,298 ARIC spec variables | data_dict.xml — declared | var_report.xml — observed |
|---|---|---|
| Bounds | `<logical_min>` present on **0** | `<stat min max>` present on **100%** of numeric variables |
| Type signal | free text, four spellings, all reading as string or encoded | a closed vocabulary: `integer`, `decimal`, `enum_integer`, `string` |

That is why `--no-var-report` is not the default. It halves the download and produces entries
with no bounds and a type derived from the declared one — which for ARIC-shaped studies means
`string` on real measurements.

<!-- facts -->

- **326 / 736** — ARIC digest files fetched, because the specs name 164 datasets
- **164** — datasets referenced, one of which dbGaP never published a dictionary for
- **~3 min** — a cold ARIC run, at the half-second courtesy delay between NCBI requests

## Running it

The `-s` schema is a pipeline product, so it has to be built first — but it is the *only*
prerequisite. The variable library never touches validation output or mapped data.

```sh
# build the inferred schema the typing step needs
make schema-create   CONFIG=path/to/study.mk

# then, either through the pipeline …
make variable-library CONFIG=path/to/study.mk DM_COHORT=aric

# … or directly, which always regenerates — the better loop while iterating
dm-bip extract-variable-library path/to/specs/<study> \
  -s path/to/output/<study>/<DM_SCHEMA_NAME>.yaml \
  --cohort aric \
  -o variable-library.yaml
```

`--cohort` may be omitted: the command reads the study accession from the specs and reports
which cohort it picked. A study with no dbGaP presence is not an error — it says so and emits
entries carrying identity only. `--no-fetch` is a genuine offline path against whatever is
already cached.

### Checking both halves

No single corpus supplies both inputs, so the check is two runs. The synthetic corpus has no
`phs` accession, which exercises the identity-only path; the ARIC run — whose arguments are the
**ARIC RUN** lines in the schematic above — exercises the dbGaP path against a cache, with no
network.

```sh
#!/usr/bin/env bash
# Test the variable library pipeline, both halves.
set -u
cd ~/Developer/dm-bip

SYNTH=~/Developer/study-palette/synthetic

echo "1. Classification half (synthetic corpus, fully offline)"
echo "   expect: 35 entries (19 continuous, 16 categorical)"
rm -f "$SYNTH/output/study_one/variable-library.yaml"
make variable-library CONFIG="$SYNTH/pipeline/example_study_one.mk" \
                      SYNTH_DIR="$SYNTH" SYNTH_OUTPUT_DIR="$SYNTH/output/study_one"

echo "2. dbGaP half (real ARIC dictionaries, cached, no network)"
echo "   expect: 5 entries (4 continuous, 1 categorical)"
uv run dm-bip extract-variable-library \
  ~/Developer/NHLBI-BDC-DMC-HV/priority_variables_transform/ARIC-ingest \
  -s /tmp/aric-mini-schema.yaml \
  --cohort aric --dbgap-cache /tmp/dbgap-live-check --no-fetch \
  -o /tmp/vl.yaml

echo "--- first entry: last 4 slots come from dbGaP ---"
head -20 /tmp/vl.yaml
```

Half 1 was last observed on 2026-09-04 reporting `35 entries from 35 source variables
(19 continuous, 16 categorical)`. Half 2 depends on two fixtures that are not checked in —
`/tmp/aric-mini-schema.yaml` and a populated `/tmp/dbgap-live-check` — so it only reproduces
where they still exist. Drop `--no-fetch` to rebuild the cache from dbGaP instead.

<!-- footer -->

dm-bip · variable library extractor · [docs/variable-library.md](variable-library.md) ·
tis-lab/BDC-Add-On-Tracker#93 · linkml/dm-bip#352
