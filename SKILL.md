---
name: confident-wrong-numbers
description: Catch a query whose SHAPE returns a confident wrong number — a count, a zero, an empty list, an all-None field — before you report it. Use whenever you are about to state a number derived from grep, wc, ls, a glob, a regex, getattr, a dict key, a column or tag name, an API call or a census script; whenever a result is 0, empty, all-None, suspiciously round or alarming; and BEFORE writing any census, sweep, audit or checker script. Not oversized files (measuring-large-artifacts) or stale written claims (verify-before-you-escalate) — the query itself is the bug.
---

# Confident wrong numbers

A query can be built so that its **shape** guarantees a plausible answer to a
question you did not ask. The instrument does not error, warn or return null; it
returns an integer, and the integer looks exactly like a right one. The wrongness
lives in the construction of the query, and nothing in the output shows it.

This is not a big-file problem, an encoding problem or a stale-doc problem. It
is a five-second `wc -l` on a path that does not exist, a `getattr` on a dict, a
glob that also matches the mask, a header read from line 2 when one file keeps
it on line 3. Each is a tiny query; each produced a number that was reported,
believed and acted on. The catalogue is in `references/instances.md`.

## The tells: shapes of query that lie

Every entry below returned a real number for a real question — just not the
question the author had in mind. Learn the shapes, not the incidents; the next
one will wear a different filename.

| the shape you wrote | what it actually measures | the honest instrument |
|---|---|---|
| **A line count as an entry count.** `grep -c`, `wc -l` on structured text | lines *containing* the pattern — one element per line is an assumption, and a stack trace is 30 of them | parse the format (XML, JSON, the log's own record boundaries) and count records |
| **An existence test as an identity test.** `[ -e dir ]`, `ls dir \| wc -l`, "the file is there" | that a *path* resolves — a folder of `__pycache__` passes, a misspelled path yields `0` with no error | test the file that *identifies* the thing (`dir/About.xml`, the key inside it); glob a pattern that errors on a bad path; list the parent |
| **A fixed index as a field.** line 2, `hits[0]`, `cols[3]`, an id read as a position | whatever sits at that position *today* — silently the wrong thing when one file differs or filesystem order changes | read by key: the first line *matching* `subject:`, the entry whose id equals N, the column by its header |
| **A pattern wider or narrower than the thing.** `*south*` also hits `_southm`; a backtick-only regex misses the bare form; `alpha > 0` admits a halo; bare names resolve against something else too | the pattern's population, which overlaps yours | enumerate what the pattern matched on a sample; check a known positive AND a known negative before trusting the count |
| **A key that does not exist.** `getattr(dict, "x")`, `row["defName"]` where the field is `def`, `<def>` where the schema says `<defName>`, `defName` where the column is `def_name` | nothing — every lookup returns `None`, `0` or an error, uniformly, for every record | discover the vocabulary first (one record printed whole, the schema, the tool's inputSchema); an all-None or all-zero result is a key mismatch until shown otherwise |
| **An error swallowed as empty.** a query whose failure prints a line your script then greps for rows; a tool leaf that does not exist and returns nothing | that the query *failed* — "0 of 109" is the shape of an exception | check the exit status / error channel before reading a result as empty |
| **A record read as the thing it records.** an exported CSV, a snapshot pool, an index table, a config file, a backup byte-identical to live | the world at the record's export time, which may be days old and edited since | the live system, stated as such; the record is evidence only about its own date |
| **A source read as the effective state.** the def alone, the roster, the base XML | a *lower bound* — patches, padding, overrides and load order add to it | the post-load view (dump, running process, resolved object); presence in the source is safe to assert, absence is not |
| **A scope that drifted.** the current map after a switch; a tool run without the flag that widens its view; a checker comparing two derived files and never the artifact | a different population than the one in your sentence | state the scope in the report; if the tool has a scope parameter, name the value you used |
| **A proxy set as the population.** a 2-name spot check subtracted as if it were the whole gated set | the proxy | count the population directly; a spot check is a smoke test, never an operand |
| **A text scan that ignores grammar.** defNames inside XML comments; `[Tool(` and its name on separate lines scanned line by line | tokens the parser would discard, or would join | strip comments / match the region as text / parse; whatever the consumer does, do that |
| **An uncalibrated zero.** "no fire started" on ground that cannot burn; "not found" from a tool that cannot see that kind | the instrument's blind spot | run it on a case where the answer is known to be non-zero first |

Two asymmetries fall out of the table and are worth carrying separately:

- **Presence is cheaper to trust than absence.** A hit proves the thing exists
  in *some* form; a miss proves only that your query did not find it.
- **Agreement is not corroboration when the instruments share a premise.** Two
  passes that both count lines, or both read the same export, will agree on the
  same wrong number. Independence means a different *shape*, not a second run.

## Before you report a number

Run this on every number that is about to leave your hands — into a reply, a
doc, an item, a decision. It takes seconds; the wrong number costs hours.

1. **Name the instrument.** Not "I checked" — the exact command, key, pattern or
   tool call. If you cannot write it down, you cannot be sure what it measured.

2. **Name the other world that returns this same number.** Ask: *if the thing I
   care about were completely different, what would still make this query say
   exactly this?* A missing directory says `0`. A wrong key says `None`
   everywhere. An error line says "no rows". A mask says "saturated". If such a
   world exists and you have not ruled it out, the number is a hypothesis.

3. **Cross-measure when it is round, zero, empty, uniform or alarming.** Use an
   instrument with a different shape — a parser instead of a scan, a listing of
   the parent instead of the child, the live system instead of the record, one
   record printed whole instead of a field extracted from all of them. A second
   run of the same query is not a cross-measure.

4. **Calibrate before you trust a zero.** Point the instrument at a case whose
   answer you already know to be non-zero. If it cannot find the known
   positive, its zero on the unknown means nothing.

5. **Report it as MEASURED or UNMEASURED, with scope.** "MEASURED 631 active
   mods, parsed from `ModsConfig.xml` `activeMods`, on this machine, now." Or
   "UNMEASURED — the instrument cannot see this from here." Never a bare
   integer, and never `0` standing in for "I could not tell."

The heuristic that compresses all five: **a count that is conveniently or
alarmingly round is a query bug until proven otherwise.**

## The alarming direction

You will check a convenient number, because it looks like something you wanted
to be true. You will *not* check an alarming one, because alarm feels like
rigour — "all four checklists have 0 bars", "79 of 84 rows have no art", "182
stale entries", "0 of N carry the trait". Each of those was a query bug, and
each was believed and repeated precisely because it was bad news. Treat the
alarming direction as the *more* suspect one: it is the one you will not
instinctively re-run, and it is the one that travels fastest once said.

## When a hook already enforces it

Some projects refuse certain query shapes mechanically — for example a
`PreToolUse` hook that blocks `grep`/`wc`/`strings` on files whose encoding a
byte scanner cannot read, and prints the instrument to use instead. Read the
refusal as a *worked example of this skill*, not as an obstacle: the hook exists
because that exact shape returned `2` where the answer was `233`, and `48`
where it was `631`, and nobody could see the difference in the output.

A hook can only match the patterns its author already met. The class is wider
than any pattern list — `getattr` on a dict and `ls` on a guessed path will
never trip a scan-blocker. So when a hook stops you, do not route around it
with a synonym tool; ask which row of the table above it is protecting you
from, and carry that row to the next query the hook cannot see.

## Neighbours

- **`measuring-large-artifacts`** — the artifact is too big to read, so you
  reached for a scanner that does not understand its encoding. This skill is the
  case where the artifact is small and the query itself is the bug.
- **`verify-before-you-escalate`** — someone *wrote* a claim and it decayed.
  This skill is about a claim you are about to generate yourself.
- **`calibrating-binary-formats`** — the bytes are opaque and you are guessing
  the encoding. Here the text is perfectly readable and you asked it the wrong
  question.

Catalogue of instances, generalized shape first and a concrete example second:
`references/instances.md`.
