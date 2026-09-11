# SPEC.md — mytools

A small reimplementation of a subset of bedtools. Real `bedtools` (v2.31.1, installed on
this VM) is the correctness oracle: where this spec and bedtools disagree, bedtools wins
unless the disagreement is listed in §8 as a deliberate deviation.

---

## 1. Scope

**Subcommands in v1:** `sort`, `merge`, `intersect`, `subtract`, `closest`.

**Explicitly NOT in v1:**

- Strand-aware behaviour anywhere (`-s`, `-S`). *Rationale: doubles the test matrix for
  every subcommand and none of the planned exercises need it.*
- `-header` handling. *Rationale: not in the v1 flag set, so there is nothing to print
  headers for.*
- Any other bedtools subcommand (`getfasta`, `slop`, `complement`, `bamtobed`, …).
- BAM input. *Rationale: `bedtools bamtobed` converts to BED first; mytools only ever
  sees text.*

## 2. Input formats

- **Formats accepted:** BED3 through BED6. Columns beyond 6 are carried through
  untouched rather than parsed. *Measured: `sort` and `intersect` both preserve a 7th
  and 8th column verbatim; `merge` discards everything past BED3 (see §4).*
- **Source:** a file argument or stdin, both supported.
- **`-` means stdin.** Matches bedtools.
- **Compressed input:** **not supported in v1.** Plain text only. *Rationale: extra code
  in every input path, and no fixture in `data/` is compressed.*
- **`track` / `browser` / `#` lines:** skipped silently, not passed through and not an
  error. Matches bedtools.

## 3. Interval semantics

- **Coordinate system: 0-based, half-open.** `chr1 100 200` covers bases 100–199. This
  is not a decision; BED says so.
- **Overlap predicate:** two intervals overlap iff

  ```
  a.start < b.end  AND  b.start < a.end
  ```

  Note the strict `<` on both sides. Every off-by-one bug in this project will be in
  that line.
- **Bookended intervals** (`a.end == b.start`) do **not** overlap. They **do** merge
  under `merge -d 0`.
- **Zero-length intervals** (`start == end`) are legal and present in `data/a.bed` and
  `data/b.bed`. bedtools handles them in ways nobody guesses correctly. Whatever
  bedtools prints is correct — encode it in the golden test with a comment saying it is
  oracle behaviour, and do not "fix" it.
- **Minimum overlap to count:** 1 bp. No `-f` / `-r` fraction flags in v1.

## 4. Flags per subcommand

| Subcommand  | Flags in v1            | Notes |
|-------------|------------------------|-------|
| `sort`      | none                   | Default sort only: by `(chrom, start)`, stable. See "What sorted means" below — `end` is **not** a sort key. |
| `merge`     | `-d N`                 | Default `-d 0`. Merges features separated by at most N bp; `-d 0` merges bookended features. **Output is always BED3** — all columns past `end` are discarded. |
| `intersect` | `-u`, `-v`, `-wa`, `-wb` | `-u` = report each A feature once if it overlaps anything; `-v` = report A features with no overlap; `-wa`/`-wb` = write the original A / B entry. Default output shape below. |
| `subtract`  | none                   | Removes the portions of A covered by B. A features entirely covered vanish. |
| `closest`   | `-d`                   | Reports every tied B feature, not one. `-d` appends a distance column — see the formula below, it is not the raw gap. |

### What "sorted" means

All of this is **measured** against bedtools v2.31.1, not assumed. Getting it wrong
breaks every `merge` and `closest` golden test.

- **Sort key is `(chrom, start)` only.** `end` is *not* a key. *Measured: given
  `chr1 100 300`, `chr1 100 200`, `chr1 100 500`, bedtools emits them in exactly that
  order — unchanged.*
- **The sort is stable.** Records with equal `(chrom, start)` keep their input order,
  including fully identical intervals. A Python `list.sort()` keyed on `(chrom, start)`
  gets this for free; `sorted(key=lambda r: (r.chrom, r.start, r.end))` does **not** and
  will diverge from the oracle on `data/a.bed`.
- **Chromosome order is lexicographic, not natural.** *Measured:*
  `chr1, chr10, chr2, chr20, chr3, chrM, chrX, chrY`. So `chr10` sorts before `chr2`.
  Plain string comparison is correct here; do not "improve" it with natural sort.
- **The sortedness check on `merge` / `closest` only inspects `(chrom, start)`.*
  *Measured: `chr1 100 500` followed by `chr1 100 200` is accepted, exit 0.* Out-of-order
  ends are not an error.

### `intersect` default output shape

*Measured:* with no flags, `intersect` emits **A's row with `start`/`end` replaced by the
intersected span**, keeping all of A's remaining columns. Given
`chr1 100 200 nameA 42 +` against `chr1 150 300 nameB 99 -`:

```
chr1	150	200	nameA	42	+
```

`-wa` suppresses the clipping and emits A's original coordinates instead.

### `closest -d` distance — not the raw gap

*Measured, and this is the trap:* the distance is `b.start - a.end + 1` for a
downstream B, symmetrically for upstream, and `0` only for a true overlap.

| Relationship | Raw gap | bedtools `-d` |
|---|---|---|
| Overlapping | — | `0` |
| Bookended (`b.start == a.end`) | 0 | **`1`** |
| 1 bp apart | 1 | `2` |
| 2 bp apart | 2 | `3` |

The obvious implementation, `b.start - a.end`, returns `0` for bookended features and
therefore reports them as overlapping. They are not — see §3. Encode the table above.

### `closest` with no B feature on the chromosome

*Measured:* emits A's row followed by `.`, `-1`, `-1`, and `-1` for the distance under
`-d`, and exits 0. It is not an error and not skipped.

```
chr9	500	600	.	-1	-1	-1
```

- **Strand-aware flags:** none in v1 (see §1).
- **`merge` requires pre-sorted input.** It does not sort for you. *Measured: bedtools
  exits 1 with "Sorted input specified, but the file … has the following out of order
  record".* `data/a.bed` and `data/b.bed` are deliberately unsorted, so golden tests must
  sort into a temp file first.
- **`closest` requires pre-sorted input**, both `-a` and `-b`. Same measured error.
- **`intersect` and `subtract` do not require sorted input.** *Measured: bedtools accepts
  unsorted A and emits results in A's file order, not coordinate order.*

## 5. Output

- **Format:** byte-for-byte identical to bedtools for every supported flag combination.
  That is the whole point; there is no independent output design.
- **Field separator:** a single tab. No padding, no alignment.
- **Trailing newline:** every output line ends with `\n`, including the last.
- **Empty results:** print nothing at all and exit 0. *Measured: bedtools does exactly
  this.*
- **`-header`:** not supported in v1.

## 6. Memory model

- **Streaming wherever possible.** Read line by line, write as you go.
- **`sort` may hold one chromosome in memory at a time** — it cannot stream, since
  ordering requires seeing the data. This is the single documented exception.
- **`merge`, `intersect`, `subtract`, `closest` stream**, relying on their sorted-input
  requirement (§4) to avoid buffering.
- **Largest input promised:** ~500,000 intervals (roughly 15–25 MB of BED), which is what
  `bedtools bamtobed -i /data/HG002.neighbourhoods.bam` produces. Being slower than
  bedtools is acceptable — it is C. Being *quadratic* is a bug.

## 7. Errors and exit codes

| Code | Meaning |
|------|---------|
| `0`  | Success, including a run that produced no output rows, and a run that skipped malformed lines. |
| `1`  | Bad input data that prevents completion. |
| `2`  | Usage error — the tool was invoked wrongly. |

| Situation                    | stderr message                                             | exit |
|------------------------------|------------------------------------------------------------|------|
| Success                      | —                                                            | `0`  |
| Malformed BED line (bad field count, non-integer coordinate) | `mytools: skipping malformed line N in FILE: REASON` | `0`  |
| `start > end`                | `mytools: malformed BED entry at line N: start > end`        | `1`  |
| Unknown flag                 | usage text                                                   | `2`  |
| Missing input file           | `mytools: FILE: No such file or directory`                   | `2`  |
| Unsorted input to `merge` / `closest` | `mytools: input is not sorted: out-of-order record at line N` | `1`  |

- **Malformed lines are skipped, not fatal.** The run continues, a warning goes to
  stderr, and the exit code stays `0`.
- **stderr is never stdout.** stdout is data and gets piped; a warning on stdout would
  corrupt a golden test.

## 8. Correctness

- **Oracle:** real `bedtools` on the files in `data/`. Non-negotiable.
- **Golden tests in v1** — one case per row, each comparing stdout *and* exit code:

  | Case | Command |
  |---|---|
  | sort | `sort -i a.bed` |
  | sort from stdin | `sort -i -` |
  | merge default | `merge -i a.sorted.bed` |
  | merge -d 0 | `merge -d 0 -i a.sorted.bed` |
  | merge -d 10 | `merge -d 10 -i a.sorted.bed` |
  | intersect | `intersect -a a.bed -b b.bed` |
  | intersect -u | `intersect -u -a a.bed -b b.bed` |
  | intersect -v | `intersect -v -a a.bed -b b.bed` |
  | intersect -wa | `intersect -wa -a a.bed -b b.bed` |
  | intersect -wb | `intersect -wb -a a.bed -b b.bed` |
  | subtract | `subtract -a a.bed -b b.bed` |
  | closest | `closest -a a.sorted.bed -b b.sorted.bed` |
  | closest -d | `closest -d -a a.sorted.bed -b b.sorted.bed` |
  | empty result | an intersect with no overlaps — expect no output, exit 0 |

- **Unit tests**, running without bedtools installed: one per edge case — bookended,
  zero-length, nested, identical, position 0, and the overlap predicate itself.

- **Known deviations from bedtools, accepted deliberately:**

  1. **Malformed coordinates.** *Measured: bedtools v2.31.1 does not handle a
     non-integer coordinate — it throws an uncaught `std::invalid_argument` and aborts
     with SIGABRT (exit 134, core dumped).* mytools skips the line with a warning and
     exits 0. This is a deliberate improvement and **cannot be golden-tested**, because
     the oracle crashes rather than producing a reference answer. Cover it with unit
     tests instead.
  2. **Exit codes on error paths.** *Measured: bedtools exits `1` for an unknown flag and
     `1` for a missing input file.* mytools exits `2` for both, per the table in §7.
     Consequence: **golden tests may only compare exit codes on success paths.** Error
     paths get unit tests, not golden tests.

## 9. Language and layout

- **Implementation language:** Python 3 (standard library only, no third-party runtime
  dependencies). Every subcommand and every test.
- **Entry point:** a single executable file `mytools` at the repo root, with a
  `#!/usr/bin/env python3` shebang. It stays one file in v1. *Rationale: deliberate —
  parallel agents editing one file is a real cost, but a package for five subcommands is
  more structure than this earns.*
- **Tests:** `tests/run_golden.sh` for the golden suite, unit tests alongside it in
  `tests/`.
