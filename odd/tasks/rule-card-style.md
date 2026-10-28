# Rule card style: polish the shell without moving the layout

Objective: make the shell look lighter by giving it one chrome language — rule-and-text cards instead of framed,
painted panels — while keeping the current distribution (right rail, header, footer, prompt) exactly where it is.
Branch: `ui/rule-card-style` (a worktree beside the checkout, base `main` 64b2fe479).
Delivery: `ask-on-risk`. Runner: `node --experimental-strip-types --test <file>` for focused tests,
`node scripts/run-test-suite.mjs` for the suite, `node scripts/check-types.mjs` for the typecheck.

## Specs

- **S1** A new card style value `rule` renders conversation cards and docked panels as rule-and-text blocks, through
  BOTH `renderCard` paths (the `previewRows` path used by quiet tools and the legacy path used by docked panels),
  branching inside `cardTop`/`cardLine`/`cardBottom`/`panelInnerWidth` before any frame arithmetic:
  the heading is one line, `<glyph> <title>` plus ` · <subtitle>` when present and a dim `  <hint>` at the end
  (the hint is dropped before it would displace the title, and the heading never exceeds the card width);
  every body row starts with `│ `; no frame glyphs (`╭ ╮ ╰ ╯`) and no painted background.
  Existing `float` and `neon` rendering stays byte-identical (their tests must keep passing unchanged).
- **S2** `card-style.json` accepts `rule`. A **missing** file resolves to `rule` (new default); an explicit `float`
  or `neon` still resolves to itself; a **malformed or unreadable** file keeps the documented `float` fallback with
  `malformed: true`, and `writeCardStyle` still refuses to replace it. `cardStyle()` recognises all three values and
  keeps `float` for an unset slot.
- **S3** Under `rule`, a docked panel draws no float padding rows: `panelHeaderRow` returns 0 and
  `panelExtraRows` returns 0, so the heading is the panel's first row.
- **S4** In the rail each todo task occupies exactly one physical line, truncated with `…`; the dock card keeps its
  current wrapping. Rail and dock share one `renderTodoCard` call and both are `scrollable`, so the host must be
  threaded explicitly (`singleLine`), never inferred. No task content is lost from the state.
- **S5** Under `rule` the Todos heading shows only the shortcut key (`ctrl+shift+t`), never
  `ctrl+shift+t collapse`/`expand`, because the heading already carries the action word. `float` and `neon` keep
  their current hint bytes. The clickable control and its hover treatment keep working.
- **S6** A rail line wider than the rail's content width must never disable the sidebar: the line is clamped to the
  content width and `sidebarState(...).active` stays true while the rail has sections. This is the defect observed in
  the prototype, where a single over-wide heading silently moved the plan and changes into the bottom dock. The
  existing test that encodes the old contract (`overflows leave native layout intact`) is rewritten, with a comment
  naming the change.
- **S7** `/gentle:customize` lists `Card style: rule` with a preview sample, and `docs/gentle-shell.md` documents
  the three values plus the new default.
- **S8** Focused tests cover S1–S6; the full suite, provider-contract, runtime harness and typecheck pass; the result
  is captured at 242 and 160 columns with the audit harness outside the repository (no in-repo capture command
  exists, and the performance worker asserts identical screens across roots, so it cannot express base-vs-change).

## Tasks

| ID | Spec | Route | Status | Commit |
|---|---|---|---|---|
| T1 | S1, S3 | inline | done | 4e87da6d7 |
| T2 | S2 | inline | done | 4e87da6d7 |
| T3 | S4, S5 | inline | done | cb960e73d (+ the widget-test adaptation) |
| T4 | S6 | inline | done | 901d7709e |
| T5 | S7 | inline | done | 4e87da6d7 |
| T6 | S8 | inline | done — evidence in the vault `shots/impl-*.png` | 4e87da6d7..901d7709e |

Three commits carry six tasks because T1/T2/T5 landed in one reviewable slice (the style and the
preference that selects it are one behavior) and T6 produces no source change of its own.

## Log

- L1 (owner, verbatim): "me gusta mas arma el plan y verificalo luego un agente con memoria limpia lo verifica y
  empiezas, no se como se haga si con un fork o como pero evalualo y empiezas con el plan listo"
- L2 (context): two earlier rounds were rejected. The first moved four constants; the second removed the rail and
  pushed the plan and agents into the bottom dock. The owner rejected both and asked for the current distribution
  to stay, with only the finish improved. The accepted look is `ui-pulido.html` in the vault audit folder.
- L3 (decision): work continues in this repository on a branch, not a new GitHub fork. The repository is already a
  fork; a branch keeps review local and `main` untouched. No push.
- L4 (decision): `rule` becomes the default only when `card-style.json` is missing. An explicit `float` or `neon`
  is never rewritten, so existing setups keep their look until the user picks the new value in `/gentle:customize`.
- L6 (verification, clean-context agent, read-only): S1 must branch in both `renderCard` paths and inside the frame
  helpers (`cardTop` hardcodes `-5`, `cardLine` uses `FRAME_COLUMNS = 4`); S2's missing-vs-malformed split is required;
  S4 has no rail/dock discriminator today (both `scrollable: true`); S5 contradicts S1 unless scoped to `rule`;
  S6 breaks exactly one existing test that deliberately asserts overflow disables the rail; S7's preview map is a
  literal Record that silently yields `undefined` under `strict:false`, and `Cards · 2/4` re-baselines to `3/5`;
  S8 has no in-repo capture path. Agents also duplicate the action word in its heading — out of scope, recorded as a
  follow-up.
- L5 (baseline, observed on 64b2fe479 before any edit): unit suite PASS; `node scripts/check-provider-contract.mjs`
  exit 0; `node --experimental-strip-types tests/runtime-harness.mjs` exit 0; `node scripts/check-types.mjs`
  exit 0. `pnpm` is not on this machine's PATH, so the suite's two wrapper stages fail with
  `pnpm: command not found`; they were run directly instead.
- L7 (implementation, observed): T1/T2/T5 in `4e87da6d7`, T3 in `cb960e73d`, T4 in `901d7709e`. The rule
  branch lives inside `cardTop`/`cardLine`/`cardBottom`/`contentColumns`/`panelInnerWidth`, so the frame
  arithmetic is never reused; a blank body row keeps its bar, as in the approved prototype (`│` alone).
  `floatOpener` already returns nothing for non-float styles, so `panelHeaderRow`/`panelExtraRows` are 0
  under rule for every theme, not only for a theme without `bg` — the prototype returned 1 there and its
  rail slice dropped rows; this implementation does not.
- L8 (hand-check, real build, no env flags): the harness builds from `git archive HEAD` and captures
  `impl-242`, `impl-160`, `impl-card-242` (expanded bash card), `impl-390`, `impl-1440`. Observed: rail
  headings `✿ Status` / `❀ Todos ▾ Collapse · 1 of 4  ctrl+shift+t` (key only), one physical row per task
  ending in `…`, every card body row behind `│` with the heading and the bar in the same column, no frame
  glyph outside the prompt, no painted panel. The rail keeps its sections at 160, 390 and 1440 columns.
  The prompt keeps its outlined box on purpose (documented).
- L9 (findings, out of scope, not fixed): the rail's `│` uses `FRAME_ROLE[info]`, which this theme maps to
  `border` (#5c554a) and reads almost invisible against #1c1b19; the prototype had the same value, so the
  finish matches what was approved, but the rail loses its vertical cue. The transcript Todo card renders
  twice per write (also on the base commit, so pre-existing). Prose still stretches to the transcript width
  at 1440 columns (the audit's measure finding, not this branch).
