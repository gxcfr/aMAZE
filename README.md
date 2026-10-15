# A-MAZE · MAZE-X

**An Intelligent Maze Solving & AI Strategy Arena** — an individual university AI project built in Python. Five implemented techniques share a validated maze environment while solving distinct tasks. No external API, database, cloud service, or separate frontend is required.

## Install and run on Windows

Install **Python 3.11 or 3.12**, then open PowerShell in this folder:

```powershell
python -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
.\.venv\Scripts\python -m streamlit run app.py
```

Open the Local URL printed by Streamlit (normally `http://localhost:8501`). Alternatively, run `setup.bat` once and then `launch.bat`. These scripts use `python` on your PATH. If needed, create the environment with `py -3.11 -m venv .venv` instead. Activation is unnecessary, avoiding PowerShell execution-policy issues.

Internet access is needed to install dependencies. The application and all training/experiments then run locally. If a port is occupied, add `--server.port 8502`. Stop the server with Ctrl+C. Use the project folder as your working directory so `.streamlit/config.toml` applies.

## Tests and experiments

```powershell
.\.venv\Scripts\python -m pytest -q
.\.venv\Scripts\python run_experiments.py --quick
# Larger bounded experiment:
.\.venv\Scripts\python run_experiments.py --output results_full
```

The CLI writes real CSV observations, a Markdown report, maze snapshots and the GA configuration into `results/` by default. `--quick` uses 16 training mazes, 16 sampled states per maze, 150 maximum epochs, five policy seeds (10000–10004), and a smaller GA. Timing depends on the machine. The GUI can export CSV/report data and individual fuzzy/GA observations. See `VALIDATION.md` for the final checks performed during development.

A measured quick-run bundle is included in [validation_results/REPORT.md](validation_results/REPORT.md), with raw CSV files and complete maze/configuration snapshots beside it.

## Using the workspace

1. **Overview:** inspect the current environment, symbols and task descriptions.
2. **Maze Explorer:** choose A* or unit-cost BFS, Start/Step, inspect g/f scores and parents. Try **Weighted detour**, **Unreachable goal**, and **Already at goal**.
3. **Adversarial Arena:** choose 1–6 individual plies and a turn budget. Both the escape agent and enemy call Minimax. Inspect a decision's evaluated search tree, bounds and skipped child roots.
4. **Neural Network Lab:** generate labeled states and train. Inspect train/test MAE, the mean-prediction baseline, loss curve and held-out maze groups. Run Neural Agent or the Manhattan movement baseline. Optional save/load uses only `models/neural.joblib`.
5. **Fuzzy Logic Lab:** run the fuzzy agent; inspect a legal candidate or manual inputs. Membership graphs, fired rules and aggregated/defuzzified output come from the actual inference. A selectable stationary enemy is distinct from the moving adversarial arena.
6. **Genetic Algorithm Lab:** evolve binary maze layouts; inspect measured generation fitness, feasibility and initial-versus-evolved structure. Use the evolved layout as the shared maze if desired.
7. **Algorithm Comparison:** run A*/BFS repetitions, depth experiments, and policies across identical seeded static mazes. Train the network first to include it. Success-rate intervals show the uncertainty of small samples.
8. **Viva & Documentation:** abstract, problem, objectives, architecture, formulas, complexity, assumptions, alternatives, limitations and 45 concise viva answers.

**Playback semantics:** Start computes one bounded run and plays its recorded events; Pause/Resume affect playback, and Step advances one recorded event. Reset clears that run. Changing generation settings or regenerating invalidates maze-dependent results. Changing algorithm/depth settings clears the related recording. The displayed computation time excludes playback. Model training runs only on the training button and persists in session state; ordinary Streamlit reruns do not refit it. No simulated metric or performance claim is prefilled.

## Representation and movement

Coordinates are `(row, col)`. The immutable grid uses **0 = passage**, **1 = wall**. Start, goal, hazards, agent and enemy are separate overlays; they never corrupt the shared terrain. Maze snapshots include every overlay and cost, with a content-derived identity in experiments.

Movement is orthogonal. Weighted A* pays **1 + terrain surcharge + hazard surcharge** when entering a cell; hazards default to +3. Costs must be finite and non-negative. The API supports arbitrary non-negative terrain surcharges; the GUI generates hazard costs. BFS comparisons remove all surcharges for **both** methods. Hazards remain visible but do not slow individual turns in the adversarial game. In static policy episodes hazards add route cost, and an optional stationary enemy captures on entry.

Generated dimensions are 5–31, with iterative backtracking, random corridors, and open fields. Random corridors sample walls and carve a random monotone route; backtracking creates passages with a connector for even dimensions. Generation guarantees reachability and may lower the requested random wall density when carving. Five fixed fixtures are independent of RNG versions. Direct Maze objects support up to 61×61, but UI bounds keep simulations practical.

## Algorithms and honest interpretation

- **A***: hand-written priority queue, g/f maps, improved parents, stale-entry skipping and closed set. `f=g+h`; Manhattan h is admissible because every solution requires at least `|dr|+|dc|` orthogonal moves costing at least one. The triangle inequality also makes it consistent. Walls and surcharges cannot lower this bound. Path length counts moves; unique expanded cells include the goal. Start=goal succeeds with zero moves. Queue exhaustion returns an empty path, infinite cost and `unreachable`.
- **Minimax + alpha-beta**: hand-written alternating maximization/minimization, legal destination moves, forced passes only when isolated, maze-aware horizon evaluation and terminal rewards. Capture has priority over escape. Depth counts individual turns. `pruned_nodes` means **skipped direct child roots**, an honest lower bound on unvisited descendants. See [adversarial details](docs/adversarial.md).
- **Neural network**: real StandardScaler → 13–32–16–1 MLPRegressor with synthetic BFS-distance/risk targets, 25% held-out **maze groups**, train-only scaling and MAE against a training-mean DummyRegressor. Runtime predictions rank legal candidates without teacher distance lookahead. A distinct Manhattan movement baseline is evaluated in episodes. Synthetic labels/local features limit generalization; a fitted model need not beat the baseline. Convergence warnings are displayed. See [neural details](docs/neural.md).
- **Fuzzy logic**: complete nine-rule NumPy Mamdani inference, triangular/shoulder memberships, min AND, clipped consequents, max aggregation and 501-sample numerical centroid. Candidate risk affects real move scores. No optional `scikit-fuzzy` dependency is needed. Risk is a rule score, not a probability. See [fuzzy details](docs/fuzzy.md).
- **Genetic algorithm**: hand-written population, tournament selection, crossover, mutation and copied elitism; binary interior walls with fixed endpoints; deterministic A* reachability/path fitness. Unreachable candidates receive a large penalty. History includes generation zero, population mean, current best and best-so-far. See [genetic details](docs/genetic.md).

Neural, fuzzy and greedy movement use the **same explicit 0.25 × previous visits penalty** to discourage short loops, and stop at a fixed move limit. This controller aid is not learned by the neural network or inferred by the fuzzy system. Neither policy is complete or optimal. Timeout is a failure in success-rate calculations. Arena tree nodes, pathfinding cell expansions, regression error, risk and genetic fitness are different measures.

## Architecture

```text
app.py                    Streamlit pages, controls, session ownership
maze_x/
  maze.py                 Immutable model, neighbors/costs, generation, samples
  search.py               Hand-written weighted A* and unit-cost BFS
  adversarial.py          Game states, Minimax, alpha-beta, arena episodes
  neural.py               Generated data, group holdout, MLP, local persistence
  fuzzy.py                Memberships, rules, Mamdani inference, move scoring
  genetic.py              Binary chromosomes, fitness, selection/evolution
  agents.py               Bounded policy episodes and controller revisit aid
  experiments.py          Task-specific experiment tables and CSV/report export
  serialization.py        Portable maze snapshots and reproducible identities
  rendering.py            Plotly maze visualization
  ui.py                   Styling and bounded recorded-event playback
run_experiments.py        Reproducible command-line experiment runner
tests/                    Algorithm, edge-case, experiment and Streamlit tests
docs/                     Exact implementation notes and academic documentation
```

Algorithm modules do not import Streamlit. Frozen Maze objects make keyed goal-distance caching safe. Neural fitting is explicit and session-held; genetic fitness caching is scoped to one run/configuration. No background training, shared mutable maze, unbounded simulation loop or mandatory model artifact is used. User inputs are bounded in both the UI and algorithm APIs.

## Dependencies and troubleshooting

Core: Streamlit, Plotly, NumPy, pandas, scikit-learn and joblib. Tests: pytest. The project does not need NetworkX, Pygame, Matplotlib, scikit-fuzzy, Node.js or a GPU. `requirements.txt` installs the directly validated versions from `requirements-tested.txt` for predictable setup. `requirements-lock.txt` additionally records the full dependency snapshot from the isolated validation environment.

A missing Plotly/Streamlit/scikit-learn import means dependencies were installed into a different Python environment: use `.venv\Scripts\python -m pip install -r requirements.txt`. An absent saved model is a normal untrained state. Model files are local and optional; only load files created by this project. If training reaches its iteration limit, increase epochs and compare the actual holdout errors rather than assuming improvement.

## Development milestones

1. Foundation: shared model, seeded generation, rendering, A*/BFS; initial 23 tests and Streamlit startup/widget check passed.
2. Adversarial search: real alternating decisions and tree metrics; 44 combined foundation/adversarial tests passed.
3. Neural network: data generation, real fitting, grouped evaluation and movement; 16 neural tests passed.
4. Fuzzy logic: full inference, plots/rules and movement; 11 fuzzy tests passed.
5. Genetic algorithm: evolution, fitness history and best-candidate integration; final validation recorded in `VALIDATION.md`.
6. Integration/QA: experiments, exports, Windows launch scripts, academic documentation and end-to-end widget tests; final validation recorded in `VALIDATION.md`.

The app remains runnable throughout development. The academic reference is [docs/ACADEMIC.md](docs/ACADEMIC.md); its descriptions are audited against the implemented modules.
#   a M A Z E  
 