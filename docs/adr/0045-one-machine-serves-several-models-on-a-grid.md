# One machine serves several models on a grid

Status: proposed (2026-09-30) — supersedes **ADR 0007 D4** ("the built-in launch stays single-engine")
and the external-only guard of **ADR 0010 D2**, and the "aliases don't merge across joins" rule that
ADR 0010's review round 3 added.

## Context

Remote mode has one identity per (machine, grid): the relay node id comes from the per-grid token
(ADR 0010). Every `grid join` on that grid therefore adds to the one union that machine serves. The
union could hold any number of `--at` and API engines, but **at most one built-in `--serve` model, and
only when it was the sole engine**, and an `--advertise-as` alias could not survive a second join.

That is a real ceiling, not a corner case. A machine with room for a coding model and a vision model
could serve only one of them through Grid's own engine. The error pointed at `grid leave` and
re-joining "every engine as external `--at`" — but a Grid-launched llama-server has no URL to re-join
by, so the advice could not be followed. Operators who alias every model (the Harness agent does) could
not add a second engine at all.

The limit was never in the serve loop. ADR 0007 already routes each job by `body["model"]` through a
`model → llm_url` table built from any number of `(llm_url, models)` pairs. Two things kept the built-in
out of it:

1. **Launch settings lived at the top of the record.** `--ctx-size`, `--parallel`,
   `--reasoning-budget`, … were record fields, so each join overwrote them. With two built-ins, the
   second join would have retuned the first.
2. **Aliases were one flat list** positionally keyed to the union, so after a second join it could no
   longer say which engine an alias belonged to (ADR 0010 round 3 refused the append for that reason).

## D-a — Every built-in spec launches its own llama-server

`_bring_up_engines` launches one llama-server per built-in spec, one after another. Each takes its own
port starting from its `endpoint_port`, skipping ports in use — which include the ones launched just
before it, so there is no port bookkeeping. A spec that fails to come up stops the servers launched
before it (the existing teardown). Routing is unchanged: each server is one more `(llm_url, models)` pair.

## D-b — A built-in spec carries its own launch settings

The `--serve` spec is written as `{endpoint_url: null, models: [file], launch: {endpoint_port, ctx_size,
n_predict, parallel, flash_attn, mmproj, temp, reasoning_budget}}`, from the flags of the join that
named it. `run_records.builtin_launch` reads it, and falls back to the record's top-level fields for a
record written before this. `--parallel` is per engine; the `--max-concurrency` it defaults from stays
the identity's (ADR 0009). The top-level fields are still written, for older readers.

Re-joining the same built-in with new flags is still a no-op, exactly as before; `--respawn` applies
them. The new flags are stored on the spec either way, so the respawn uses them.

## D-c — Aliases belong to the engine they were given for

`--advertise-as` is stored on the spec it names (`spec.advertise_as`, one per model) and merges like
models do: a model keeps its alias, a join naming it again may give it a new one, and a model never
aliased is advertised as itself. `run_records.spec_aliases` reads a spec's own list, else the record's
flat list — which only ever belonged to a sole engine, so it applies to a lone spec and to nothing in a
union. The record's top-level `advertise_as` becomes `run_records.union_aliases`: every advertised name
once, in order — exactly the old list for a single engine, and a set every existing reader
(`own_model_case`, `grid engines`, the Harness) already reads correctly.

A merge that would give two models of one engine the same alias is refused by the CLI **before**
anything is stopped, instead of by the child after the engine already serving was stopped.

## D-d — The join adopts a legacy record before merging

`_engine_union` moves a legacy record's top-level aliases and launch settings onto its sole spec
before the merge (`_owning_its_settings`). Without it, the first join after an upgrade would retune or
un-alias the engine that was already serving. `grid leave --engine` goes through the same union, and
now also matches an alias — a built-in has no URL, so its model or alias is the only handle a person
has on it.

## D-e — Adding or removing a built-in respawns; an older child is never signalled what it can't read

A hot reload re-advertises; it cannot launch or stop a llama-server. So any union holding a built-in
respawns on change, as `_hot_reloadable` already required for a lone built-in (ADR 0010 D3). External
and API engines still hot-reload.

An older child's reload reads the flat alias list and refuses a union with several aliased engines —
after the CLI had printed "hot-reloaded". The child now stamps `per_engine_aliases: true` itself at
start-up (`_stamp_own_pid`, beside its pid). The CLI carries it across a hot reload (it describes the
process, which did not change) and clears it on respawn (the new child states it). `_hot_reloadable`
requires it for a union with more than one engine that uses aliases; without it, it respawns.

## Consequences

- The ceiling is gone: one machine can serve several built-in models and external engines on one grid,
  each with its own port, context, slots, reasoning budget, projector and alias.
- Memory is the operator's call: nothing prevents launching more models than fit. Each llama-server
  reserves its own KV cache (`--ctx-size` × slots), as one did before.
- Adding a built-in restarts the ones already running (seconds to minutes for large models). Launching
  or stopping one built-in inside a running child, zero-drop, is the natural next step and needs the
  reload to own launched processes.
- Local mode is untouched: there, each join is already its own engine record.

## Verified live (2026-09-30, grid.autonomous.ai, one Apple M1 Pro)

A legacy identity serving one built-in model (joined by v0.3.46), then, with this change:
`--serve` of a second GGUF with `--advertise-as` and `--ctx-size 8192` → "Appended … engines=2", two
llama-servers on two ports, both listed by the relay within a second and both answering through it;
then `--at` an mlx-lm server with `--advertise-as` → engines=3, the record kept each engine's alias and
the second built-in's `ctx_size`, and all three answered through the relay; then
`grid leave --engine <alias>` twice → back to the first model, which kept serving.
