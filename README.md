# NextGSystem Study

A study of the [srsRAN Project](https://github.com/srsran/srsRAN_Project) gNB.
We read the O-CU / O-DU disaggregation, find where efficiency actually lives,
and test every restructuring idea as a **falsifiable hypothesis** against a
measured baseline — rather than asserting it. The standing questions are how to
make the RAN more efficient and how AI-RAN attaches to it.

> Status: **bootstrap**. The literature survey is largely complete (on Drive).
> srsRAN code analysis and baseline bring-up are next. No experiments have run
> yet — the hypotheses are stated, not answered.

## Quickstart — open the site

The website is a single self-contained `index.html` (no build step):

```sh
python3 -m http.server --directory . 8793
# then open http://localhost:8793/
```

It is the project's front door — architecture, design, the DSL, the performance
framework, the hypotheses, the roadmap, and the survey, one navigation surface.

## Where to go

| I want to… | Open |
|---|---|
| See the whole project in 60 seconds | [`index.html`](index.html) → Overview |
| Understand the gNB disaggregation (CU / DU / RU, F1 / E1 / FAPI / OFH) | `index.html` → Architecture |
| Read the restructuring thesis and operating principles | `index.html` → Design |
| See the declarative experiment/deployment language (planned) | `index.html` → DSL |
| See the evaluation dimensions and the configuration space | `index.html` → Performance |
| See the falsifiable claims we intend to test | `index.html` → Hypotheses |
| See what's done / in-flight / missing | `index.html` → Roadmap |
| See the literature scope | `index.html` → Survey (corpus maintained privately, shared with collaborators on request) |

## Principles

- **Evidence is falsifiable.** Every efficiency claim is a hypothesis with a
  verifier and a numbered run record — never an assertion.
- **Point, don't duplicate.** The site links to analysis, specs, and result
  pages; it does not reproduce them.
- **Restructuring lives at the interfaces.** F1 / E1 / FAPI / OFH are the seams
  the gNB is already cut along, so a proposal is "redraw this boundary."
- **Measure against a reproducible baseline.** RF-free (ZMQ / RU emulator)
  bring-up first, so a teammate reproduces results without a radio.

## Roadmap

| Phase | Goal | State |
|---|---|---|
| 0 · Bootstrap | Scope, principles, this site | site done; repo + submodule to do |
| 1 · Survey | Map AI-RAN, RAN efficiency, scheduling | corpus done (Drive); srsRAN lens to write |
| 2 · Analyze srsRAN | RF-free baseline, architecture map, bottleneck inventory | to do |
| 3 · Restructure | Numbered RFCs proposing boundary moves | to do |
| 4 · Study | Run the hypotheses as experiments | to do |
| 5 · Synthesize | Roll findings into a paper / report | to do |

Phases 1 and 2 run in parallel — a natural split for two teammates (one reads,
one reads the code), converging at Phase 3.

## Layout (as it fills in)

```
NextGSystem-Study/
├── index.html        the project site (the front door)
├── README.md         you are here
├── srsran/           srsRAN Project core — git submodule slot (to pin)
├── survey/           the srsRAN restructuring lens over the Drive corpus
├── analysis/         architecture / components / baseline / bottlenecks
├── specs/            declarative experiment & deployment specs
├── runs/             numbered, immutable experiment records
└── restructure/      numbered restructuring RFCs
```

The skeleton exists; each folder carries a README explaining what fills it.
All are empty for now — which is correct at bootstrap. The project advances by
filling them in order: pin `srsran/`, run the RF-free baseline into
`analysis/baseline/`, then let `specs/` → `runs/` → `restructure/` follow.
