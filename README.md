# OpenADMET PXR — Structure Prediction Track

**Best submission:** `esm2_board_s200_af3_submission` — **LDDT-PLI 0.5538 · BiSyRMSD 3.53 · LDDT-LP 0.9188**.

---

## Method (winning submission)

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

## Competitive context

| Entry | LDDT-PLI | Method |
|---|---|---|
| leader (`dnan-ipd`) | 0.5725 | ESMFold2 ensemble + **physics selection** (OrbMol-v2 interaction energy + force gate) |
| **this work** | **0.5538** | ESMFold2 + AF3-pocket transplant (own-method) |
| `xX-its-amit-Xx` | 0.5472 base | multi-model z-hybrid ensemble |
| `aetherark` | 0.545 | ESMFold2 no-MSA + PoseBusters/GNINA/minimize |

Our 0.5538 **beats both other public-data entries**; the leader's edge is a **better selector**, not better generation.

## In progress — physics-based pose selection

Reproducing the leader's insight: select by an ML-potential (UMA/OMol25) **interaction energy** `E(complex) − E(protein) − E(ligand)` instead of confidence, over a **fixed pocket** (identical atom set per pose → comparable energies) with a max-force gate. On the ground-truth refset this **beats best-iptm by +0.044** (e.g. rescues a 0.11→0.85 catastrophic iptm miss to the oracle pose), confirming that physics can cross the selection wall where confidence cannot. Board evaluation ongoing.

## Software

| Purpose | Software |
|---|---|
| Structure generation | **ESMFold2** (primary), Boltz-2 / Boltz-2 NR-fine-tune, OpenFold3, AlphaFold3, Protenix, Chai-1 |
| Scoring (official metric) | **OpenStructure (OST 2.11.1)** — LDDT-PLI / BiSyRMSD / LDDT-LP |
| Physics selection | **fairchem / UMA (OMol25)** ML interatomic potential; ASE (optimization) |
| Protonation | PDBFixer + OpenMM (AMBER14, protein H); RDKit (ligand H); Open Babel |
| Cheminformatics | RDKit (bond-order assignment, symmetry-aware matching) |
| Docking / refinement (tested) | AutoDock Vina, GNINA, OpenMM |
| Compute | HiPerGator (SLURM, NVIDIA L4); Python (NumPy, SciPy) |

## Contribution

- **AF3-pocket transplant lever** — placing the ESMFold2-selected ligand into an AlphaFold3-quality receptor pocket, a simple splice that adds **+0.030 LDDT-PLI** over raw ESM and forms our best submission.
- **Rigorous, ground-truth validation harness** — a 17-system PXR hold-out scored with the *official* OST scorer, used to gate every lever (the public board is fragment-only, so this is the real generalization check).
- **Systematic falsification** of confidence-selection, refinement, and failure-tail-rescue levers — mapping the *selection wall* (pick ≈ random, oracle ~0.72 uncapturable) with data rather than intuition.
- **Physics-selection reproduction + bug fix** — identified that a naive per-pose pocket makes interaction energies *incomparable* (variable atom set), and fixed it with a **fixed-pocket** formulation, recovering a **+0.044** ground-truth gain over best-iptm.
