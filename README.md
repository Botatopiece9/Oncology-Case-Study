# Oncology-Case-Study: NEK2 Virtual Screening — Vecura Pipeline Run

An end-to-end AI-guided virtual screening run against **NEK2** (Serine/threonine-protein kinase Nek2, UniProt `P51955`), executed on the [Vecura](https://vecura.ai) drug discovery platform. The workflow chains target identification, de novo molecule generation, structure-based screening, ADMET filtering, docking, MM/GBSA binding free energy, and ML-based bioactivity / drug-target interaction prediction into a single reproducible graph.

This repository contains the exported workflow definition and the per-stage input/output artifacts produced by the run, so the full pipeline can be inspected or re-imported into Vecura.

## Target

| | |
|---|---|
| Gene / symbol | NEK2 |
| UniProt ID | P51955 |
| Organism | *Homo sapiens* |
| Reference structure | PDB [2W5A](https://www.rcsb.org/structure/2W5A) (X-ray, 1.55 Å) |

## Pipeline overview

The workflow is an 11-node directed graph ([`workflow/NEK2_Virtual_Screening_workflow.json`](workflow/NEK2_Virtual_Screening_workflow.json)) with two branches that converge at docking:

```
Target Mapping ──► Structure Acquisition ──► Structure Preparation ──┬──► Pocket Identification ─┐
   (UniProt)              (RCSB + AlphaFold        (PDBFixer)         │                            ├──► Docking (GNINA) ──┬──► Binding Free Energy (Uni-GBSA)
                            fallback)                                 │                            │                       ├──► Bioactivity Prediction (LigoSpace)
Molecule Generation ──► Candidate Library ──► ADMET Screening ────────┴────────────────────────────┘                       └──► Drug-Target Interaction (DeepPurpose)
   (REINVENT4)              (RDKit)              (ADMET engine)
```

| # | Stage | Engine | Purpose |
|---|-------|--------|---------|
| 1 | **Target Mapping** | UniProt | Resolves the query `NEK2` to UniProt entry `P51955` and its full sequence |
| 2 | **Structure Acquisition** | RCSB (AlphaFold fallback) | Fetches the best available experimental structure — PDB 2W5A |
| 3 | **Structure Preparation** | PDBFixer | Cleans/protonates the receptor for downstream docking |
| 4 | **Molecule Generation** | REINVENT4 (staged learning) | De novo generates candidate molecules, scored by a geometric-mean objective over QED (0.6), MW, SlogP, TPSA, H-bond acceptors/donors (0.3 each), keeping the top 10 |
| 5 | **Candidate Library** | RDKit | Standardizes generated compounds into a structured library |
| 6 | **ADMET Screening** | ADMET engine | Computes physicochemical/ADMET property profiles (oral route) |
| 7 | **Pocket Identification** | P2Rank | Detects candidate binding pockets on the prepared receptor and ranks them |
| 8 | **Docking** | GNINA | Docks ADMET-passed compounds into the top pocket (exhaustiveness 128, 1 pose/compound, score cutoff ≤ −1) |
| 9 | **Binding Free Energy** | Uni-GBSA (MM/GBSA, `igb=2`) | Recomputes binding free energy on docked poses |
| 10 | **Bioactivity Prediction** | LigoSpace | Predicts IC50 from the docked complexes |
| 11 | **Drug-Target Interaction** | DeepPurpose | Predicts Kd (nM) from compound + target sequence, independent of the docking pose |

## Repository structure

```
.
├── workflow/
│   └── NEK2_Virtual_Screening_workflow.json   # Full Vecura workflow export (nodes, configs, edges)
├── data/                                      # Raw per-stage outputs, as exported from Vecura
│   ├── target_mapping/
│   │   ├── targets.csv                        # Resolved UniProt target
│   │   └── sequences.csv                       # Full NEK2 FASTA sequence
│   ├── structure_acquisition/
│   │   ├── structures.csv                      # Structure metadata (PDB ID, resolution, method)
│   │   └── structures/NEK2.pdb                 # Raw fetched structure
│   ├── structure_preparation/
│   │   ├── prepared_structures.csv
│   │   └── structures/NEK2.pdb                 # PDBFixer-prepared receptor
│   ├── admet_screening/
│   │   ├── compounds.csv                       # Generated compounds + REINVENT4 scores
│   │   └── profiles.csv                        # Full ADMET property profiles
│   ├── pocket_identification/
│   │   └── pockets.csv                         # 9 ranked pockets (P2Rank score/probability, residues, centroid)
│   └── docking/
│       ├── poses.csv                           # Docking scores per compound/pose
│       ├── compounds.csv
│       ├── receptor/prepared.pdb               # Receptor used for docking
│       └── poses/*.sdf                         # Docked ligand poses (GNINA)
└── results/                                    # Flattened, run-level summary tables
    ├── generated_compounds.csv                 # All 20 REINVENT4-generated candidates
    ├── bioactivity_predictions.csv             # LigoSpace IC50 predictions (top compounds)
    ├── dti_predictions.csv                     # DeepPurpose Kd predictions, ranked
    └── binding_energies.csv                    # Uni-GBSA MM/GBSA decomposition
```

## Results

Of the 20 REINVENT4-generated candidates, 5 compounds passed ADMET screening and were carried through docking, MM/GBSA, bioactivity, and DTI prediction. Best pocket (rank 1, P2Rank score 26.05, probability 0.91) was used for docking.

| Compound | SMILES | Docking score (kcal/mol) | GNINA CNN affinity | ΔG MM/GBSA (kcal/mol) | Predicted IC50 (µM) | Predicted Kd (nM) |
|---|---|---|---|---|---|---|
| opt2  | `Cc1c(CN2CCC3(CCNCC3)CC2)cc(CCC(=O)O)c(O)c1C` | −8.31 | 6.371 | −36.56 | 3.37 | 4590.98 |
| opt6  | `CCOc1ccc(CN(C)CC(=O)NCCc2cc(C)n[nH]2)cc1` | −7.34 | 6.104 | −45.83 | 3.57 | 2998.77 |
| opt7  | `C=C1CCC2C(C)(CO)C(O)CCC2(C)C1CCc1ccoc1O` | −7.44 | 5.898 | −34.42 | 3.27 | **25.91** |
| opt16 | `COC(=O)c1sc(Cl)cc1OCC(O)CNC(C)C` | −5.95 | 4.814 | −34.08 | 3.94 | 180161.69 |
| opt19 | `COc1ccc(CCNC(=O)CNC(=O)CC(C)C)cc1OC` | −6.87 | 5.614 | −36.84 | 3.42 | 7507.46 |

**opt6** gives the strongest MM/GBSA binding free energy (−45.83 kcal/mol) and the best docking score (−8.31 kcal/mol was opt2's docking score — note opt6 leads on ΔG, opt2 leads on docking/CNN affinity), while **opt7** stands out on the DeepPurpose Kd prediction (25.9 nM, several orders of magnitude tighter than the other candidates). Docking score and CNN affinity favor opt2; predicted Kd favors opt7 — the two ML-based interaction models rank compounds differently, which is expected given they use different input representations (pose-based vs. sequence-based), and is worth reconciling with orthogonal validation (e.g. FEP or assay data) before prioritizing a single lead.

## Reproducing / re-importing the run

The complete workflow — including every node's engine choice, parameters, and connectivity — is stored in [`workflow/NEK2_Virtual_Screening_workflow.json`](workflow/NEK2_Virtual_Screening_workflow.json). It can be re-imported into Vecura to rerun or modify the pipeline.

## Notes

- All `s3://` URIs referenced inside the CSVs point to Vecura's internal object storage for this run and are not publicly resolvable; the corresponding files needed for downstream use (prepared receptor, docked poses) are included directly in `data/`.
- Property columns prefixed `properties.*` are passed through unmodified from the Vecura pipeline outputs.
