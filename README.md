# OpenADMET PXR — Structure Prediction Track

**Best submission:**  **LDDT-PLI 0.5538 · BiSyRMSD 3.53 · LDDT-LP 0.9188**.

---

## Method

An **ESMFold2 co-folding ensemble** with a **pocket transplant** for pose quality:

1. **Generation.** ESMFold2 (latest SDK, no-MSA), **200 diffusion samples per ligand** on the 293-aa PXR LBD. No templates, no fine-tuning.
2. **Selection.** Best pose per ligand by interface-pTM (best-iptm).
3. **Pocket transplant.** The selected ESM ligand is rigidly placed into an **AlphaFold3-quality pocket** (Kabsch superposition on pocket Cα), which lifts LDDT-LP from ~0.885 (ESM) to 0.919 and carries the ligand into a cleaner receptor context.
Net effect vs raw ESM (0.5238): **+0.030 from the transplant → 0.5538.**

## Validation

All development was gated on a **17-system PXR hold-out refset** with the **official OpenStructure LDDT-PLI scorer** (OST 2.11.1, defaults: radius 6 Å, thresholds [0.5,1,2,4], `add_mdl_contacts=True`, superposition-free, heavy-atom, symmetry-maximized). The board's public leaderboard reflects fragment ligands only, so the refset was the primary generalization check.

## What we tried (and what the data said)

| Lever | Result |
|---|---|
| ESMFold2 vs Boltz-2 / OpenFold3 / AF3 / Protenix / Chai | ESMFold2 no-MSA best single cofolder (~0.545 raw class) |
| AF3-pocket transplant | **+0.030 → 0.5538 (kept)** |
| Confidence selectors (iptm, iface-iptm, mPAE, AF3 ranking) | selection wall: pick ≈ random, oracle ~0.72 uncapturable |
| Refinement (Vina min, OpenMM, sidechain repack) | inert-to-negative (FF basin ≠ crystal) |
| Failure-tail rescue (Boltz / AF3 / Protenix poses) | regressed — our ESM placement already beats them on its own worst picks |
| Boltz-2 NR/PXR fine-tune (fragment-trained) | board-dead (bimodal fragments, unharvestable by iptm) |

## Software

| Purpose | Software |
|---|---|
| Structure generation | **ESMFold2** (primary), Boltz-2 / Boltz-2 NR-fine-tune, OpenFold3, AlphaFold3, Protenix, Chai-1 |
| Scoring (official metric) | **OpenStructure (OST 2.11.1)** — LDDT-PLI / BiSyRMSD / LDDT-LP |
| Physics selection | **fairchem / UMA (OMol25)** ML interatomic potential; ASE (optimization) |
| Protonation | PDBFixer + OpenMM (AMBER14, protein H); RDKit (ligand H); Open Babel |
| Cheminformatics | RDKit (bond-order assignment, symmetry-aware matching) |
| Docking / refinement (tested) | AutoDock Vina, GNINA, OpenMM |
| Compute | HiPerGator (SLURM); Python (NumPy, SciPy) |

## Contribution

- **AF3-pocket transplant lever** — placing the ESMFold2-selected ligand into an AlphaFold3-quality receptor pocket, a simple splice that adds **+0.030 LDDT-PLI** over raw ESM and forms our best submission.
- **Rigorous, ground-truth validation harness** — a 17-system PXR hold-out scored with the *official* OST scorer, used to gate every lever (the public board is fragment-only, so this is the real generalization check).
- **Systematic falsification** of confidence-selection, refinement, and failure-tail-rescue levers — mapping the *selection wall* (pick ≈ random, oracle ~0.72 uncapturable) with long tail data rather than intuition.
- **Physics-selection reproduction + bug fix (to do)** — identified that a naive per-pose pocket makes interaction energies *incomparable* (variable atom set), and fixed it with a **fixed-pocket** formulation.
