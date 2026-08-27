# debug198x docs

Documentation for the [Debug198x format](https://github.com/debug198x/debug198x)
— the cross-CPU debug-info sidecar written by Asm198x and read by Emu198x.

- **[The Debug198x format](debug198x.md)** — the specification. Record shapes,
  field names, address-space qualifiers, and the compatibility rules a reader
  must follow. Frozen at v1 on 2026-08-18; it evolves additively from here.

The design reasoning behind the format lives with the code, in
[`decisions/debug198x-format.md`](https://github.com/debug198x/debug198x/blob/main/decisions/debug198x-format.md).
