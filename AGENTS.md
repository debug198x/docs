# Debug198x docs

> Read [`PRINCIPLES.md`](PRINCIPLES.md) first. [`MANIFESTO.md`](MANIFESTO.md) is why the project exists.

This repo owns the `.debug198x` format specification. It sits inside the `Debug198x/` org container alongside the reference crate (`../debug198x/`) and the public site (`../debug198x.github.io/`).

## Read first

- [`debug198x.md`](debug198x.md) — the format specification.
- [`../debug198x/AGENTS.md`](../debug198x/AGENTS.md) — the crate's rules before code changes.

## Ownership

- Commit specification changes here: `Debug198x/docs/`.
- Commit crate changes in `Debug198x/debug198x/`.
- Change the spec and the crate in the same change, not one after the other: the specification is the contract Asm198x and Emu198x are both written against, and a spec that leads or lags the code is worse than no spec.
