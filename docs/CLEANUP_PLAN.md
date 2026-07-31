# Cleanup plan — `research_UPWINS_Microscene` → `upwins-microscene-preprocessing`

> **Proposal for owner review. Nothing has been built.** This document is the plan
> only; no new repo, branch, or commit exists in `upwins-microscene-preprocessing`.
> Written 2026-07-31 against `research_UPWINS_Microscene` @ `6032500` (`main`).
>
> **Updated 2026-07-31 with the owner's answers.** Decisions **1–4** in §6 are
> now **answered**, and §6 is renumbered to match the owner's numbering (1 nothing
> ships, 2 vectorize only and flag, 3 `.img` plus the pair convention, 4 delete the
> WIP section). Decision 2 resolved to *vectorize only*, defined precisely in **§6d**
> so it is not read more broadly later. The `LICENSE` copyright line is confirmed
> flagged-but-unchanged (§6b); the six items in §6c stand on their stated defaults.

**Goal:** produce a third client-ready deliverable, `upwins-microscene-preprocessing`,
that sits alongside `upwins-hsi-preprocessing` and `upwins-veg-classifier` and looks,
installs, and reads exactly like them.

**Reference material reviewed:** `upwins-hsi-preprocessing` (`AUDIT_HANDOFF.md`,
`docs/audit_plan_alternate.md`, `docs/temp_audit_plan_cross_check.md`),
`upwins-veg-classifier` (`docs/temp_auditplan.md`), and the two research repos they
were cut from (`research_species_mapping`, `research_NN_Hyperspectral_Vegetation`).

**Verdict up front.** The science here is sound and self-contained — a
white-reference/dark-frame reflectance conversion, ROI collection, and ROI
separability analysis. The packaging is roughly where `research_species_mapping` was
before its cleanup, and in two respects worse: there is no config file at all (paths
are literals edited in-notebook, with red-HTML banners telling the user to edit
them), and about 60 % of the repo by size is team scratch space, stale PDF exports,
and committed data. There is also **one real data-loss bug** (§B3) that the companion
repo already fixed and this repo still carries. None of this is hard to fix; it is
the same list of moves the other two repos already made. The one science question
(§B4) has been answered — *vectorize only, change no values* (§6a, decision 2).

---

## 0. Context for a fresh implementation session

**Read this section first.** It holds everything a new session needs that is not
recoverable from the repos themselves. Everything below §0 is analysis; this is the
operating manual.

### What exists, and where

| Repo | Commit at time of writing | Role |
|---|---|---|
| `upwins/research_UPWINS_Microscene` | `6032500` (`main`) | **Source.** All content to be ported. This plan lives on branch `claude/upwins-microscene-cleanup-plan-7gttbi` at `docs/CLEANUP_PLAN.md`. |
| `upwins/upwins-microscene-preprocessing` | **empty — no commits on any branch** | **Target.** Build here. |
| `upwins/upwins-hsi-preprocessing` | `ca90b2f` | **Primary template.** Closest analogue: same viewer, same ROI pickles, same notebook shape. Copy its `pyproject.toml`, `.gitignore`, `.devcontainer/`, `docs/data.md`, `docs/recording_runbook.md`, the `REPO_ROOT` block, and `src/hsiViewer/`. |
| `upwins/upwins-veg-classifier` | `6abfce1` | **Secondary template**, and the consumer of this repo's outputs. |
| `upwins/research_species_mapping`, `upwins/research_NN_Hyperspectral_Vegetation` | — | Provenance only — the research repos the two templates were cut from. Nothing to port. |

The four audit documents worth reading before touching anything:
`upwins-hsi-preprocessing/AUDIT_HANDOFF.md` (the closest model for this work),
`.../docs/audit_plan_alternate.md`, `.../docs/temp_audit_plan_cross_check.md`, and
`upwins-veg-classifier/docs/temp_auditplan.md`.

### Environment constraints

- **No microscene imagery, no dark frames, no ROI pickles**, and **no display**. The
  notebooks cannot be executed end to end here. Verification is static plus
  unit-level execution of extracted logic — exactly the position both companions'
  audits were in, and their "what was verified / what was not" tables are the model
  for how to report it.
- `PyQt5` imports fine headless with `QT_QPA_PLATFORM=offscreen`; only the live
  viewer windows need a real display.
- `spectral` may not build from source in a bare environment. If it will not install,
  modules that import it can still be checked with `sys.modules['spectral']` stubbed
  (the technique `upwins-veg-classifier`'s audit used for `batch_predict`).
- `numpy`, `scikit-learn`, `matplotlib`, `pandas`, `PyYAML` install cleanly.

### Editing notebooks safely — read before touching any `.ipynb`

The companions' protocol applies, **with one microscene-specific trap they did not
have.** Their notebooks all round-trip through
`json.dump(nb, f, indent=1, ensure_ascii=True)` + a trailing `\n`. **These do not
agree with each other:**

| Notebook | Round-trips with |
|---|---|
| `1 UPWINS Mircoscene preprocesing.ipynb` (and its ` copy`) | `indent=1, ensure_ascii=True` |
| `2 UPWINS Mircoscene Creating ROIs.ipynb` | `indent=1, **ensure_ascii=False**` |
| `3 UPWINS Microscene ROI visualization and Analysis.ipynb` | `indent=1, **ensure_ascii=False**` |

Using the wrong setting rewrites every non-ASCII character in the file and turns a
three-line change into a whole-file diff. **Detect per file before writing:**

```python
import json
orig = open(p, encoding='utf-8').read()
nb = json.loads(orig)
ENSURE_ASCII = next(ea for ea in (True, False)
                    if json.dumps(nb, indent=1, ensure_ascii=ea) + '\n' == orig)
```

Then assert the round-trip again after every write. Two further rules, both learned
the hard way in the companion sessions:

- Store a cell's `source` as a **list of lines each ending in `\n`** (last line bare
  if it has none), never a single string. Both are legal nbformat; the single string
  reshapes the file and produces a whole-cell diff. All cells in all three notebooks
  currently use the list form.
- **Assert on the exact expected text before deleting or replacing cell source**, and
  write the edit script so it saves only after every assertion passes. A partial
  write to a notebook is painful to unpick.

All three notebooks are `nbformat` 4.2.

### Locating cells

Cell numbers throughout this document are **0-based indices into `nb['cells']`,
counting markdown cells** — not Jupyter's displayed numbering. They are accurate
against `6032500`, but **locate by content anyway**, because the phases insert and
delete cells:

```python
hits = [i for i, c in enumerate(nb['cells'])
        if 'ANCHOR TEXT' in ''.join(c['source'])]
```

| Notebook | Purpose | Anchor text | Cell |
|---|---|---|---|
| nb 1 | imports + the two hardcoded dirs | `dir_dark = 'data/100101_Allied` | 3 |
| nb 1 | the `file_names` class (§B1) | `class file_names:` | 4 |
| nb 1 | filename selection (§B2) | `fname_hres_jpg = fnames.jpg[0]` | 8 |
| nb 1 | crop bounds (§A1) | `crop_rows = [1,1999]` | 11 |
| nb 1 | white-reference rows (§A1, §B5) | `white_ref_rows = [0,530]` | 14 |
| nb 1 | viewer on the **uncropped** cube (§B6) | `hva.viewer(im.Arr, im.wl)` | 15 |
| nb 1 | white-reference FPA mean | `im_FPA = np.mean(im_wr, axis=0)` | 19 |
| nb 1 | **the conversion (§B4, §6d)** | `# convert to reflectance` | 20 |
| nb 1 | save (§B7) | `spectral.envi.save_image(fname_im+'_ref'` | 26 |
| nb 1 | red-HTML "you will need to change" cells (§A1) | `<span style="color: red">` | 2, 7, 10, 13 |
| nb 1 | screenshot-only markdown cells (§C5) | `copy-paste from hsi_viewer` / `attachment:image.png` | 16, 17, 23, 24 |
| nb 2 | hardcoded image pair (§A1) | `fname_img = 'data/morven_9-2025/` | 4 |
| nb 2 | **`NameError` on a clean run (§A4)** | `RGB_image = msf.make_rgb(im.Arr, wl, stretch` | 6 |
| nb 2 | the ROI-drawing viewer call | `hvr.viewer(im, stretch=[0,99.5]` | 10 |
| nb 2 | hardcoded ROI pickle (§A1) | `fname = 'data/pkl/ROIs_4-25_Ilex_vom.pkl'` | 12 |
| nb 3 | dead Windows paths (§A2) | `C:/spectral_data/spectral_images/Vegetation_` | 2 |
| nb 3 | live paths (§A1, §A2) | `data/4-10-2025/100124_HarrisMicro` | 3 |
| nb 3 | plot loop drawing the wrong mean (§B9) | `class_spectra[i,:].flatten()/np.mean(` | 14 |
| nb 3 | PCA whitening | `Log-Eigenvalues for the Covariance Martrix` | 21 |
| nb 3 | LDA (all classes / hardcoded subset) | `roi_names_veg = ['Soli_sem'` | 25, 26 |
| nb 3 | Mahalanobis, near-duplicate pair (§B8) | `MD = np.zeros((nSpec, nClasses))` — **matches both**, by design | 28, 30 |
| nb 3 | **WIP metrics section to delete (§A3)** | `# WORK-IN-PROGRESS Evaluation Metrics` | 31–35 |

Every anchor above was verified against `6032500`: each matches the listed cell, and
each is unique to it except the `MD = np.zeros(...)` line, which is deliberately the
anchor for the near-duplicate pair.

**Duplicated blocks to collapse in phase 4:** nb 2 cells 13–16 are the same ROI
evaluation as nb 3 cells 8–11 (unpack `roiData` → mask grid → RGB-with-ROIs overlay →
show `df`). One implementation in `src/upwins_microscene/display.py`, called from both.

### Syncing `hsiViewer` (phase 1)

Take the **companion's** copy, not this repo's. They are the same file with two
differences, both of which matter:

- **Line endings.** `research_UPWINS_Microscene/hsiViewer/*.py` is LF;
  `upwins-hsi-preprocessing/src/hsiViewer/*.py` is CRLF. After normalizing, four of
  the five modules are byte-identical.
- **One hunk** — `hsi_viewer_ROI.py` line 523, the `loadROIs` fix (§B3). The
  companion has it; this repo does not.

So the sync is a straight copy of all five modules from
`upwins-hsi-preprocessing/src/hsiViewer/`, plus its `__init__.py`. Confirm with:

```bash
for f in hsi_viewer.py hsi_viewer_2.py hsi_viewer_ROI.py hsi_viewer_array.py hsi_viewer_layers.py; do
  diff <(tr -d '\r' < research_UPWINS_Microscene/hsiViewer/$f) \
       <(tr -d '\r' < upwins-hsi-preprocessing/src/hsiViewer/$f)
done
# Expected: silence for four files; for hsi_viewer_ROI.py, exactly the 4-line
# loadROIs fix at ~523. Anything else means the companion has moved on — re-check
# before copying.
```

**Do not rename `hsiViewer`.** Every ROI pickle on disk — including this repo's
`test_baccharis_ham_with_backgrounds.pkl` — deserializes to
`hsiViewer.hsi_viewer_ROI.ROIs_class`, and `upwins-veg-classifier` ships a stand-in
class at that exact path. Renaming here silently breaks both.

### Working agreement

- Develop on `claude/upwins-microscene-cleanup-plan-7gttbi` in
  `upwins/upwins-microscene-preprocessing`; do not push to `main` without being told.
- **Make no commits to `upwins-hsi-preprocessing` or `upwins-veg-classifier`.** Read
  them freely — that is what §1 is for. If you find a defect in either, add it to §8
  as a note; do not act on it.
- One commit per phase (§5). Phases 5–7 are independently approvable.
- The decisions in §6a and §6b are **settled**; §6c items proceed on their defaults.
  Do not re-litigate any of them, and do not invent a replacement for the `LICENSE`
  copyright line.

---

## 1. The pattern the two cleaned repos established

This is the baseline. The new repo should copy each answer rather than invent one.

| Problem | The established answer | Microscene today |
|---|---|---|
| Where parameters live | `config.yaml` at the root; "edit paths/parameters here rather than in the notebooks" | no config; literals in cells, with `<<<=== You need to modify these ===>>>` banners |
| Notebooks in `notebooks/`, config at root | `REPO_ROOT` walk-up + absolutize configured paths. **No `os.chdir`, no `sys.path` mutation** | notebooks at root; `import microscene_functions` only resolves from the root |
| Importing repo code | `src/` layout + `pyproject.toml` + `pip install -e .` | bare imports off cwd |
| ROI pickle compatibility | `hsiViewer` kept as a **top-level import name** so `hsiViewer.hsi_viewer_ROI.ROIs_class` unpickles | same import name today — must be preserved |
| Data | `data/` gitignored **in full**; **nothing ships**; docs say so plainly | `data/` ignored, but a 5.8 MB ROI pickle is committed at the root |
| Data documentation | `docs/data.md` (not `data/README.md` — the mount hides it) | none |
| README shape | Quickstart → notebook table → Layout → Data → *If you use the devcontainer* → Acknowledgment (NSF 2319470) | two lines |
| Narration | a short markdown cell above **every** code cell; notebooks double as the written walkthrough | ~14 markdown cells in nb 1, 5 in nb 2, 8 in nb 3 — several are pasted screenshots |
| Devcontainer | `python:3.12-bookworm` + `python3-pyqt5`, `postCreateCommand` editable install, `${localEnv:HOME}` mount, no `--gpus` | none — `.gitignore` actively ignores `.devcontainer/` |
| requirements | pinned, with a comment header explaining why | unpinned; 8 packages nothing imports |
| Tutorial | `docs/recording_runbook.md`, cross-referenced as Video 1 / Video 2 | none |
| Tests / CI | **none, by owner decision** (recorded, not silently absent) | none |
| `CITATION.cff` | deliberately **deleted** from both repos (`ca90b2f`, `6abfce1`) | n/a — do not add one |
| Working audit doc | kept in-repo (`AUDIT_HANDOFF.md`, `docs/temp_auditplan.md`) | this document is its analogue |

---

## 2. Inventory and disposition

Total tree ≈ 34 MB working, 53 MB of git history. Deliverable content is ≈ 1 MB.

| Path | Size | Disposition |
|---|---|---|
| `1 UPWINS Mircoscene preprocesing.ipynb` | 0.47 MB | **Keep** → `notebooks/01_convert_to_reflectance.ipynb` |
| `1 UPWINS Mircoscene preprocesing copy.ipynb` | 0.47 MB | **Drop.** Differs from the above in exactly three cells and only in literals (`crop_rows`, `crop_cols`, `white_ref_rows`) — i.e. it is the same notebook pointed at a second collection. That is precisely what `config.yaml` is for (§A5). |
| `2 UPWINS Mircoscene Creating ROIs.ipynb` | 2.6 MB | **Keep** → `notebooks/02_create_rois.ipynb` |
| `3 UPWINS Microscene ROI visualization and Analysis.ipynb` | 3.0 MB | **Keep, trimmed** → `notebooks/03_roi_analysis.ipynb` (see §A3 on its broken tail) |
| three notebook PDFs | 11 MB | **Drop** — stale exports of the above; the runbook covers exports as an action item |
| `Efficient_Hyperspectral_Target_Detection…pdf` | 2.5 MB | **Drop** — a third-party IEEE paper; unrelated to the pipeline and a redistribution question we should not inherit. Cite it in `docs/` if it is a method reference. |
| `from_Gia/`, `from_Luz/`, `from_Meesun/`, `from_Jim/` | 7.5 MB | **Drop.** Their own `notes.txt` says they are per-person upload folders for review. `from_Jim/util_scripts.py` is an ancestor of `upwins-veg-classifier/src/upwins_veg/spectral_collection.py` (same `set_color`, `sort_dict_by_list`, `SpectralCollection`) — already delivered there. |
| `in_development_code/` | 3.5 MB | **Drop** — exploratory, Windows-absolute paths, calls `read_im()` which is never defined |
| `hsiViewer/` (5 modules) | 84 KB | **Keep** → `src/hsiViewer/`, but take the **companion's copy**, not this one (§B3, §C3) |
| `microscene_functions.py` | 4 KB | **Keep, split** → `src/upwins_microscene/` (only `make_rgb` is defined; it imports `pandas`/`sklearn`/`pickle`/`os` and uses none of them) |
| `test_baccharis_ham_with_backgrounds.pkl` | 5.8 MB | **Drop** — a committed ROI pickle (`hsiViewer.hsi_viewer_ROI.ROIs_class`); nothing ships |
| `requirements.txt` | | **Rewrite** — pin, drop the 8 unused packages (§C1) |
| `README.md` | 2 lines | **Rewrite** to the companion shape |
| `LICENSE` | | **Keep MIT** — but the copyright line differs from both companions (§C7, owner decision) |
| `.gitignore` | | **Rewrite** — it currently ignores `.devcontainer/` (§C8) |

---

## 3. Findings

Severity follows the companions' convention: **A** = blocking for handoff,
**B** = correctness, **C** = packaging / hygiene.

### A. Blocking — the repo cannot be handed over as-is

| # | Finding | Evidence |
|---|---|---|
| **A1** | **No configuration file.** Every path, crop bound and white-reference bound is a literal inside a code cell, and the notebooks instruct the user to edit them: five `<span style="color: red">You will need to change…</span>` markdown cells in nb 1, plus the `<<<=== You need to modify these ===>>>` banner convention introduced in cell 0. This is the exact thing `config.yaml` was introduced to replace in the other two repos, and it is the single largest gap. | nb 1 cells 0, 2, 3, 7, 8, 10, 11, 13, 14; nb 2 cells 4, 12; nb 3 cells 2, 3 |
| **A2** | **Dead Windows paths at the top of notebook 3.** Cell 2 sets `C:/spectral_data/...` and `C:\\spectral_data\\...`; cell 3 immediately overrides all three with `data/...` paths. Cell 3 also contains a commented-out copy of itself. A reader cannot tell which is authoritative. | nb 3 cells 2–3 |
| **A3** | **Notebook 3's last section does not run.** Under `# WORK-IN-PROGRESS Evaluation Metrics`, cells 33–35 reference `gt_list`, `MD_all` and `class_names` — **none of which is ever defined** in the notebook (cell 28 defines `MD`, not `MD_all`). Run-all ends in `NameError`. Shipping a notebook whose last three cells crash is the same class of false promise the companions' Phase 1 closed. **Answered: delete the section** (§6a, decision 4). | nb 3 cells 31–35 |
| **A4** | **Notebook 2 raises `NameError` on a clean run.** Cell 6 calls `os.path.basename(fname)`, but `fname` is not bound until cell 12; cells 4–5 define `fname_img` / `fname_hdr`. The notebook only appears to work because cell 12 had been run in a previous session. | nb 2 cell 6 |
| **A5** | **Two copies of notebook 1** whose only differences are three tuning literals for a second collection. Nothing says which is current. | `1 …ipynb` vs `1 … copy.ipynb`, cells 3/11/14 |
| **A6** | **No client-facing documentation.** README is a title and one sentence. No `docs/`, no data guide, no layout, no Quickstart, no recording runbook, no NSF acknowledgment — the four things a client opening this beside the other two repos will immediately miss. | `README.md` |
| **A7** | **No devcontainer, and `.gitignore` prevents one.** `.gitignore` line 3 is `.devcontainer/`. Both companions offer the container as the primary install path. | `.gitignore` |

### B. Correctness

| # | Finding | Evidence | Proposed handling |
|---|---|---|---|
| **B1** | **`file_names.find_files()` writes to globals, not `self`.** Every append targets the module-level name `fnames`, and every `os.path.join` uses the global `dir` / `dir_dark` rather than `self.dir` / `self.dir_dark`. The class therefore works for exactly one instance that must be named `fnames`, in a notebook that must define `dir`. A second instance silently appends to the first. | nb 1 cell 4 | Fix while extracting to `src/upwins_microscene/file_search.py`. Behavior-preserving for the single-instance use. |
| **B2** | **Hardcoded list indices into the file search.** `fname_png = fnames.png[1]` — index **1**, so a collection with one `.png` raises `IndexError` and one with three picks arbitrarily. `.jpg[0]` / `.hdr[0]` have the same issue. Missing files become the string `'None'`, which surfaces much later as `FileNotFoundError: 'None'`. | nb 1 cell 8 | Print the candidates, select by config key with a clear error naming the directory searched. |
| **B3** | **Data-loss bug in the ROI viewer.** `loadROIs` stores `copy.deepcopy(self.ROImask_empty[:])` — an **empty** mask — instead of the mask it just read. Open an ROI `.pkl` to extend it, add a region, save: the pre-existing ROIs are written back empty. This is the companion's finding **B5**, fixed there on 2026-07-30; this repo's copy still has the bug, and notebook 2 is *exactly* the open-extend-save workflow. | `hsiViewer/hsi_viewer_ROI.py:523` | Take the companion's fixed file. See §C3 — the two copies are otherwise identical. |
| **B4** | **Reflectance conversion — three things to decide, not silently change.** `im_ref[:,c,b] = (counts[:,c,b] − dark_mean[b]) / (white_FPA[c,b] − dark_mean[b])`. (i) The white reference is resolved **per column**, but the dark is a **scalar per band** (spatially averaged over the whole dark cube), mixing two resolutions. (ii) No guard against `white_FPA[c,b] − dark_mean[b] ≈ 0` → `inf`/`NaN` propagating into the saved cube. (iii) No clipping; nothing flags physically impossible reflectance. Also a double Python loop (`nc × nb` passes) where one broadcast expression would do. | nb 1 cell 20 | **Answered: vectorize only** (§6a, decision 2). The arithmetic is reproduced bit-for-bit; (i), (ii) and (iii) are left as they are and recorded in the audit doc, with a non-finite-output warning that changes no value. Exact definition in **§6d**. |
| **B5** | **Crop / white-reference geometry is assumed, never checked.** `white_ref_rows` indexes the **cropped** array while `crop_rows` indexes the raw one; nothing asserts the white-reference band lies inside the crop, or that the panel spans the full cropped width (which the markdown says it must). A bad crop yields a wrong calibration with no error. | nb 1 cells 11, 14, 18 | Add asserts with messages naming the config keys — the companions' `image`/`image_hdr` mismatch-guard pattern. |
| **B6** | **The viewer cell inspects the wrong array.** nb 1 cell 15 opens `hva.viewer(im.Arr, im.wl)` on the **uncropped** cube while everything downstream uses `imArr_cropped`. | nb 1 cell 15 | Point it at the cropped array. |
| **B7** | **`_ref` filename convention disagrees with itself across notebooks.** nb 1 saves with `ext=''`, producing an extension-less `<name>_ref` image; nb 2's example input is `raw_55691_or_ref.img`. The companion repos settled on `.img` for `_ref` products and on explicit `image` + `image_hdr` config pairs. | nb 1 cell 26 vs nb 2 cell 4 | **Answered: `.img`** (§6a, decision 3), with the explicit `image` / `image_hdr` config pair and the stem-mismatch assert both companions use. |
| **B8** | **Notebook 3 cells 28 and 30 are ~90 % duplicated**, `roi_means = {}` is created and never filled in either, and `mu` is silently re-bound from the global mean (cell 21) to a per-class mean inside the loop — so cell 25's LDA depends on cells having been run in one specific order. | nb 3 cells 21, 25, 28, 30 | Extract the shared Mahalanobis routine to `src/upwins_microscene/roi_stats.py`; the two cells become two calls. |
| **B9** | **A plot loop that draws the wrong thing.** nb 3 cell 14 plots every class's normalized spectra inside a loop, then calls `plt.plot(means[name]…)` *after* the loop using the leaked loop variable — so only the last class's mean is drawn, over all classes' spectra, with a title naming that one class. | nb 3 cell 14 | Move the mean plot inside the loop (matches cell 13's intent). |
| **B10** | **`im.List` / `dataList` computed in both nb 2 and nb 3 and used once**, in nb 3 cell 30. On a full cube this is a second full-size copy in memory. | nb 2 cell 5, nb 3 cell 4 | Compute where used. |

### C. Packaging, hygiene, docs

| # | Finding | Evidence |
|---|---|---|
| **C1** | **`requirements.txt` unpinned, and 8 of its 19 entries are imported nowhere** in the deliverable code: `pyshp`, `rasterio`, `statsmodels`, `plotly`, `hyperspectral_gta_data` (only in commented cells — plus one *uncommented but unused* import in nb 1 cell 3), `python-dotenv` and `pymongo` (only `from_Jim/working.ipynb`), `ipython`. Actual runtime set: numpy, matplotlib, Pillow, spectral, scikit-learn, pandas, PyQt5, PyQt5-sip, pyqtgraph — plus PyYAML once `config.yaml` exists. Both companions pin with an explanatory header. |
| **C2** | **No `pyproject.toml`, no `src/` layout.** `import microscene_functions` and `from hsiViewer import …` resolve only when cwd is the repo root; moving notebooks into `notebooks/` (which this plan does, for parity) breaks them. This is the companions' P0-1, solved there with `src/` + `pip install -e .` + the `REPO_ROOT` walk-up. |
| **C3** | **`hsiViewer/` is a stale fork of the companion's.** After normalizing CRLF, all five modules are **byte-identical** to `upwins-hsi-preprocessing/src/hsiViewer/*` except for one hunk: the four-line B5 fix this repo lacks. Also, nb 2 and nb 3 import `hsi_viewer`, `hsi_viewer_layers` and `hsi_viewer_ROI` but use only `hsi_viewer_ROI` — the companion's still-open finding **C4** (three unused viewer modules), inherited here. |
| **C4** | **Notebook filenames** carry spaces, two typos (`Mircoscene`, `preprocesing`) and a literal ` copy`. The companions use `NN_verb_noun.ipynb`. |
| **C5** | **~9 MB of committed cell outputs and pasted screenshots.** nb 3 alone carries 2.8 MB of outputs (27 PNGs); nb 2 carries 1.3 MB of outputs and 1.3 MB of attachments. Four markdown cells in nb 1 are *only* a pasted hsiViewer screenshot ("copy-paste from hsi_viewer") standing in for narration. |
| **C6** | **The same copy-pasted import block in all three notebooks**, including imports each does not use (`copy`, `pickle`, `importlib`, `hsv`, `hvl`, and `PCA` in nb 3 — imported, never called). The companions cleared this (their C5). |
| **C7** | **`LICENSE` says `Copyright (c) 2025 William F Basener`;** both companions say `Copyright (c) 2025 upwins`. Owner decision — do not change unilaterally. |
| **C8** | **`.gitignore` ignores `.devcontainer/`,** which is why there is no container. It also lacks the companions' explanatory comments and the `*.egg-info/` rule the editable install needs. |
| **C9** | **Typos throughout the client-facing prose:** `proceedires`, `Buils`, `Exctract`, `origonal`, `iamge` (×3), `bollean`, `refleactance`, `Transofrmed`, `drop ranges` (for *crop* ranges), `smays`. Both companions did a typo pass before handoff. |
| **C10** | **Mixed kernel metadata** across notebooks (3.12.9 / 3.11.4 / 3.11.9), including in the files being kept. The companions normalized this (their C8). |
| **C11** | **Notebook 1's section headings jump** from `Part 3` straight to `Part 6. Save reflectance image` — Parts 4 and 5 do not exist. |

---

## 4. Proposed target repo

```
README.md                     Quickstart → notebook table → Layout → Data →
                              "If you use the devcontainer" → Acknowledgment
LICENSE                       MIT
config.yaml                   every path and parameter, commented
pyproject.toml                src/ layout; upwins_microscene; no [project.dependencies]
requirements.txt              pinned, with the companions' comment header
.gitignore                    data/ in full + notebook/Python/build cruft
.devcontainer/                python:3.12-bookworm + python3-pyqt5,
                              postCreateCommand editable install,
                              ${localEnv:HOME} mount, no --gpus
notebooks/
  01_convert_to_reflectance.ipynb    dark + white-reference → reflectance
  02_create_rois.ipynb               draw and save labeled ROIs
  03_roi_analysis.ipynb              ROI spectra, PCA/LDA, Mahalanobis separability
src/upwins_microscene/
  __init__.py
  file_search.py              the file_names class, fixed (B1/B2)
  display.py                  make_rgb, ROI overlay, mask grid
  reflectance.py              white-reference conversion + geometry guards (B4/B5)
  roi_stats.py                class means, whitening, LDA, Mahalanobis (B8)
src/hsiViewer/                synced from upwins-hsi-preprocessing, incl. the B5 fix.
                              Import path kept as `hsiViewer` so ROI pickles load.
docs/
  data.md                     what you supply, what the notebooks produce, the mount
  recording_runbook.md        tutorial guide, cross-referenced with the other two
data/                         gitignored in full; nothing ships
```

---

## 5. Phases

Each phase is one commit. They are ordered so the repo is verifiable as early as
possible; phases 1–3 are the ones the later ones depend on.

| # | Phase | What lands | Depends on |
|---|---|---|---|
| **0** | **Scaffold** | New repo, `main` from an empty root: `LICENSE` (copied verbatim — **C7**, see §6b), `.gitignore` (**A7, C8** — and it must **not** ignore `.devcontainer/`), `README` skeleton, `pyproject.toml` (**C2**), pinned `requirements.txt` (**C1**), `.devcontainer/` (**A7**). Notebooks not yet moved. | — |
| **1** | **Fidelity import** | Import the three notebooks under their new names (**C4**), `hsiViewer/` (the companion's copy — **B3, C3**; procedure in §0), and `microscene_functions.py`, **contents otherwise unchanged**, into the target layout. Everything in §2 marked *Drop* is simply not imported. Record a byte-identity / cell-diff table like `AUDIT_HANDOFF.md` §1 so the science is provably preserved across the move. | 0 |
| **2** | **Packaging** | `src/upwins_microscene/__init__.py`, the `REPO_ROOT` walk-up in all three notebooks, `import microscene_functions` → `from upwins_microscene import display`. `from hsiViewer import …` **unchanged** (§0). Closes **C2**. First point the repo actually runs — verification recipe (§7) runs here. | 1 |
| **3** | **Config** | `config.yaml` with paths, crop bounds, white-reference bounds, stretch and RGB band targets. Delete the five red-HTML cells and the `<<<===>>>` banners; delete the duplicate notebook 1 (its literals become a commented second-collection example). Closes **A1, A2, A5**. | 2 |
| **4** | **Extract support code** | `file_search.py`, `display.py`, `reflectance.py`, `roi_stats.py`. Fixes **B1, B2, B8, B10** on the way; conversion vectorized (behavior-identical). | 3 |
| **5** | **Correctness** | **B3** (viewer fix — arrives free from phase 1 if the companion's copy is used; verify), **B5** geometry guards, **B6** viewer array, **B9** plot loop, **A4** `fname` NameError, **A3** delete notebook 3's WIP metrics section, **B7** `.img` + config pair + stem assert. **B4 is not in this phase** — the vectorization lands in phase 4 as a behavior-identical extraction (§6d), and nothing else about the formula changes. | 4 |
| **6** | **Narration + docs** | Markdown cell above every code cell; replace the four screenshot-only markdown cells (nb 1: 16, 17, 23, 24) with real narration (**C5**, first half); typo pass (**C9**); `README.md`, `docs/data.md`, `docs/recording_runbook.md`. Closes **A6**. | 5 |
| **7** | **Hygiene** | Strip committed cell outputs (**C5**, second half), normalize kernelspec (**C10**), trim import blocks (**C6**), fix headings (**C11**), decide on the three unused viewer modules (**C3**, second half — the companion's still-open C4). | 6 |
| **8** | **Audit doc** | An `AUDIT_HANDOFF.md`-equivalent recording what shipped, what was declined, what was verified and what could not be, and the open owner confirmations — including **C7** (the `LICENSE` line, §6b) and the §6c defaults actually taken. | 7 |

Phases 5–7 are independently approvable; you can take any subset.

**Coverage check.** All 28 findings in §3 are claimed by a phase above, so §5 doubles
as the implementation checklist: A1–A2, A5 → 3 · A3–A4 → 5 · A6 → 6 · A7 → 0 ·
B1–B2, B8, B10 → 4 · B4 → 4 (vectorize only, §6d) · B3, B5–B7, B9 → 5 · C1–C2, C7–C8
→ 0 · C3–C4 → 1 and 7 · C5 → 6 and 7 · C6, C10–C11 → 7 · C9 → 6. If a phase lands
without touching a finding it claims, say so in the phase-8 audit doc rather than
letting it disappear.

---

## 6. Decisions

> **Status — answered 2026-07-31.** Four decisions are settled; do not re-litigate
> them. **The list below is renumbered to match the owner's numbering** — 1 nothing
> ships, 2 vectorize only and flag, 3 `.img` plus the pair convention, 4 delete the
> WIP section — so a reference to "decision N" means the same thing in the chat, in
> this document, and in the audit doc that follows. (An earlier revision numbered
> these 1, 3, 4, 5 inside an eleven-item list; that numbering is dead.)
>
> §6a holds the four answered decisions, §6b the one flagged-and-left-alone item, §6c
> the remaining six that proceed on their stated defaults. This mirrors
> `AUDIT_HANDOFF.md` §6a/6b/6c in the companion repo. §6d defines decision 2's
> "vectorize only" precisely.

### 6a. Answered by the owner — implement as stated, do not re-litigate

1. **Does anything ship?** ✅ **Answered: no.** Matching both companions — **no data
   ships, no from-clone run**, with the docs saying so plainly rather than implying
   otherwise. This is what makes dropping the committed 5.8 MB ROI pickle correct,
   and it means `README.md`, `docs/data.md` and `config.yaml`'s header all state that
   the user supplies imagery under a gitignored `data/`.

2. **The reflectance conversion (§B4).** ✅ **Answered: vectorize only, and flag.**
   Nothing changes numerically. Specifically, all three sub-questions are left as they
   are and recorded rather than fixed:
   (a) the dark correction **stays** a per-band scalar while the white reference stays
   per-column;
   (b) a ~0 denominator **still** produces `inf`/`NaN` — no raise, no substitution —
   with a warning added that counts non-finite output pixels;
   (c) output is **not** clipped.
   What does change is the *implementation*: the double Python loop becomes one NumPy
   broadcast expression producing bit-identical values. **See §6d** for exactly what
   that does and does not cover, and how it is proven. This mirrors how the companions
   handled their inherited `gain*(counts+offset)` discrepancy — flag, do not silently
   fix — with the difference that theirs was later fixed on an explicit owner ruling,
   and this one may be too.

3. **`_ref` file extension (§B7).** ✅ **Answered: `.img`.** Notebook 1 writes
   `<name>_ref.img` + `.hdr`; `config.yaml` uses the explicit `image` / `image_hdr`
   pair convention with the `Path(...).stem` mismatch assert both companions carry.
   This also makes microscene reflectance products drop straight into
   `upwins-veg-classifier` without a filename special case.

4. **Notebook 3's WORK-IN-PROGRESS metrics section (§A3).** ✅ **Answered: delete
   it.** Cells 31–35 (`# WORK-IN-PROGRESS Evaluation Metrics` through the ROC block)
   come out. The audit doc will record what they were reaching for — a per-class
   `classification_report` and ROC/AUC over Mahalanobis scores — and that finishing it
   needs a ground-truth vector and a per-class score matrix that the notebook never
   builds. Nothing else in notebook 3 depends on them: `MD` (cell 28), the histograms
   (29) and the probability images (30) are all self-contained, so the notebook runs
   end to end once the tail is gone.

### 6b. Flagged and left alone — owner confirmed

**`LICENSE` copyright line.** ✅ **Confirmed 2026-07-31: flag it, do not change it
for now.** This repo says `Copyright (c) 2025 William F Basener`; both companions say
`Copyright (c) 2025 upwins`. The new repo carries this repo's line unchanged, and the
divergence is recorded in the audit doc as an open owner item — not silently
harmonized. This is deliberately the same posture as the companions' **P2-9 / D**
(grant number / license / companion-repo name awaiting one explicit owner "yes"), and
one answer would close all three repos at once. **Do not invent a replacement line.**

### 6c. Stated defaults — a session can proceed, but confirm if you disagree

None of these was raised, so each proceeds as written. All are cheap to reverse *if
caught early*; item 4 (package name) is the one that gets expensive after the fact.

1. **`hsiViewer` (§C3).** It will then live in three repos. *Default: vendor a synced
   copy* (what the other two do), with a comment in each naming
   `upwins-hsi-preprocessing` as the source of truth. Alternative: extract a fourth
   shared package — cleaner, but a fourth thing for the client to install.

2. **A batch conversion script?** `upwins-hsi-preprocessing` ships
   `scripts/batch_convert_reflectance.py`. Microscene conversion needs a per-image
   crop and white-reference row range, so a batch script would need a per-image table
   in the config. *Default: no batch script* unless you convert microscene collections
   in bulk.

3. **Tests / CI.** Declined in both companions, on the record. *Default: same* — no
   test files, no `.github/workflows`. The checks in §7 cover the same ground without
   adding files to the repo.

4. **Package name `upwins_microscene`** (mirroring `upwins_hsi`, `upwins_veg`), and
   the **recording runbook as "Video 3"** of the series. *Default: both yes.* Say now
   if you want a different package name — renaming later touches every notebook.

5. **NSF Grant No. 2319470** in the Acknowledgment, and naming the other two repos as
   companions. *Default: yes, matching both.* Related to §6b: the companions' **P2-9 /
   D** owner confirmation of the grant number is still open there too.

6. **History.** *Default: fresh history*, one "Initial commit … (client delivery)",
   exactly as `upwins-hsi-preprocessing` was created (`d0027c0`).
   `research_UPWINS_Microscene` stays as the provenance record.

### 6d. What "vectorize only" means (decision 2)

Spelled out here because "vectorize" can be read as licence to tidy the maths, and it
is not. **The rule is: every output value is bit-for-bit what the current notebook
produces. Only the number of Python statements executed changes.**

**What the cell does today** (nb 1 cell 20) — a double loop over columns and bands,
each pass writing one column of `nr` values:

```python
im_ref = np.zeros((im.nr, im.nc, im.nb))
for c in range(im.nc):
    for b in range(im.nb):
        im_ref[:,c,b] = np.squeeze((imArr_cropped[:,c,b] - image_dark_mean[b])
                                   / (im_FPA[c,b] - image_dark_mean[b]))
```

With this notebook's own crop (`crop_rows = [1,1999]`, `crop_cols = [95,600]`) that is
`505 × nb` interpreter iterations, each doing NumPy work on 1998 elements.

**What it becomes** — the same expression, once, with broadcasting:

```python
im_ref = np.zeros((im.nr, im.nc, im.nb))
# (nr,nc,nb) - (nb,) -> (nr,nc,nb);  (nc,nb) - (nb,) -> (nc,nb), broadcast over rows.
im_ref[:] = (imArr_cropped - image_dark_mean) / (im_FPA - image_dark_mean)
```

**Why this is bit-identical, not merely close:**

- Every operation is **elementwise**. There is no sum, mean, dot product or other
  reduction, so there is no accumulation order for NumPy to re-associate — the usual
  reason a vectorized rewrite drifts in the last bits does not arise here.
- Each output element is still computed as exactly two subtractions and one division
  on exactly the same operands, in exactly the same IEEE double/single precision.
- **Dtype is preserved deliberately.** The pre-allocated `np.zeros(...)` (float64) is
  kept and written through `im_ref[:] = ...`, so the right-hand side is still
  evaluated in the operands' dtype and *then* widened on store — the same two steps,
  in the same order, as the loop. Writing `im_ref = (...) / (...)` instead would
  return the operand dtype and quietly change the array's type.
- `np.squeeze` on a 1-D slice was already a no-op.

**What "only" excludes.** None of the following is done, in this phase or any other,
without a further ruling from you:

| Not done | Why it is tempting |
|---|---|
| Resolving the dark frame per column, to match the white reference | §B4(i) — the resolution mismatch is real, but changing it changes every number |
| Guarding or substituting a ~0 denominator | §B4(ii) — `inf`/`NaN` still propagate into the saved cube exactly as today |
| Clipping the output to a physical range | §B4(iii) |
| Reordering the crop, dark subtraction, or division | would change rounding |
| Changing the saved dtype (`.astype('float32')` in cell 26 stays) | — |

**The one addition, which changes no value:** after the conversion, count non-finite
pixels and warn if any exist (`np.isfinite(im_ref).all()`). It reports; it does not
substitute. This surfaces §B4(ii) to the user instead of letting `NaN`s reach the
classifier silently. Note one honest side effect of vectorizing: NumPy's own
`RuntimeWarning` for a zero denominator is currently emitted per offending `(c,b)`
pass and will instead be emitted once for the whole array. The values are unchanged;
the number of warning lines is not.

**The one real cost:** the loop held a single `(nr,)` temporary at a time; the
broadcast form materializes two cube-sized temporaries before the store. On a large
cube that raises peak memory by roughly two extra copies. If that is a problem on your
machine, the same expression chunked over column blocks is still bit-identical
(elementwise ops do not care where the block boundaries fall), and I will use that
form instead — say the word and it goes in from the start.

**How it will be proven.** The extracted function and a verbatim copy of the current
loop are run against the same inputs and compared with
`np.testing.assert_array_equal` (which requires identical `NaN` and `inf` placement,
not just closeness), plus an explicit `dtype` and `shape` check. Inputs: a synthetic
cube in the same dtypes that deliberately includes a zero denominator and a
zero-variance column, and — if you can supply one — a real microscene cube. **No real
cube is available in this environment**, so as in both companions' audits, that half
of the check will be listed as not verified rather than implied.

> **Already checked (2026-07-31).** The synthetic half of that test was run while
> writing this plan, on a `(61, 17, 29)` cube in both float32 and float64, including a
> column with an exact zero denominator: `assert_array_equal(loop, broadcast)` passes
> in both, with identical `dtype` and identical `NaN` placement. It also confirms the
> dtype point above — with float32 inputs the loop-plus-preallocated form yields
> `float64`, while the `im_ref = (...) / (...)` form yields `float32`. Keeping the
> `np.zeros` destination is what makes the two agree.

---

## 7. Verification recipe

The same shape as the companions', so the new repo can be checked the same way.

```bash
python3 -m venv /tmp/v
/tmp/v/bin/pip install -r requirements.txt
/tmp/v/bin/pip install -e .          # puts upwins_microscene + hsiViewer on the path

# must succeed from inside notebooks/ — no chdir, no sys.path mutation
cd notebooks && QT_QPA_PLATFORM=offscreen /tmp/v/bin/python -c "
from pathlib import Path; import yaml, pickle
r = Path.cwd()
while not (r/'config.yaml').exists() and r != r.parent: r = r.parent
C = yaml.safe_load(open(r/'config.yaml'))
import hsiViewer.hsi_viewer_ROI        # required to unpickle any ROI file
from upwins_microscene import display, reflectance, roi_stats, file_search
print('ok')
"
```

Plus, per the companions' notebook-editing protocol:

```bash
# every code cell parses, and each notebook round-trips byte-identically
python - <<'PY'
import ast, json, glob
for p in sorted(glob.glob('notebooks/*.ipynb')):
    nb = json.load(open(p))
    for c in nb['cells']:
        if c['cell_type'] == 'code' and not any(l.lstrip().startswith(('%','!')) for l in c['source']):
            ast.parse(''.join(c['source']))
    assert open(p).read() == json.dumps(json.load(open(p)), indent=1, ensure_ascii=True) + '\n', p
    print(p, 'ok')
PY

python -m pyflakes src/upwins_microscene/*.py src/hsiViewer/*.py
```

**What cannot be verified here:** anything needing a real microscene cube, a dark
frame, or a display for the PyQt viewer. As with both companions, the notebooks will
not have been executed end to end — that will be stated plainly in the audit doc
rather than implied otherwise.

---

## 8. Out of scope unless you ask

- Reworking the ROI analysis science in notebook 3 (PCA/LDA/Mahalanobis) — it is
  ported faithfully or not at all.
- Anything in `upwins-hsi-preprocessing` or `upwins-veg-classifier`. Their two open
  items (**C4**, three unused viewer modules; **P2-9 / D**, attribution confirmation)
  are noted here only because this repo inherits both.
- Merging microscene ROI collection with `upwins-hsi-preprocessing`'s notebook 03.
  They look similar but calibrate differently — benchtop white panel + dark frame vs
  in-scene tarps + empirical line — and separate repos match how the work is done.
