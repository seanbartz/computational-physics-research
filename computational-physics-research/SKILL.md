---
name: "computational-physics-research"
description: "Plan, build, validate (known/published results, conservation, convergence), and git-track physics simulations step by step, scaffolding from one run to parallel sweeps and a paper notebook."
---

# Computational physics research

The goal is results a referee (and the user, six months from now) can trust. Trust comes
from building in small steps where every step is checked against something already known,
and from code a physicist can read, run on one parameter point, and interrogate.

Treat the user as a working physicist: skip textbook exposition, but be explicit about
numerical choices (method, tolerances, grids, scales) because those are where silent errors
hide. Use American English spelling in code, comments, and prose (center, normalize,
vectorize, behavior).

## Start with a plan the user approves

Before writing any code for a new project — or for a new phase of an existing one — propose
a plan and wait for the user's go-ahead. The user knows the literature and the physics
better than any default checklist, so the plan is where their knowledge gets built in.

The plan (short; LaTeX for math) contains:

1. **Setup**: the dimensionless formulation, the independent dimensionless parameters, and
   any pushback on the request (redundant sweeps, loose definitions — see below).
2. **Staged increments**, starting from a single run, each one line: what gets added,
   and the **proposed check** for it (known result, conserved quantity, convergence test,
   comparison with the previous stage).
3. **An explicit invitation** for the user to:
   - swap in or add known results they trust (an analytic limit, a toy model, a number
     from their own earlier work);
   - name **published results to confirm** — a paper (arXiv ID, DOI, or a PDF they
     attach) and the specific figure, table, or equation to reproduce.
4. **Cost outlook**: rough runtime, and whether parallel runs or a compute cluster are
   likely to be needed.

Then stop and wait. Update the plan based on the reply and restate only what changed. Once
approved, the increments in it don't each need separate approval; propose a revised plan
whenever the direction changes (a check fails in a way that changes the approach, a new
phase starts, a speedup or major restructuring is on the table). If the session is clearly
unattended, write the plan down (in `PLAN.md`), state that you proceeded without approval,
and follow it.

Keep the approved plan in `PLAN.md` in the repository, checking off increments as they pass,
so the user can see where the project stands at a glance.

### Published results to confirm

When the user points to a paper, confirming it becomes a check in the plan:

- Get the paper (search arXiv or the web, or use the PDF the user attached) and
  pull out the exact numbers: table values, quoted fits, equation-level results. For
  figures, extract points (digitize carefully) and record the extraction uncertainty.
- Map their conventions onto ours before comparing: units, normalizations, sign
  conventions, parameter definitions, and which approximation they used. Report any
  mismatch you find instead of quietly adjusting.
- Record each target in `tests/published/` (a small data file or Python dict with citation,
  figure/table/equation number, their parameters in our dimensionless variables, and the
  tolerance justified by their stated precision or the digitization error), and write a
  test that compares against it.
- If our code disagrees with a published result beyond tolerance, that's a finding, not
  a failure to hide: check our side first (convergence, conventions), then tell the user
  exactly where and by how much we disagree.

## Before writing code: nondimensionalize and question the setup

1. **Dimensionless variables, always.** Pick natural scales (e.g. 1/Ω_c and v/Ω_c for a
   gyrating particle; the primaries' separation and orbital period for the restricted
   three-body problem; 1/κ for a soft-wall model) and write the equations in dimensionless
   form. The code works only in dimensionless variables; conversions to physical units live
   in one clearly labeled place (a `units.py` or a section of `model.py`) and are applied
   only for output. State the scales and the resulting dimensionless groups to the user.
2. **Count the real parameters.** After nondimensionalizing, identify the dimensionless
   groups the physics actually depends on, plus any scaling symmetries. If the user asks
   for a sweep over variables that are redundant (e.g. a 2D grid in κ and λ when the answer
   is κ·f(λ/κ⁴)), **push back before running it**: explain the reduction, propose sweeping
   only the independent group(s), and show how to recover the requested grid from the
   reduced one. Let the user decide; they may have a reason for the original grid.
3. **Question physical definitions.** Criteria that are easy to code but physically loose
   (an escape radius that doesn't guarantee escape, a cutoff that defines a "bound" state,
   an order parameter threshold) should be checked and flagged early, with a quantitative
   test of whether they mean what they claim.

## The core loop: every step stands on a verified step

Work in increments. Each increment is one new piece of physics or numerics, and it is not
"done" until it passes its checks. Never add two unverified things at once — if a check
fails you want exactly one suspect.

**Scaffold from one run upward.** The first increments always produce a tool that runs and
shows *one* case (one trajectory, one spectrum, one configuration) with numbers printed and
a simple plot. Only after that single-run tool is validated do you build loops, maps, or
sweeps on top of it — even when the user's end goal is a big map. The single-run tool stays
in the project permanently as the way to inspect any point of any later sweep.

For each increment:

1. **Restate the increment and its check in one or two lines** (from the approved plan,
   including any known or published results the user supplied). If you can't name a check,
   pick a smaller increment or a limiting case first.
2. **Implement**, defaulting to trusted library routines (see "Numerics").
3. **Validate** (details in Appendix A):
   - *Known result*: the first version of any new simulation must reproduce an analytic
     result or the output of a simpler, already-trusted model (exactly solvable limits,
     published numbers, the previous stage in the limit where the new term vanishes).
   - *Generalized solver vs. simpler solver*: when adding a more general solver (e.g. a
     relativistic pusher), run it in the regime where the simpler one applies (v ≪ c) and
     compare against the already-validated simpler solver, including the size of the
     expected correction (e.g. O(v²/c²)).
   - *Conserved quantities*: energy, momentum, angular momentum, Jacobi constant, charge,
     norm, Wronskian, constraint residuals… Report the relative drift as a number.
   - *Convergence*: vary resolution parameters (dt, grid N, tolerance, cutoff, basis size,
     samples) by factors of 2 (or 10 for tolerances), and report the observed order and the
     remaining-error estimate.
4. **Back-compatibility (regression)**: when a feature generalizes an existing model, the
   new code with the new ingredient switched off must reproduce the old results. Save
   reference outputs from verified stages (small `.npz` files in `tests/reference/`) and keep
   tests that compare against them.
5. **Document and commit** once the checks pass: add the stage's section to the progress
   notebook (see "The progress notebook: built at every stage"), execute the notebook top to
   bottom, check off the increment in `PLAN.md`, and commit code, tests, references and
   notebook together (see "Version control"). A stage is not finished until its notebook
   section exists and runs.
6. **Report** a compact validation summary (check → expected → obtained → status), then
   name the next increment *and its planned check*, inviting the user to suggest a
   different or additional check.

If a check fails, stop adding features and diagnose. Don't loosen a tolerance to make a test
pass without explaining why the new tolerance is justified (e.g. it matches a derived error
estimate).

## Version control

Every project is a git repository, created at the start (`git init`) with a `.gitignore`
for `__pycache__/`, `.ipynb_checkpoints/`, `.pytest_cache/`, and large or regenerable
outputs (big sweep results, or use Git LFS if the user wants them tracked). Small
reference files in `tests/reference/` and `tests/published/` *are* committed; they're part
of the validation record.

- **Commit along the way**: the plan, each validated increment, each new reference file,
  each notebook section. A commit message says what was added and how it was checked,
  e.g. `Add E×B drift stage; drift speed matches E/B to 1e-9, ε=0 matches stage-1 reference`.
- Keep commits to a working, tested state on the main branch. Exploratory or half-done
  work goes on a branch (`git switch -c try-spectral-basis`) and is merged once it passes.
- Tag milestones the paper depends on (`git tag stage2-validated`, `paper-v1-figures`).
- Record the git hash (and whether the tree was dirty) in the metadata of every saved
  result, so any figure can be traced to the exact code that made it.
- Don't push to a remote, rewrite history, or change the user's git config unless asked.
- If a folder the user already has is not a repository, ask before running `git init` in
  it; in a folder created for the project, just do it.

## Code written for human readers

The user reads this code to understand and trust it. Comment generously — more than you
would in production software.

- **Every module starts with a full header docstring** containing:
  - what the module does and where it sits in the overall program flow;
  - the equations being solved, in dimensionless form, with the nondimensionalization
    (scales and dimensionless groups) and the source (paper + equation number, or textbook);
  - the numerical method and *why* it was chosen over alternatives, with its known error
    behavior (e.g. "Boris: exact |v| conservation in pure B; gyrophase lag 2 arctan(Ω dt/2)");
  - interfaces: what the main functions take and return, array shapes;
  - validity limits and assumptions (non-relativistic, paraxial, r ≪ L…).
- **Describe the overall program flow** in one place: a `README.md` (or the header of
  `run_single.py`) with a short diagram-like list of which module calls which, from
  command-line input to printed output, and which stage of the project each file belongs to.
- **Section comments** inside functions for each logical step ("# 1. Build the grid in
  u = κz", "# 2. Assemble the tridiagonal Hamiltonian", …), and inline comments explaining
  *why*: why this tolerance, this boundary condition, this branch.
- **Descriptive names.** Prefer `escape_radius`, `larmor_radius`, `n_modes`,
  `dilaton_quartic_coeff`, `cyclotron_period` over `r_e`, `rl`, `nm`, `lam4`, `T`. Short
  symbols are acceptable only when they match an equation written in the docstring right
  above (e.g. `u`, `V(u)` in a function whose docstring defines them), and even then say
  what they are.
- **Explicit parameters**: a small dataclass for dimensionless parameters, no globals.
- **Metadata with results**: parameters, solver settings, code version (git hash), date.
- **Markdown files use LaTeX** for math: `$\Omega_c = qB/m$`, `$m_n^2 = 4\kappa^2(n+1)$`.
  This applies to the README, reports, validation summaries, and notebook markdown cells.

Default project layout (adapt names to the problem):

```
project/
├── README.md           # program flow, dimensionless variables, how to run/test (LaTeX math)
├── PLAN.md             # approved staged plan with checks; increments checked off as they pass
├── .gitignore
├── physics/            # the model: small, well-commented functions, one idea per module
│   ├── model.py        # dimensionless equations + unit conversions
│   ├── observables.py  # energies, spectra, conserved quantities, order parameters
│   └── solve.py        # wraps solvers; all numerical knobs exposed as arguments
├── tests/              # test_known_results / test_conservation / test_convergence / test_regression
│   ├── reference/      # saved outputs of verified stages
│   └── published/      # target values from papers, with citations and tolerances
├── run_single.py       # one parameter point → printed numbers + quick plot
├── sweep.py            # many points, parallel; calls the same function as run_single
├── results/
└── notebooks/
    ├── progress_summary.ipynb   # validation record, one section per stage, updated every stage
    └── paper_figures.ipynb      # publication figures (later)
```

- **One code path.** `sweep.py` calls exactly the function `run_single.py` calls, so any
  point from a sweep can be reproduced and inspected with
  `python run_single.py --param1 ... --param2 ...`. When outputs are several layers deep
  (sweep → fits → derived quantity), `run_single.py` can print each layer (`--verbose`).
- `run_single.py` prints labeled key numbers (dimensionless, and physical units if
  relevant), conserved-quantity drift, and has `--plot` for a quick look.
- Use `pytest`; tell the user to run it after every change.

## Numerics: trusted tools by default, faster tools when they pay off

**Default** to numpy/scipy and well-established packages: they're tested far more thoroughly
than anything written in a session. Prefer vectorized numpy over Python loops.

- ODE IVPs: `scipy.integrate.solve_ivp` (`DOP853`/`RK45` non-stiff; `Radau`/`BDF`/`LSODA`
  stiff; set and expose `rtol`/`atol`). A short symplectic integrator (leapfrog/Boris) is a
  justified exception for long-time Hamiltonian or Lorentz-force evolution — say why.
- BVPs: `solve_bvp`, or shooting with `solve_ivp` + `brentq`/`root`.
- Eigenproblems: `scipy.linalg.eigh`/`eigh_tridiagonal`/`eig_banded`, `sparse.linalg.eigsh`.
- Quadrature: `quad` (pass `points=` at singularities), `simpson` for sampled data.
- Special functions `scipy.special`; interpolation `CubicSpline`, `RegularGridInterpolator`;
  fits `curve_fit` with uncertainties; random numbers `np.random.default_rng(seed)` with the
  seed recorded, `SeedSequence.spawn` for workers.

**Scaffold to faster tools when the cost estimate says it's worth it.** After the trusted
version is validated, time one point. If the projected run (see parallelization) is long
enough to hurt — roughly more than tens of minutes on the machine at hand, or it will be
rerun many times — suggest a faster path, with an estimated speedup, as the *next increment*:

1. Cheapest first: vectorize, reduce Python overhead in the right-hand side, pick a better
   method (e.g. a spectral basis instead of finite differences), exploit the symmetry
   reductions found earlier.
2. JIT-compile the hot inner function with Numba (`@njit` right-hand side, event functions,
   matrix assembly) while keeping the scipy driver.
3. A fully compiled integrator or solver (Numba-compiled loop, `numbalsoda`, JAX, or a
   C/Fortran routine) when the scipy driver's own overhead dominates.

Each faster version is a new increment with the trusted version as its oracle: a test
compares fast vs. trusted on a set of points (including hard ones) to a stated tolerance,
and the trusted version stays in the code base as a reference (`method="reference"`).
Describe the tradeoff to the user and let them choose; don't make the jump silently.

## Parallelization: local machine first, cluster when it earns it

Start on whatever machine the work is already running on, and find out what it has
(`os.cpu_count()`, available RAM) before estimating anything. Read Appendix B before
writing `sweep.py`. Short version:

- Parallelize over **independent parameter points** with `ProcessPoolExecutor` or `joblib`,
  using roughly (number of cores − 1 or 2) workers, BLAS threads pinned to 1 per worker.
- Don't parallelize before the single-run version is validated, or when the whole job takes
  about a minute serially.
- Estimate cost first: time one point, × number of points, ÷ number of workers. Tell the
  user.
- **Flag a compute cluster** when the local estimate is more than several hours even after
  reasonable speedups, when memory per worker × workers approaches the machine's RAM, or
  when the sweep will be rerun many times. Batch-scheduler template in Appendix B.

## Plots

No title, no gridlines, large fonts for tick and axis labels, axis labels with units
(dimensionless labels like $t\,\Omega_c$ are fine). "Graph y vs. x" means y vertical,
x horizontal.

## The progress notebook: built at every stage

`notebooks/progress_summary.ipynb` is the project's validation record, the notebook the user
opens to see where things stand. Start it with the first validated stage and **extend it at
the end of every increment**, in the same commit as the code, never as a catch-up job at
the end. It contains:

- **Header**: project, a table mapping each section to its PLAN stage and to what it checks,
  and a setup cell that adds the repository root to `sys.path` (locate it by searching upward
  for the package directory, with an environment-variable override), imports from `physics/`,
  sets the plot style, and prints the git hash.
- **One section per validated stage**:
  - a markdown cell explaining the physics and numerics of the stage, with equations (LaTeX;
    display `$$...$$` is fine in notebooks);
  - code cells that **import** the model, solvers and oracles from `physics/`, and the seeded
    test cases from `tests/`, never re-implementing them, and that reproduce the stage's key
    validation as a figure: numerics against the known or published result, conserved-quantity
    drift, a convergence plot with the observed order;
  - printed numbers that match the validation table.
- **An interactive single-run explorer** (ipywidgets sliders calling the same function as
  `run_single.py`, with a static call above it so the executed notebook shows a figure) once a
  single-run tool exists.
- **A closing section** with the cumulative results table (check → expected → obtained),
  the physics learned so far, and the suggested next steps. Rewrite it at each stage.

Keep it cheap, well under a couple of minutes for "Restart & Run All": use the fast solver
path once it exists, and load stored results (`tests/reference/`, `results/`) for expensive
sweeps rather than recomputing them. Before committing, execute it
(`jupyter nbconvert --to notebook --execute --inplace notebooks/progress_summary.ipynb`) and look
at every figure, not only at whether it ran. Keep a slow test that executes a copy, so an
interface change that breaks the notebook is caught. Commit it with outputs, so it reads on
GitHub.

## The publication notebook

As soon as a result looks like it will appear in a paper, start or extend
`notebooks/paper_figures.ipynb` (see Appendix C). It imports from
`physics/`, loads saved results or recomputes cheap ones, and regenerates each figure and key
number with "Restart & Run All". Markdown cells use LaTeX.

## Communicating each step

End each increment with: what was added (one line); the validation table; the progress-notebook
section added (and that it ran top to bottom); the commit made; how to reproduce
one point (`python run_single.py ...`) and run tests (`pytest`); runtime / speedup /
parallelization notes if relevant; any pushback on the setup (redundant parameters,
loose definitions); and the proposed next increment with its planned check. Use LaTeX for
math in any markdown you write. Keep it short; the numbers carry the message.

---

## Appendix A: Validation playbook

Three families of checks: known results, conservation, convergence. Plus regression
(back-compatibility) as the project grows. Each check should produce a number compared
against a tolerance, written as a pytest test, and summarized for the user.

### 1. Known results

Pick the check *before* writing the code, propose it in the plan, and let the user swap in
or add known results they prefer. Checks the user names take priority. Ways to find one:

| Situation | Typical known result |
|---|---|
| New ODE/Hamiltonian system | Integrable limit: harmonic oscillator, Kepler orbit (period, eccentricity vector), free fall, pendulum small-amplitude period |
| Adding an interaction / coupling g | g → 0 reproduces the uncoupled solution; small g matches first-order perturbation theory (check the O(g) slope, not just g=0) |
| Finite temperature / density | T → 0 or μ → 0 reproduces the vacuum code / earlier paper |
| Eigenvalue / spectrum problems | Exactly solvable potentials (harmonic, infinite well, Coulomb, Pöschl–Teller); soft-wall AdS/QCD: linear Regge spectrum m²_n = 4κ²(n+1) for vector mesons in the simplest model |
| PDEs | Manufactured solutions (pick u, compute the source that makes it exact); separable/plane-wave solutions; steady states |
| Monte Carlo | Exactly solvable small systems (2×2 Ising by enumeration, 1D Ising, ideal gas); known critical exponents as a later check |
| N-body / collisions | Two-body reduces to Kepler; momentum/energy exchange for elastic collisions; known choreographies |
| New, more general solver (relativistic, full nonlinear, more fields) | Run it where the simpler validated solver applies (v ≪ c, small amplitude, fields decoupled) and compare; check the difference scales like the expected correction, e.g. O(v²/c²) |
| Faster reimplementation (Numba, compiled) | Agrees with the trusted scipy version on a fixed set of points, including chaotic/hard ones, to a stated tolerance |
| Anything already published | Reproduce a figure or table number from the reference paper before extending it (see "Published results" below) |

Compare quantitatively: relative error vs. tolerance consistent with the solver settings.
"The curve looks right" is a starting point, not a pass.

Toy-model comparison: when no analytic result exists, build the simplest version (fewer
dimensions, linearized, fewer fields) that the full code must reduce to, verify the toy
against analytics, then verify full-in-limit against toy.

### 1b. Published results

Store each target in `tests/published/<short_name>.py` (or `.json`/`.csv` for digitized data):

```python
# tests/published/kkss2006.py
CITATION = "Karch, Katz, Son, Stephanov, PRD 74, 015005 (2006), arXiv:hep-ph/0602229"
SOURCE = "vector-meson spectrum m_n^2 = 4 kappa^2 (n + 1)  (cite the exact eq. number from the paper)"
CONVENTIONS = "Their kappa^2 z^2 dilaton = ours with u = kappa z; masses in units of kappa"
PARAMS = {"quartic_coeff_hat": 0.0}          # in our dimensionless variables
TARGET = {"mass_sq_over_kappa_sq": [4, 8, 12, 16, 20, 24]}
RTOL = 1e-8                                  # exact result; limited by our numerics
```

- Digitized figure data: record the tool and method used, and set the tolerance from
  the digitization scatter (repeat the extraction to estimate it), not a guessed number.
- Numbers quoted to k significant figures get a tolerance of about half a unit in the
  last digit.
- The test's failure message should print ours vs. theirs and the citation.

### 2. Conserved quantities

List every quantity that should be conserved (or satisfy a known identity) and monitor it:

- Energy, linear and angular momentum, center-of-mass motion (N-body, rigid bodies)
- Norm / unitarity (quantum time evolution), probability sums (Markov chains)
- Charge, particle number, baryon number
- Wronskian or first integrals of ODEs; constraint equations in GR / holography
  (e.g. the Hamiltonian constraint of Einstein's equations as a residual)
- Symmetries: results invariant under rotation/translation of initial data, under the
  model's discrete symmetries, time-reversal (integrate forward then back)

Report `max_t |Q(t) − Q(0)| / |Q(0)|` (or absolute if Q(0)≈0). Know what to expect:
non-symplectic adaptive methods show secular drift ∝ t at the level of rtol; symplectic
methods show bounded oscillation. A drift much larger than the tolerance is a bug; a drift
behaving differently than the method predicts is a warning.

### 3. Convergence

- Identify every resolution parameter h (dt, dx, N grid points, N basis states, cutoff
  z_max or r_max, rtol, number of samples).
- Compute the observable at h, h/2, h/4 (three levels minimum).
- Observed order: p ≈ log2(|A(h) − A(h/2)| / |A(h/2) − A(h/4)|). Compare to the method's
  nominal order. Richardson estimate of the remaining error: |A(h/2) − A(h/4)| / (2^p − 1).
- For adaptive solvers, vary rtol/atol by factors of 10 and show the answer changes by
  about that much until it plateaus.
- Cutoffs (IR/UV, box size, z_max near a horizon or boundary): vary them and show
  insensitivity. Asymptotic boundary conditions deserve special care — check that the
  result does not depend on where the asymptotic expansion is matched.
- Monte Carlo: error ∝ N^(−1/2); estimate errors with blocking/bootstrap, check
  autocorrelation times.
- Show a log–log convergence plot when it's informative; otherwise a small table.

Choose production resolution with margin: the error estimate should be well below the
effects the paper will claim.

### 4. Regression / back-compatibility

- After a stage passes, save a small reference output (`np.savez` with parameters) to
  `tests/reference/stage_name.npz`.
- Tests load it and compare the current code at those parameters with `np.allclose` at a
  stated `rtol`.
- When generalizing (adding μ, a new field, a new potential term), add a test that the new
  code with the new ingredient switched off matches the old reference.
- If a reference must change (a bug fix), say so explicitly to the user, explain why, and
  regenerate it deliberately — never silently.

### Validation summary format

```
Check                                  Expected        Obtained        Status
-------------------------------------  --------------  --------------  ------
Kepler period (e=0.5)                  6.28319         6.28318         PASS  rel 1.6e-6
Energy drift over 100 orbits           < 1e-8          3.2e-10         PASS
Convergence order in dt (leapfrog)     2               1.99            PASS
μ=0 limit vs. stage-2 reference        allclose 1e-8   max rel 4e-11   PASS
```

## Appendix B: Parallelization and clusters

### Local: the workstation or laptop at hand

The usual pattern is independent parameter points. Keep the per-point function pure
(inputs → result dict), importable from `physics/`, and identical to what `run_single.py` calls.
Check the available core count and memory before choosing a worker count rather than
assuming one.

```python
# sweep.py  (sketch)
import os
# One BLAS thread per worker; otherwise N processes × many threads fight for cores.
# Must be set before numpy is imported.
for v in ("OMP_NUM_THREADS", "OPENBLAS_NUM_THREADS", "MKL_NUM_THREADS",
          "VECLIB_MAXIMUM_THREADS", "NUMEXPR_NUM_THREADS"):
    os.environ.setdefault(v, "1")

import itertools, json, time
import numpy as np
from concurrent.futures import ProcessPoolExecutor, as_completed
from physics.solve import run_point          # same function run_single.py uses

def main():
    grid = [dict(T=T, mu=mu) for T, mu in itertools.product(
        np.linspace(0.10, 0.20, 41), np.linspace(0.0, 0.4, 21))]
    n_cpu = os.cpu_count() or 1
    n_workers = max(1, n_cpu - 2)              # leave a couple of cores for the OS/UI
    results = []
    with ProcessPoolExecutor(max_workers=n_workers) as ex:
        futures = {ex.submit(run_point, p): p for p in grid}
        for f in as_completed(futures):
            p = futures[f]
            try:
                results.append({**p, **f.result()})
            except Exception as e:           # one bad point must not kill the sweep
                results.append({**p, "error": repr(e)})
    # save results + metadata (params grid, solver settings, git hash, date)

if __name__ == "__main__":                   # required with the 'spawn' start method
    main()
```

Notes:
- With the `spawn` start method (the default on macOS and Windows, and increasingly
  elsewhere) the `if __name__ == "__main__":` guard is required, and worker functions must
  be importable (not lambdas or notebook-defined functions). In notebooks, put the worker in
  a `.py` module.
- `joblib.Parallel(n_jobs=-2)(delayed(run_point)(p) for p in grid)` is an equally good
  choice and handles BLAS thread limits via `threadpoolctl`.
- Chunk very cheap points (`ex.map(..., chunksize=k)`) so overhead doesn't dominate.
- Save incrementally (e.g. one file per point, or append every N) for long sweeps so a
  crash doesn't lose everything, and so a rerun can skip completed points.
- Numba-compiled functions work fine in worker processes; use `cache=True` so each worker
  doesn't recompile, and time the compiled version before re-estimating the sweep cost.
- A single big linear-algebra problem is the opposite case: one process, let the BLAS use
  all threads.
- A GPU is rarely worth it for scipy-style work; mention it only if the workload is large
  dense array math and a library like JAX, PyTorch or CuPy clearly fits.

Before running: time one point, report `t_point × N_points / n_workers` to the user.

### When to suggest a compute cluster

Suggest it, with the estimate, when any of these hold:
- the local wall-time estimate is more than roughly several hours (or running overnight is
  inconvenient), and the work splits into many independent jobs;
- memory per worker × number of workers would approach the machine's RAM;
- the sweep will be repeated many times (e.g. scanning model parameters, bootstrap
  ensembles, many random seeds);
- a single job needs MPI-scale parallelism beyond one node.

Otherwise the local machine is simpler — no queue, no file transfer.

### Job-array pattern

If the user has access to a batch scheduler, use a job array: the array index selects a
chunk of the parameter grid; each task calls the same `run_point` and writes its own output
file. A tiny `collect.py` merges them. Before submitting a big array, submit a short test
with two tasks and compare its output to `run_single.py` locally for the same points.

Keep queue/partition/account/module names as placeholders the user fills in once; ask if
unknown, and adapt the directives to whichever scheduler they use (SLURM, PBS/Torque, LSF,
SGE, or a cloud batch service — the structure is the same, only the directive syntax and
the environment variable holding the array index change).

### Example job-array script (SLURM syntax; adapt to your scheduler)

```bash
#!/bin/bash
# Batch job array for a parameter sweep.
# Each array task runs a chunk of the parameter grid through the SAME run_point()
# function used by run_single.py locally, and writes results/chunk_<ID>.npz.
# Submit a smoke test first:   sbatch --array=0-1 job_array.sh
# Then the full sweep:          sbatch job_array.sh
#
#SBATCH --job-name=sweep
#SBATCH --partition=PARTITION_NAME      # fill in for your cluster
#SBATCH --account=ACCOUNT_NAME          # fill in if required
#SBATCH --array=0-99                    # number of chunks - 1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1               # raise if run_point itself is multithreaded
#SBATCH --mem-per-cpu=2G
#SBATCH --time=02:00:00
#SBATCH --output=logs/%x_%A_%a.out

set -euo pipefail
mkdir -p logs results

# module load python/3.11              # adjust to your cluster
# source ~/venvs/physics/bin/activate

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OPENBLAS_NUM_THREADS=$SLURM_CPUS_PER_TASK
export MKL_NUM_THREADS=$SLURM_CPUS_PER_TASK

# sweep.py should accept --chunk i --nchunks N and process only its slice of the grid.
python sweep.py --chunk "$SLURM_ARRAY_TASK_ID" --nchunks "$SLURM_ARRAY_TASK_COUNT" \
                --out "results/chunk_${SLURM_ARRAY_TASK_ID}.npz"
```

## Appendix C: Publication notebook

`notebooks/paper_figures.ipynb` is the reproducibility record for the paper: anyone with
the repo can "Restart & Run All" and get every figure and key number.

### Structure

1. **Header cell (markdown)**: project, date, and a table mapping notebook sections to
   paper items ("Fig. 2 — spectrum vs T", "Table I — critical temperatures", "Eq. (14) — fit value").
2. **Setup cell**: imports from `physics/` (never paste model code into the notebook),
   plotting style (no titles, no gridlines, large tick/axis-label fonts), paths, and a flag
   `RECOMPUTE = False`.
3. **Validation cell**: a short run of the key checks (or `!pytest -q`) so the notebook
   demonstrates the code is still validated.
4. **One section per figure/number**: brief markdown on what it shows and the parameters;
   load stored results (or recompute if cheap or `RECOMPUTE`); make the figure; save it to
   `figures/fig_name.pdf`; print the key numbers the text will quote.
5. **Environment cell at the end**: numpy/scipy/python versions and git hash.

### Practice

- Add a section when a result becomes a likely paper item, not at the end of the project.
- Expensive sweeps run from `sweep.py` (locally or on a cluster); the notebook only loads
  their saved output, which carries its parameters and code version.
- Numbers quoted in the paper should be printed by the notebook, with uncertainties
  from the convergence estimates where relevant.
- If a figure cell takes more than a minute or two, it belongs in a script whose output
  the notebook loads.
- After adding a section, run the whole notebook top to bottom (e.g.
  `jupyter nbconvert --to notebook --execute`) to confirm it runs cleanly.
