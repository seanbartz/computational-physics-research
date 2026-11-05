# Computational physics research — a Claude skill

A skill that makes Claude build physics simulations the way a careful researcher would:
one small increment at a time, each one checked against something already known, with the
validation record, the git history, and the paper notebook built as you go rather than
reconstructed at the end.

It is written for research codes that have to survive a referee — and, more to the point,
have to still make sense to you six months later.

## What it does

When the skill is active, Claude will:

- **Plan before coding.** It proposes a staged plan — the dimensionless formulation, the
  independent parameters, and one increment at a time each paired with the check that will
  validate it — and waits for your approval. The plan lives in `PLAN.md` with increments
  checked off as they pass.
- **Nondimensionalize, and push back on the setup.** It picks natural scales, counts the
  dimensionless groups the physics actually depends on, and argues with you *before*
  running a sweep over redundant parameters. Physically loose definitions (an escape
  radius that doesn't guarantee escape, a cutoff defining a "bound" state) get flagged and
  tested rather than quietly coded.
- **Scaffold from a single run.** The first deliverable is always a tool that runs *one*
  case and prints numbers. Sweeps, maps, and parallel runs get built on top of it only
  after it is validated, and `run_single.py` stays in the project forever as the way to
  inspect any point of any later sweep.
- **Validate everything.** Known results and analytic limits, published numbers from
  papers you name, conserved quantities with the drift reported as a number, convergence
  with the observed order, and regression tests against saved reference outputs. Every
  check becomes a `pytest` test and a line in a validation table.
- **Reproduce published results on request.** Point it at a paper and a figure, table, or
  equation; it maps conventions onto yours, records the target with its citation and a
  justified tolerance in `tests/published/`, and writes the test. Disagreement is reported
  as a finding, not smoothed over.
- **Write code for human readers.** Full module docstrings with the equations, the
  nondimensionalization, the method and *why* it was chosen, and its known error behavior.
  Descriptive names over terse symbols. Results carry their parameters, solver settings,
  and git hash.
- **Keep two notebooks.** `progress_summary.ipynb` is the validation record, extended in
  the same commit as every increment. `paper_figures.ipynb` regenerates every paper figure
  and quoted number with "Restart & Run All."
- **Estimate cost before spending it.** It times one point, projects the sweep, and tells
  you the number before launching it — and suggests a faster implementation (vectorize →
  Numba → compiled) or a cluster only when the estimate justifies it, always keeping the
  trusted version as the oracle for the fast one.

## Installing

### Claude Code

Copy the skill directory into your personal skills folder:

```bash
git clone https://github.com/seanbartz/computational-physics-research.git
```

```bash
cp -r computational-physics-research/computational-physics-research ~/.claude/skills/
```

Or, to make it available only in one project, copy it into that project's
`.claude/skills/` directory instead. Restart Claude Code and the skill will be listed.

### claude.ai

Upload `computational-physics-research/SKILL.md` as a skill in your settings.

## Using it

Mention what you're building and the skill triggers on its own:

> Set up a simulation of a charged particle in crossed E and B fields; I want to get to a
> map of escape times over the field-strength plane.

> Reproduce Fig. 3 of arXiv:XXXX.XXXXX, then extend it to finite chemical potential.

Expect Claude to come back with a plan and a question rather than a directory full of
code. That's the point — the plan is where your knowledge of the literature and the physics
gets built in. Edit it, swap in the limits and known results *you* trust, and then let it
build.

## Assumptions

- **Python**, with numpy/scipy as the default toolbox, `pytest` for checks, and Jupyter for
  the two notebooks. Numba/JAX/compiled routines appear only as a later, validated
  optimization increment.
- **Git** for every project, with the hash recorded in saved results so any figure can be
  traced to the code that made it.
- **No particular hardware.** Parallel sweeps size themselves from `os.cpu_count()`, and
  the cluster guidance is scheduler-agnostic; the SLURM job array in Appendix B is an
  example to adapt, not a requirement.

## Contents

```
computational-physics-research/
└── SKILL.md    # the whole skill: workflow, plus appendices on validation,
                # parallelization, and the publication notebook
```

## License

MIT — see [LICENSE](LICENSE).
