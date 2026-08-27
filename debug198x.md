# The Debug198x format

> **Status: frozen at v1 (2026-08-18).** Record shapes and field names are
> stable. The format evolves **additively** from here — new record types and new
> fields may be added at any time, and a conforming reader must tolerate both
> (see *Three rules make the format durable* below). A **breaking** change now
> requires a new decision, a `format_version` bump, and a migration path.
>
> `format_version` remains `"0.1"`, and that string denotes the frozen v1
> specification. It was deliberately not bumped at the freeze: only a breaking
> change moves it, and nothing about the wire format changed. Do not read
> `"0.1"` as "still draft".
>
> The freeze event, the evidence it was gated on, and the rules for evolution
> afterwards are governed by the
> [format decision record](https://github.com/asm198x/asm198x/blob/main/decisions/debug198x-format.md).

Debug198x is the 198x family's **cross-CPU debug-info format**: one sidecar
file that maps assembled bytes back to source lines and symbols, identically
for a C64, Spectrum, NES, Amiga, or Intellivision program. Asm198x writes it
(`--debug`); a debugger — first the Emu198x importer — reads it to render
`jsr init` instead of `jsr $c012`, resolve a breakpoint set on a source line
to an address, and highlight the line when it hits.

This page is the format's specification, written so that you can **implement a
reader from this page plus the
[conformance fixtures](https://github.com/asm198x/asm198x/tree/main/crates/asm198x/tests/fixtures/debug198x)
without reading Asm198x source**. The fixtures are the spec's executable half:
each is a source file plus its exact expected sidecar, enforced in CI.

## The file

A `.debug198x` file is **NDJSON**: UTF-8 text, one JSON object per line. Every
record carries a `t` field naming its type. Four types exist today:

| `t` | one per file? | carries |
|-----|---------------|---------|
| `header` | yes, first line | format + tool identity, CPU, dialect, sources |
| `section` | one per section | id, name, optional absolute base |
| `symbol` | one per symbol | name, kind, location or value |
| `line` | one per source-bearing span | file, line, section, offset, length |

Three rules make the format durable, and at v1 they are promises rather than
intentions:

- **A reader must skip records with an unknown `t`** instead of failing. New
  record types are added this way, without a version break.
- **A reader must ignore unknown fields** inside a known record. New fields are
  added this way.
- **A reader must carry a `space` shape it does not recognize** rather than
  reject the file, and should give its caller a way to find out that it did —
  see *The consumer model*. New address-space shapes are added this way.

Numbers are **decimal JSON integers** throughout, u64-capable. Hexadecimal is a
rendering concern for tools, never a wire concern.

## `header` — identity

```json
{"t":"header","format":"debug198x","format_version":"0.1","tool":"asm198x","tool_version":"0.0.7","cpu":"z80","dialect":"pasmo","sources":["game.z80"]}
```

| field | meaning |
|-------|---------|
| `format` | always `"debug198x"` |
| `format_version` | the version of *this specification* the file conforms to |
| `tool`, `tool_version` | the producing tool — informational, never load-bearing |
| `cpu` | the target CPU (`"z80"`, `"6502"`, `"68000"`, `"cp1610"`, …) |
| `dialect` | the source syntax (`"pasmo"`, `"acme"`, `"ca65"`, `"vasm"`, `"asl"`, …) |
| `sources` | the source file(s) the image was assembled from, in producer file-id order |

`sources` is **ordered**: `sources[0]` is the root input, and each included
file appears exactly once, in the order the assembly first reached it — the
producer's file-id order. A file included twice still gets one entry (its
first inclusion). Every `line` record's `file` matches one of these entries
verbatim, so a consumer can key files by either the string or its index.
Binary assets pulled in by an `incbin`-style directive are data, not source —
they never appear in `sources`.

Branch on `format_version` for a breaking change; additive changes never bump
it incompatibly.

## `section` — where bytes live

Every address in the file is a **(section, offset)** pair. A section is a
segment of the image with an id, a human-readable name, and — when the
producer knows it — an absolute base address.

```json
{"t":"section","id":4,"name":"CODE","base":32768}
{"t":"section","id":0,"name":"code"}
{"t":"section","id":0,"name":"bank1","space":{"slot":3,"page":1}}
```

| field | meaning |
|-------|---------|
| `id` | the integer other records reference |
| `name` | the segment/hunk name (`"main"`, `"CODE"`, `"ZEROPAGE"`, `"code"`, `"bank1"`) |
| `base` | absolute address of offset 0 — **absent when the section is relocatable or not CPU-addressable** |
| `space` | optional [address-space qualifier](#space--the-address-space-qualifier) for the section as a whole — on a banked machine, the (slot, page) it lives in |

Three base postures cover every producer:

- **Flat and linked-absolute images** (the flat engine's single `main` section,
  ca65's NES segments) carry their real base: absolute addressing is the
  degenerate case of (section, offset), with zero ceremony for the reader.
- **Relocatable sections** (Amiga hunks) carry no base. The reader's caller
  supplies a **base map** — section id → loaded address — at lookup time; see
  *The consumer model* below.
- **Non-CPU-addressable sections** (the NES iNES header, CHR data in PPU
  space) carry no base either, so they can never alias a CPU address in a
  lookup. A consumer that addresses another space (a PPU viewer) supplies its
  own base map for them.

A **pageable** section is the relocatable posture with a known destination: no
`base` (where it lands depends on what is paged in), but a `space` naming the
(slot, page) it belongs to. That pairing is what lets a consumer turn a live
paging state into a base map without inspecting a single symbol — see *The
consumer model*.

### Sections that contribute no bytes

A section can describe address space that is **not in the image**. The asl-family
dialects are where this shows up on the flat path: `asl` reserves without
writing and `p2bin` starts the file at the lowest written address, so a leading
`org` gap or reservation moves the load address instead of padding it.

```asm
buf:    ds 3        ; reserved — no bytes in the image
start:  db 9        ; the image begins here, at $0003
```

```json
{"t":"section","id":0,"name":"main","base":3}
{"t":"section","id":1,"name":"reserved","base":0}
{"t":"symbol","name":"buf","kind":"label","section":1,"offset":0}
{"t":"symbol","name":"start","kind":"label","section":0,"offset":0}
{"t":"line","file":"t.s","line":2,"section":0,"offset":0,"length":1}
```

`buf` is a real label at a real address, but it sits below where the image
begins and offsets are unsigned, so it cannot be expressed against `main`. The
`reserved` section is based where the source's addresses started and holds no
bytes — a BSS region, expressed with the fields sections already have.

Two things follow for a consumer:

- **Do not assume section ids run in image order, or that every section has
  bytes.** `main` keeps id 0 whether or not a `reserved` section exists, so a
  sidecar without one is unchanged. Anything mapping sections to file offsets by
  accumulating lengths must skip a section that contributes none.
- **A `reserved` section carries symbols but no `line` records.** A line span
  describes bytes that exist; these do not.

## `symbol` — names

```json
{"t":"symbol","name":"start","kind":"label","section":0,"offset":0}
{"t":"symbol","name":"start","kind":"entry","section":0,"offset":0}
{"t":"symbol","name":"BORDER","kind":"const","value":254}
{"t":"symbol","name":"draw","kind":"label","section":0,"offset":16,"space":{"slot":3,"page":1}}
```

| `kind` | fields | meaning |
|--------|--------|---------|
| `label` | `section`, `offset`, optional `space` | a code/data location |
| `entry` | `section`, `offset`, optional `space` | the program's entry point (an `end <addr>` targeting a label upgrades that label in place — one symbol, not two) |
| `const` | `value` | an `equ`/`=` binding — a value, never an address |

The `const`/`label` split is load-bearing: a constant sharing its low bits with
a label's address stays distinguishable, and constants never answer
address-to-symbol lookups.

### `space` — the address-space qualifier

Most records carry no `space` field: a flat CPU's addresses need no
qualification, and **producers never fabricate one** (a Z80 file contains no
bank data; a 65816 program placed in bank 0 carries plain addresses). It may
appear on a `section` (the space that section as a whole lives in) and on an
address-kind `symbol` (the space that one address lives in); a symbol's own
`space` is the finer truth where it carries one, and its section's is the
default for everything in it — including `line` records, which carry no
qualifier of their own. Two shapes exist for machines that need more:

- `{"slot": 3, "page": 1}` — a banked/paged location: hardware slot `slot`
  with memory page (bank) `page` paged into it. Two symbols sharing a CPU
  address in different pages stay distinct. **`page` is the join key**: it says
  which bank the code belongs to, while `slot` records where the producer
  expected that bank to appear. A consumer building a base map from paging state
  matches on `page` alone — matching the pair hides a bank the machine has put
  somewhere else, and a page can legitimately be live in two slots at once (a
  128K keeps bank 5 at `$4000` *and* can select it into `$C000`).

One shape only, deliberately. A **reader must carry a `space` shape it does not
recognize rather than reject it** — the record still resolves through its
section, and the qualifier is treated as uninterpretable. This is the
skip-unknown rule applied to `space`, and it is what keeps the set of shapes open
after v1: without it, a reader meeting a future shape fails the whole file.

The paged shape is exercised today by the hand-authored Spectrum 128 fixture
(`spectrum128-banked.*`), on its sections and its symbols alike, and is
cross-checked against a real 128K paging model in Emu198x; emission paths
populate it when a machine needs it.

A `{"bank": <n>}` shape for 65816-style bank bytes was specified in draft
0.1 and **withdrawn before v1**, having never been emitted by a producer, pinned
by a fixture, or read by a consumer — and being derivable anyway, since a 65816
address resolves to its full 24-bit value and the bank byte is its top 8 bits. A
file carrying one still loads: it is an unrecognized shape, and unrecognized
shapes are carried. A real banked shape can be added when a 65816 consumer has an
actual requirement for one.
The fixture's companion table (`spectrum128-banked-sld.md`) shows every banked
record projecting onto sjasmplus SLD long addresses by pure arithmetic — the
record model loses nothing an SLD consumer needs.

## `line` — the source map

```json
{"t":"line","file":"game.z80","line":4,"section":0,"offset":0,"length":2}
```

`length` units at `(section, offset)` were produced by line `line` of `file`.
One record per source-bearing statement. Three rules:

- **The padding rule:** bytes with no source — `org` gap fill, alignment
  padding (an Amiga `even`, a linker gap) — belong to **no** line record. A
  reader asked about such an address answers "no source here", which is the
  truth.
- Lines that emit nothing (`equ`, comments, blank lines) get no record; their
  text is in the source file the header names.
- **The own-file rule:** `file` is the file the line counts within, spelled
  exactly as its `sources` entry — a line inside an included file names *that*
  file and its own 1-based line number, never the root input that included it.
  The `z80-spectrum-multifile` and `6502-nes-multifile` fixtures pin this: the
  included file's bytes carry `"file":"…multifile.inc"` records.

A binary-inclusion directive (`incbin` and its dialect kin) produces **one**
record covering the whole payload: `file` and `line` are the directive's own
position, `length` the payload's size in address units. The pulled-in binary
is data, not source, so nothing points into it and it never appears in
`sources`.

## Address units

Offsets, lengths, and bases are in the **CPU's own address units** — what the
programmer's addresses mean on that machine. For every byte-addressed CPU that
is bytes. For the word-addressed CP1610 (Intellivision) it is **decles**
(16-bit words): a 12-byte image is 6 decles, its spans total 6, and a label's
address is a decle address, matching how every Intellivision address is
written. The `cp1610-intellivision` fixture pins this: span lengths sum to
half the image's byte count.

## The consumer model

A reader implements three lookups over the records (the fixtures exercise all
three):

- `addr_of(name)` — symbol name → absolute address.
- `symbol_at(addr)` — absolute address → the symbol defined there.
- `line_at(addr)` — absolute address → the line record covering it.

Each takes an optional **base map** (section id → absolute base). Resolution
of a record's absolute address is: *the base map's entry for its section if
present, else the section's recorded `base`, else the record does not resolve*
— it is silently outside the addressable world for this lookup.

That last clause is the design's hinge, and it is deliberate:

- **A relocatable program** (Amiga hunks) resolves once the loader's actual
  hunk addresses are supplied as the base map.
- **A banked machine's paging state *is* the base map.** Map only the sections
  currently paged in: with bank 1 in slot 3, `symbol_at($C010)` names `draw`;
  swap the map to bank 3 and the same address names `music`. No map, no
  answer — which is correct, because without paging state the question is
  ambiguous.

  Build that map from the sections' `space`: for each slot, take the sections
  whose `space` names the page currently in it and map them to the slot's
  address. A page the image has no code in contributes nothing, so the lookup
  declines to answer instead of answering from another bank. Mapping two pages
  of one slot at once describes a state the hardware cannot be in — one slot
  holds one page — and the lookups will answer from whichever record comes
  first rather than reporting the contradiction.
- **Non-CPU sections never pollute CPU lookups**, because nothing maps them.
- **A section whose `space` the reader cannot interpret should be surfaced, not
  silently left unmapped.** An unreadable shape matches no paging state, so it
  never maps and its symbols never resolve — which from the outside looks exactly
  like a bank that is paged out. Both are "no symbols here". Carrying an unknown
  shape rather than rejecting the file is the right trade, but it converts a loud
  failure into a quiet one, so a reader **should** give its caller some way to
  ask "is anything here described in a way I cannot read?" — otherwise a producer
  typo costs symbols and leaves nobody a thread to pull. Emu198x's importer does
  this with an `unreadable_spaces()` query.

A base-map entry always wins over a recorded `base`, so a consumer can rebase
even a based section (a flat blob loaded somewhere unusual).

## Producer guarantees

- Writing a sidecar **never changes the assembled image**: bytes with and
  without `--debug` are identical. The fixture corpus asserts this for every
  family.
- Addresses are **post-link** (what a debugger needs), never file offsets:
  a linker-placed NES label resolves to its ROM address.
- Sidecars are deterministic for a given source and tool version.

## Changelog

- **Zero-byte sections documented** (2026-08-19) — a producer may emit a
  section that contributes no bytes, to hold symbols in address space the image
  does not cover. The flat asl-family path does this for a trimmed leading gap.
  **No shape changed**: it uses `sections` and `base` as they already are, and a
  sidecar without such a region is byte-identical to before. See *Sections that
  contribute no bytes*.
- **v1 frozen** (2026-08-18) — record shapes and field names are stable; the
  format evolves additively from here, and a breaking change needs a new
  decision, a `format_version` bump, and a migration path. **No shape changed at
  the freeze**, and `format_version` deliberately stays `"0.1"`, now denoting the
  frozen v1 specification. Gated on: the Emu198x importer consuming a real
  sidecar end-to-end, the banked fixture's three validation legs (the last
  cross-checked against a real 128K paging model), and a bounded review that ran
  twice — producing `space` on `Section`, then the unknown-shape rule and the
  withdrawal of the unexercised `bank` shape.

- **0.1** (2026-08-18) — **unrecognized `space` shapes are carried, not fatal;
  the `bank` shape is withdrawn.** A reader that meets a `space` it does not know
  keeps the record and treats the qualifier as uninterpretable, so a future shape
  cannot fail an older reader's parse. Without this the set of shapes would close
  permanently at v1, because `space` is matched structurally. `{"bank": <n>}`
  is withdrawn unexercised — never emitted, fixtured, or read, and derivable from
  a 24-bit address — leaving `{"slot", "page"}` as the one shape, which is
  fixture-pinned and hardware-cross-checked. Files carrying a `bank` still load.
  Also documents that **`page`, not the `(slot, page)` pair, is the join key**.
- **0.1** (2026-08-18) — **`section` gains an optional `space`**, additively: a
  pageable section can now state the (slot, page) it belongs to, so a consumer
  turns a live paging state into a base map by lookup instead of inferring it
  from whichever symbol happens to sit in the section — and a section holding
  only `line` records places at all. A record's own `space` stays the finer
  truth; its section's is the default. Absent on every flat producer, so all
  existing sidecars are byte-identical and older readers are unaffected. The
  `spectrum128-banked` fixture carries section spaces.
- **0.1** (2026-07-07) — multi-file data semantics specified; **no shape
  change**. `sources` is ordered by the producer's file table (`sources[0]` =
  the root input, included files in first-inclusion order); a `line` record's
  `file` names its own file verbatim from that list (the own-file rule); a
  binary inclusion (`incbin` and kin) yields one record covering the whole
  payload at the directive's position. Exercised by the new
  `z80-spectrum-multifile` and `6502-nes-multifile` fixtures.
- **0.1** (2026-07-06) — initial draft: `header`/`section`/`symbol`/`line`
  records; label/entry/const symbol kinds; bank and paged space shapes; the
  section/offset addressing model with base-map rebasing; the padding rule;
  address units; unknown-record and unknown-field tolerance.
