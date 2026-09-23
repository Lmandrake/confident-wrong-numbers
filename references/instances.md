# Instances: queries whose shape returned a confident wrong number

Each entry gives the **generalized shape** first, then one concrete case. The
cases come from a RimWorld modding project (paths and def names are examples,
nothing more); the shapes are not specific to it. Sections mirror the table in
`SKILL.md`.

Contents:
1. [A line count as an entry count](#1-a-line-count-as-an-entry-count)
2. [An existence test as an identity test](#2-an-existence-test-as-an-identity-test)
3. [A fixed index as a field](#3-a-fixed-index-as-a-field)
4. [A pattern wider or narrower than the thing](#4-a-pattern-wider-or-narrower-than-the-thing)
5. [A key that does not exist](#5-a-key-that-does-not-exist)
6. [An error swallowed as empty](#6-an-error-swallowed-as-empty)
7. [A record read as the thing it records](#7-a-record-read-as-the-thing-it-records)
8. [A source read as the effective state](#8-a-source-read-as-the-effective-state)
9. [A scope that drifted](#9-a-scope-that-drifted)
10. [A proxy set as the population](#10-a-proxy-set-as-the-population)
11. [A text scan that ignores grammar](#11-a-text-scan-that-ignores-grammar)
12. [An uncalibrated zero](#12-an-uncalibrated-zero)
13. [Shared-premise agreement](#13-shared-premise-agreement)

---

## 1. A line count as an entry count

**Shape.** `grep -c` and `wc -l` count *lines*. Any file that puts several
records on one line, or one record across several lines, makes the line count
and the record count unrelated. The output is an integer either way.

- **Config with many elements per line.** `grep -c '<li>' ModsConfig.xml`
  returned **48**; parsing the XML and counting `activeMods` children returned
  **631**. The file writes many `<li>` on one line. Caught only because the
  owner pushed back on a finding built on it.
- **Log with many lines per record.** `grep -c` on a game log counted a
  30-line stack trace as 30 errors. The honest figure was 245 distinct / 329
  occurrences, from a tool that understands the log's record boundaries.

**Honest instrument:** a parser for the format. For XML that is
`ET.parse(p).find("activeMods")` and `len(...)`; for a log it is whatever tool
knows where one error ends and the next begins.

## 2. An existence test as an identity test

**Shape.** "Does the path exist" and "is the thing there" are different
questions. A path can exist and be hollow; a path can be wrong and the shell
will answer `0` rather than error.

- **`ls <dir> | wc -l` on a directory that does not exist prints `0`.** An
  agent counted `artpipe/queue/` — which does not exist — and told the owner
  the job queue was empty. The real directory was `artpipe/pending/` and held
  **182** jobs. `ls` wrote its error to stderr; `wc` counted zero lines of
  stdout; the pipeline printed `0` and nothing else.
- **`[ -e src/RimMandrake/Pits ]` passed** while the folder held only
  `__pycache__`. The mod had been merged away weeks earlier; a validation
  checklist stayed "VALIDATED, 12 binding bars" against nothing.
- **A database file at the expected path was 0 bytes.** The real dump lived
  elsewhere. Every instrument pointed at the repo path read an empty database
  and reported everything absent.

**Honest instrument:** test the file that *identifies* the thing
(`dir/About/About.xml`, a manifest, a non-empty size), or glob a pattern that
fails loudly on a bad path (`ls dir/*.json` errors; `ls dir | wc -l` does not).
Before repeating a `0` from a listing, list the parent and confirm the child is
spelled as you think.

## 3. A fixed index as a field

**Shape.** Reading position N assumes every record keeps the field at N. The one
record that does not is the one that matters, and the query cannot tell you it
skipped it.

- **`subject:` read from line 2.** 77 walk files carry it on line 2;
  `AtmosphericBase.md` carries it on line 3. A sweep reading line 2 reported
  zero failures across 78 files while missing the only broken one.
- **`glob("*south*")[0]`.** Filesystem order decided whether the first hit was
  the art or the colour mask. See also §4.
- **An id read as an index.** A savegame grid stores a feature's *uniqueID*
  (21..92); a renderer read it as an index into the features list. Every region
  got a plausible tile count and every one was wrong. An id-vs-index confusion
  never errors; it renames everything.

**Honest instrument:** read by key. The first line *matching* `subject:`; the
entry whose id *equals* N; the column by its header name.

## 4. A pattern wider or narrower than the thing

**Shape.** The glob or regex has its own population. Where it is wider than
yours, it counts things you did not mean; where it is narrower, it misses forms
you did not think of. Both produce a clean integer.

- **Wider: `*south*` also matches `_southm.png`**, the colour mask, which is
  saturated across ~99% of its pixels by convention. A "does this head carry
  baked colour" check therefore depended on which file globbed first, and
  inverted 4 of 7 decisions. Tell: male and female variants of the same species
  reading 253 vs 0 on structurally identical files.
- **Wider: `alpha > 0` as a bounding box.** A sub-visible export halo (alpha
  1–16, under 6% opacity) extended a sprite's box 80 px past its painted body,
  producing two false "clean" verdicts. Threshold at a level a viewer can see.
- **Wider: bare names as "stale".** A scan for donor-mod names in a roster
  reported **182** stale rows; the real figure was **32**. The other 150 were
  bare Star Wars names that resolved fine against an *active* donor.
- **Narrower: a backtick-only regex.** A subject packageId was backticked in
  28 files, bare in 34, absent in 16. The regex read `None` for 50 of 78.

**Honest instrument:** print what the pattern matched on a sample and look at
it; run it against a known positive *and* a known negative before trusting
the count. Measure `_south.png` alone, not `*south*`.

## 5. A key that does not exist

**Shape.** A lookup on a key the object does not have returns the same nothing
for every record. The uniformity is the tell — a real population is rarely all
zero — but uniformity in the alarming direction reads as a catastrophe rather
than a bug.

- **`getattr(dict, "must_show")`.** `northstar.parse()` returns a dict, so
  `getattr` yielded `None`, `len()` yielded `0`, and all four VALIDATED
  checklists read as "0 bars". Use `w["must_show"]`.
- **`<def>` where the schema says `<defName>`.** Counting light fixtures in a
  ship layout export by `<def>` elements returned **0**; the real answer was 11
  of 2002.
- **`row["defName"]` where the API returns `def`.** A census over a live tool's
  output read every record as nameless.
- **`defName`/`defType` where the SQL columns are `def_name`/`def_type`.** See
  §6 for what happened to the error.
- **A tool leaf that does not exist, or a parameter named wrong.**
  `jawa/pawn_detail` (no such tool) and `pawnId` (the parameter is `pawn`)
  each produced "0 of N carry the trait".

**Honest instrument:** discover the vocabulary before counting — print one
record whole, read the schema, read the tool's own inputSchema. Treat an
all-`None`/all-`0` result as a key mismatch until a known positive comes back
non-zero.

## 6. An error swallowed as empty

**Shape.** The query fails, prints its failure on stdout or stderr, and the
next stage counts rows in that output. Zero rows. The failure has become a
measurement.

- **Wrong column name → error line → "0 of 109".** The SQL query errored; the
  script grepped its output for result rows, found none, and reported a
  confident zero.
- **A nonexistent tool → empty response → "0 of N".** (§5.) No error was
  surfaced to the sentence that carried the number.

**Honest instrument:** check the exit status and the error channel *before*
interpreting an empty result. In a script, an empty result on a query that
should return something is a failure, not a finding.

## 7. A record read as the thing it records

**Shape.** An export, snapshot, index or config describes the world at the
moment it was written. Reading it later as the world *now* is a query on the
wrong object; the numbers are exact and stale.

- **A tiles CSV exported from a savegame** was read for a live tile count. A
  bridge edit after the export date was invisible to it; the live system said
  222 tiles on one biome, the CSV still said 222 on the donor.
- **A plant pool CSV (an August snapshot)** was used to count dead-temperature
  flora: 46 claimed, 6 real against the current dump.
- **A config file's mtime** was read as evidence of what a running process had
  loaded. A restore had copied the backup back with metadata preserved, so live
  and backup were byte-identical and shared the pre-swap mtime. A crash on a
  19-mod tier was filed as a 618-mod production crash.
- **An index table graded `canon` on rows the entries themselves contradicted,**
  and mapped a sprite to the wrong entry entirely. When the entry and the index
  disagree, suspect the index.

**Honest instrument:** the live system, named as such in the report — the
running process's own "loaded with mods:" line, a live read through the bridge,
the entry rather than the index. A record is evidence about its own export
date; say which direction the evidence runs.

## 8. A source read as the effective state

**Shape.** The file you can read is one input to a pipeline. Patches, padding,
overrides and load order add to it. A source read is a lower bound, and the
asymmetry matters: presence in the source is safe to assert, absence is not.

- **A biome's `wildAnimals` list** read from the def alone showed 13 animals
  and "no big silhouette". A patch layer added 12 more, 7 of them large, and
  each animal's own `wildBiomes` field added load-time padding on top. The
  authoritative instrument reads a post-patch dump.
- **A `--check` that compares roster vs. a hardcoded dict** never looked at
  the hand-authored base XML the generator overwrites at load, so a def cut at
  the design layer sat live and stale while the check reported clean — four
  times in one session across independent agents.
- **A validator run without `--defs`** saw only the current mod's `Textures/`
  and reported "missing texture" for art a *later* mod supplies. Re-adding the
  "missing" art then silently reverted the other mod's verified-live art.

**Honest instrument:** the resolved view — the running process, a post-load
dump, the validator with its full input set. When only the source is
reachable, report the number as a lower bound and say so.

## 9. A scope that drifted

**Shape.** The instrument answers about a scope other than the one in your
sentence — a different map, a narrower directory, a default that was never
widened, a default that was widened on a guess.

- **A census tool read `Find.CurrentMap`**, so after any map switch or cull it
  silently reported the wrong map.
- **`ls artpipe/queue/`** instead of `artpipe/pending/` (§2) is a scope error
  as much as an existence one.
- **An audit's `marineChecked` scope defaulted to `['Coast']`;** widening it on
  a guess flagged 313 unrelated placements and an agent auto-removed 50.

**Honest instrument:** put the scope in the report — "on map X", "under
directory Y", "with scope Z". If a tool has a scope parameter, name the value
you used, and run it at its default before widening.

## 10. A proxy set as the population

**Shape.** A small set built for one purpose (a smoke test, a spot check) is
later used as an operand as though it were the whole population.

- **A build script's `GM_TOOLS`** was a 2-name check that the `#if` had fired
  at all. A selftest subtracted it as if it were the entire gated tool set, so
  a healthy 284/284 build read as "41 tools missing".

**Honest instrument:** count the population directly. A set built to answer
"did this happen at all" cannot answer "how many".

## 11. A text scan that ignores grammar

**Shape.** The consumer of a file has a grammar — it skips comments, joins
continuation lines, resolves inheritance. A scan that does not honour the same
grammar counts tokens the consumer never sees, or misses ones it assembles.

- **A naming lint counted defNames inside XML comments.** Five false positives
  survived a clean pass.
- **`[Tool(` and its name string sit on separate lines** in one codebase's
  style; a line-by-line scan of the gated region returned a false 0. Match the
  region as text.

**Honest instrument:** whatever the consumer does, do that — strip comments,
parse, or match across lines. If the consumer is a parser, use the parser.

## 12. An uncalibrated zero

**Shape.** The instrument cannot see the kind of thing you asked about, or was
pointed at a case that cannot produce a positive. Its zero is a statement about
its blind spot, not about the world.

- **"No fire started" on bare sand.** The fire instrument gates on
  flammability; sand cannot burn. Validate on a wood stack first.
- **A cell-info call returned empty `things[]`** on cells occupied by a
  building and items. The grav engine was invisible to it; a different tool
  with defName + rect was the existence instrument.
- **A texPath scan over `ThingDef` alone** reported "no art" for 79 of 84
  ported animals. Animal art hangs off `PawnKindDef.lifeStages`. The
  alarming-direction figure was believed for a while.

**Honest instrument:** before trusting a zero, run the same query on a case
you already know is non-zero. If it cannot find the known positive, its zero
means nothing.

## 13. Shared-premise agreement

**Shape.** Two measurements agree; both were built on the same wrong premise.
Agreement is corroboration only when the instruments differ in *shape*.

- Two passes counting `<li>` lines agree on 48. Two reads of the same CSV
  agree on 222. A re-run of the same `getattr` agrees on 0. None of these
  is a second measurement.

**Honest instrument:** a cross-measure changes the shape — parse instead of
scan, live instead of record, one record printed whole instead of a field
extracted from all. If you cannot name what premise the second instrument
does *not* share with the first, it is the same instrument.
