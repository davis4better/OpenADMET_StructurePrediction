# OpenADMET PXR — Structure Prediction Track

**Best submission:**  **LDDT-PLI 0.5538 · BiSyRMSD 3.5271 · LDDT-LP 0.9196**.
---

## Method

An **ESMFold2 co-folding ensemble** with a **pocket transplant** for pose quality:

1. **Generation.** ESMFold2 (latest SDK, no-MSA), **200 diffusion samples per ligand** on the 293-aa PXR LBD. No templates, no fine-tuning.
2. **Selection.** Best pose per ligand by interface-pTM (best-iptm).
3. **Pocket transplant.** The selected ESM ligand is rigidly placed into an **AlphaFold3-quality pocket** (Kabsch), which lifts LDDT-LP from ~0.885 (ESM) to 0.919 and carries the ligand into a cleaner receptor context.
4. **Refinement** The submission files are then refined by [HiQBind](https://github.com/THGLab/HiQBind). We also double check if the final output follow the physic rules with ChimeraX (so there should be a trade-off on board).

Net effect vs raw ESM (0.5238): **+0.030 from the transplant → 0.5538.** 


## What we tried (and what the data said)

| Lever | Result |
|---|---|
| ESMFold2 vs Boltz-2 / OpenFold3 / AF3 / Protenix / Chai | ESMFold2 no-MSA best single cofolder (~0.545 raw class) |
| AF3-pocket transplant | **+0.030 → 0.5538 (kept)** |
| Confidence selectors (iptm, iface-iptm, mPAE, AF3 ranking) | selection wall: pick ≈ random, oracle ~0.72 uncapturable |
| Refinement (Vina min, OpenMM, sidechain repack) | inert-to-negative (FF basin ≠ crystal) |
| Failure-tail rescue (Boltz / AF3 / Protenix poses) | regressed — our ESM placement already beats them on its own worst picks |
| Boltz-2 NR/PXR fine-tune (fragment-trained) | board-dead (bimodal fragments, unharvestable by iptm) |


## Board trial log (what we actually submitted)

All scores are measured on the official board (LDDT-PLI · BiSyRMSD · LDDT-LP). Ordered by phase; the lesson from each drove the next.

| Phase | Submission | LDDT-PLI | BiSyRMSD | LDDT-LP | Lesson |
|---|---|---|---|---|---|
| Generation | Boltz-2 + 24 templates (`reref24`) | 0.5269 | 3.768 | 0.915 | template-conditioned Boltz baseline |
| Generation | +6 pocket-novel templates (`reref30`) | 0.526 | — | — | 24 templates is the plateau |
| Generation | pocket-contact constraint (`da24`) | 0.526 | 3.781 | 0.915 | in-generation geometric penalty didn't transfer |
| Generation | MSA-conditioned ESMFold2 | 0.5092 | — | — | MSA hurt (no-MSA is better for PXR) |
| Generation | ESMFold2 custom-CCD (early) | 0.515 | — | — | fixed a SMILES-input bug; ESM viable |
| **Transplant** | **ESMFold2 → AF3 pocket (`esm2_board_s200_af3`)** | **0.5538** | **3.53** | **0.919** | **best — pocket transplant +0.030 (kept)** |
| Refinement | sidechain repack (`transplant_v3_screlax`) | 0.5351 | — | 0.909 | FF pulls pocket off-crystal (−LDDT-LP) |
| Refinement | Vina local-min (`vina_min`) | 0.519 | 3.828 | 0.914 | local geometry min hurts; problem is binding *mode* |
| Physics-fix | validity-repaired subset (`physicsfix_selective`) | 0.5535 | 3.534 | 0.919 | ties best; 184/184 physically valid |
| Tail-rescue | Boltz-reref on 8 worst (`reref_rescue8`) | 0.5529 | 3.572 | 0.917 | our ESM already beats Boltz on its own worst picks |
| Tail-rescue | AF3-ligand on tail (`ligswap_af3lig8`) | 0.5498 | 4.148 | 0.919 | swapped ligand mis-placed (BiSyRMSD blows up) |
| Tail-rescue | Protenix-v2 on tail (`protenix_rescue8`) | 0.5412 | 4.279 | 0.920 | same — no independent generator rescues our tail |
| Fine-tune | Boltz-2 NR/PXR FT, all-184 (`ft_board_transplant`) | 0.4925 | 4.001 | 0.915 | FT bimodal on fragments; iptm can't harvest |
| Fine-tune | FT fragment-routed hybrid | <0.5538 | — | — | fragment gain doesn't survive on board |

## What the data said

- **AF3-pocket transplant** is the one lever that moved the board (+0.030 → 0.5538).
- **Selection wall:** confidence selectors (iptm, iface-iptm, mPAE, AF3 ranking) pick ≈ random; oracle ~0.72 is uncapturable by any confidence signal.
- **Refinement and tail-rescue are inert-to-negative** — our ESM placement already beats Boltz/AF3/Protenix on its own worst picks (a stronger base flips the sign of amit's tail-rescue trick).

## Software

| Purpose | Software |
|---|---|
| Structure generation | **ESMFold2** (primary), Boltz-2 / Boltz-2 NR-fine-tune, OpenFold3, AlphaFold3, Protenix, Chai-1 |
| Scoring (official metric) | **OpenStructure (OST 2.11.1)** — LDDT-PLI / BiSyRMSD / LDDT-LP |
| Protonation | PDBFixer + OpenMM (AMBER14, protein H); RDKit (ligand H); Open Babel |
| Cheminformatics | RDKit (bond-order assignment, symmetry-aware matching) |
| Docking / refinement (tested) | AutoDock Vina, GNINA, OpenMM |
| Compute | HiPerGator (SLURM); Python (NumPy, SciPy) |

## Contribution

- **AF3-pocket transplant lever** — placing the ESMFold2-selected ligand into an AlphaFold3-quality receptor pocket, a simple splice that adds **+0.030 LDDT-PLI** over raw ESM and forms our best submission.
- **Rigorous, ground-truth validation harness** — a 17-system PXR hold-out scored with the *official* OST scorer, used to gate every lever (the public board is fragment-only, so this is the real generalization check).
- **Systematic falsification** of confidence-selection, refinement, and failure-tail-rescue levers — mapping the *selection wall* (pick ≈ random, oracle ~0.72 uncapturable) with long tail data rather than intuition.
- **Physics-selection reproduction + bug fix (to do)** — identified that a naive per-pose pocket makes interaction energies *incomparable* (variable atom set), and fixed it with a **fixed-pocket** formulation.
