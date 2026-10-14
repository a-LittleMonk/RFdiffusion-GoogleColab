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

Cell 4 now checks itself and raises if NumPy got pulled up to 2.x, if the CUDA
11 libraries are missing, if DGL is the wrong version, or if torch cannot see the
GPU. Cell 6 raises if RFdiffusion still does not import or a weights file is
truncated. Both failures are terminal — see *If something fails* below, and do
not re-run the cell in place.

## Run all

Two things need you; everything else, including saving back to Drive, is
unattended.

1. **Cell 2 waits for a Google Drive authorisation popup.** Run all will sit
   there until you click through it.
2. **Cell 7's fields must already say what you want built.** Its defaults make a
   100-residue monomer and ignore whatever is in Drive — a file sitting in the
   inputs folder is not picked up on its own, you have to put its name in
   `input_pdb`.

Set those up and the rest runs through. Because the setup cells raise on a bad
environment, a Run all stops at the real cause rather than cascading into
misleading errors further down, and cell 7 refuses to start at all unless cell 6
finished.

The first design run is also a few minutes slower than later ones — see below.

**Run cell 9 before closing the tab.** Colab deletes `/content` when the runtime
ends, and the designs live there until you copy them to Drive.

**The first design run is a few minutes slower than later ones.** RFdiffusion
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
| `run_mode` | `Production`, `Export check`, `Fast debug`, or `Smoke test` — see below |
| `name` | Output prefix for the files |
| `design_type` | `monomer`, `binder`, `motif scaffolding`, or `partial diffusion` |
| `contigs` | What to build — see below |
| `input_pdb` | Target structure. Tried in this order: a file in your Drive `inputs` folder (`tgfb1_dimer.pdb`), a full path, or a 4-letter PDB code like `1D8R`, which is downloaded. Blank for a monomer. |
| `hotspots` | Residues the binder should engage, e.g. `A59,A83,A91` |
| `iterations` | Diffusion steps, 50 is a sensible start |
| `num_designs` | How many to generate |

### Run modes

| Mode | What it does | Time | Use it when |
|---|---|---|---|
| `Production` | Runs exactly what the fields say | hours | Real designs |
| `Export check` | Shrinks every contig segment to ~12 residues, 2 steps, then verifies the PDB and copies it to Drive | ~1 min | "Will I actually get a file out of this?" |
| `Fast debug` | Keeps your contigs and hotspots at full size; 5 steps, 1 design | a few min | Validating the real contig string and hotspots |
| `Smoke test` | Ignores every field; 20-residue monomer | <1 min | Is the install broken, or my settings? |

None of the three debug modes produce usable designs — 2–5 diffusion steps
cannot fold anything. They answer "does this run at all", never "is this design
good", and each prints a banner saying so.

**Why `Export check` is the fast one.** Runtime is dominated by residue count,
not step count: attention is quadratic in the number of residues, so a 250-residue
complex costs far more per step than a 25-residue one. `Fast debug` cuts the steps
10x but keeps the size, so a 3-hour run still takes minutes. `Export check` cuts
both, which is what gets it to about a minute.

It keeps the *shape* of your contig while shrinking it: target slices keep their
start residue and take the first 12 (so the numbering stays valid against your
real PDB), designed ranges collapse to a fixed 12, and chain breaks are untouched.
It prints the before and after so you can see what it ran. Two things to know:

- **Hotspots are dropped.** Shrinking the target moves those residues out of
  range. Use `Fast debug` to validate hotspots.
- **A bare chain id (`A`) cannot be shrunk**, since it means "the whole chain"
  and there is no way to know where it ends without guessing. The banner says so,
  and the check will take longer than a minute.

Output goes to `<name>_exportcheck_*`, so your real results are never touched,
and the check reads the PDB back and reports its atom and residue count before
copying it to Drive. If all of that passes, the only thing left untested is
whether the designs are any good.

Every run now prints its elapsed time, and a production run of more than one
design also prints the per-design average — useful for sizing a batch.

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
