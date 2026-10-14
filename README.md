# RFdiffusion on Colab

A Colab notebook that runs RFdiffusion on current Colab. Clean template — no
outputs, no run history, parameters at their defaults.

## Why this exists

Colab ships Python 3.13. RFdiffusion needs 3.10, and the stock ColabDesign
diffusion notebook fails on current Colab for that reason. This notebook
installs Python 3.10 into a separate environment at `/content/rfenv` and runs
inference through it as a subprocess, leaving Colab's own Python alone.

## Running it

Set the runtime to a GPU, then run cells 1–9 in order.

| Cell | | Time |
|---|---|---|
| 1 | Check the GPU | — |
| 2 | Mount your Google Drive | — |
| 3 | Install Python 3.10 | ~1 min |
| 4 | Build the environment | ~6 min, ~3 GB |
| 5 | Download RFdiffusion and model weights | ~1 GB |
| 6 | Compile SE3Transformer | ~3 min |
| 7 | Generate designs | depends |
| 8 | View a design | — |
| 9 | Save results to Drive | — |

**Stop and read cell 4's output before continuing.** It should show
`numpy: 1.26.4`, `dgl: 1.1.x`, and `torch<->numpy: 1.0`. If numpy has been
pulled up to 2.x, something later installed over it and DGL will crash.

**Run cell 9 before closing the tab.** Colab deletes `/content` when the runtime
ends, and the designs live there until you copy them to Drive.

The first design run is a few minutes slower than later ones. RFdiffusion
computes its IGSO3 noise schedule on first use and caches it. The upstream
notebooks download a prebuilt cache from `files.ipd.uw.edu/krypton/schedules.zip`,
but that file was removed from the server and now 404s — the download fails
silently and you would never notice except for the extra minutes. That step is
removed here deliberately; don't add it back.

## Working from your Google Drive (cell 2)

Cell 2 mounts your Drive and creates `My Drive/RFdiffusion/inputs` and
`My Drive/RFdiffusion/outputs`. Put any target structure you want to design
against into the `inputs` folder — the cell lists what it finds there, so you
can copy a name straight into `input_pdb`.

Nothing else depends on it. Skip cell 2 and the notebook still works with
uploaded files and PDB codes; the design cell only looks in Drive when the name
you gave is not already a file on disk.

Cell 9 writes the designs back to `My Drive/RFdiffusion/outputs` unless you
name a different folder.

## Setting up a run (cell 7)

| Field | What it does |
|---|---|
| `run_mode` | `Production`, `Fast debug`, or `Smoke test` — see below |
| `name` | Output prefix for the files |
| `design_type` | `monomer`, `binder`, `motif scaffolding`, or `partial diffusion` |
| `contigs` | What to build — see below |
| `input_pdb` | Target structure. Tried in this order: a file in your Drive `inputs` folder (`tgfb1_dimer.pdb`), a full path, or a 4-letter PDB code like `1D8R`, which is downloaded. Blank for a monomer. |
| `hotspots` | Residues the binder should engage, e.g. `A59,A83,A91` |
| `iterations` | Diffusion steps, 50 is a sensible start |
| `num_designs` | How many to generate |

### Run modes

| Mode | What it does | Use it when |
|---|---|---|
| `Production` | Runs exactly what the fields say | Real designs |
| `Fast debug` | Keeps your target, contigs and hotspots; forces 5 diffusion steps, 1 design, no trajectory files | Checking a new contig string or target before paying for a full run |
| `Smoke test` | Ignores every field; diffuses a 20-residue monomer | Deciding whether a failure is your settings or the install |

Both debug modes print a banner and the backbones they produce are noise — five
diffusion steps is nowhere near enough to fold anything. They exist to answer
"does this run at all", not "is this design good".

`Smoke test` is the one to reach for when a run fails and the traceback is
unhelpful. It touches nothing but the environment, the weights and the output
path, so if it passes the install is sound and the fault is in your settings.

### Contigs

| Goal | `contigs` | `input_pdb` |
|---|---|---|
| Monomer, fixed length 100 | `100-100` | blank |
| Monomer, sampled 70–100 | `70-100` | blank |
| Binder to chain A | `A1-150/0 70-100` | target |
| Scaffold a motif | `40/A163-181/40` | target |
| Keep chain A, rebuild the rest | `A` | target |

`/0` marks a chain break.

Always start with 50 iterations and 1 design to confirm the pipeline runs before
committing time to a real batch.

**Give `hotspots` for binder design.** Without them the binder will dock
wherever it likes on the target, which is rarely the site you care about.

## Worked example — a binder to TGF-β1

Put the TGF-β1 dimer PDB (3KFD) in `My Drive/RFdiffusion/inputs`, run cell 2,
then:

```
name        = tgfb1_binder
design_type = binder
contigs     = A1-112/0 B1-112/0 70-100
input_pdb   = tgfb1_dimer_3kfd.pdb
hotspots    = A31,A34,A91,A94
```

That presents both chains of the dimer as the target and builds a 70–100 residue
binder against the interface hotspots on chain A.
