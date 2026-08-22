> [中文文档](fep-analysis.md)

# FEP and Analysis

## Role

FEP/analysis is reserved for free-energy calculations, trajectory inspection, uncertainty analysis, and reports.

## Inputs

- prepared complex or docking result
- ligand series
- simulation settings

## Input and parameter requirements

Production RBFE success depends mainly on input quality and mapping parameters:

Protein input:

- The protein PDB must carry complete hydrogens; when hydrogens are missing the system rebuilds them with PDBFixer, but rebuilt side chains are placed without clash checking and disordered side chains (e.g. arginine guanidinium groups) can land on neighboring atoms.
- Before running, the system scans inter-residue heavy-atom contacts (< 1.6 Å fails the job and lists the residues). For disordered charged side chains far from the pocket (> 10 Å), set "receptor clash handling" to "truncate to ALA" at submission time and list the residues (e.g. `A:741`, `A:832`); truncating only remote residues has negligible impact on ΔΔG.

Ligand series:

- Use a congeneric series so atom mappings exist; prepare all ligands together as a 3D SDF with explicit hydrogens.
- Ligand poses should share one binding-site frame. Keep the default "automatic O3A alignment"; even with O3A disabled, the system rigidly refines each ligand onto the reference over the mapped atoms (Kabsch) after mapping selection.

Mapping parameters:

- "Max 3D offset" is capped at 1.0 Å; keep the default 0.75 Å. A hybrid topology whose mapped atoms deviate by more than 1.0 Å explodes from valence strain during propagation (NaN), so the system rejects it and fails early.
- Leave "allow element changes" off unless you know why you need it.

Protocol parameters:

- Prefer timestep 1–2 fs and minimization_steps ≥ 20000. When validating a new target, first confirm the full chain with shorter production steps before scaling up.

## Outputs

- ΔG / ΔΔG tables
- trajectories
- analysis reports
- `fep_result` asset: stores the RBFE network, edge table, status, uncertainty, and raw result files.
- `fep_output` asset: the derived SDF additionally generated after FEP completes; preserves the input molecular conformations while writing FEP result fields into SDF properties, so it can be loaded, sorted, and exported in ligand preparation and interaction analysis.

## API / Automation

Currently a planned entry point; later workers will create jobs through `POST /api/v1/jobs` and return result assets via `output_asset_ids`.

## Server6 Example: RBFE dry-run on two sets of docked pose libraries

This example is completed in the `Example` project on server6, creating FEP/RBFE planning results for two sets of docking results respectively.

![FEP: select inputs, reference, and dry-run/production parameters](images/example2-step-14-fep-inputs-reference-dryrun-boxed.jpg)

![FEP: view edge table, download report or analyze derived SDF](images/example2-step-15-fep-results-buttons-edge-table-boxed.jpg)

Input assets:

- Protein: `Example 1TA2 receptor prepared ligand-removed`
- Pocket: `Example 1TA2 176 binding pocket`
- Congeneric docking pose library: `Example 1TA2 congeneric Uni-Dock screening result docked pose library`
- de novo docking pose library: `Example 1TA2 PocketXMol de novo Uni-Dock result docked pose library`

Outputs of this validation:

- Congeneric library: `Example 1TA2 congeneric docking-pose RBFE plan clean`
  - `fep_result`: 193 planned edges
  - `fep_output`: `Example 1TA2 congeneric docking-pose RBFE plan clean annotated SDF`, 194 SDF records
- de novo library: `Example 1TA2 de novo docking-pose RBFE plan clean`
  - `fep_result`: 68 planned edges
  - `fep_output`: `Example 1TA2 de novo docking-pose RBFE plan clean annotated SDF`, 69 SDF records

How to read the results:

1. Click `View FEP · N edges` to view the network/edge table. The dry-run results show `planned`, which is used to confirm the ligand map and does not represent real ΔΔG.
2. Click `Analyze FEP SDF · N molecules` to load the derived SDF into interaction analysis, where the structure, docking score, and FEP fields can be viewed together.
3. On the ligand preparation page, the `fep_output` category can be loaded as a normal SDF asset for 2D/3D viewing, sorting, and export.

Current limitations:

- The FEP worker currently requires `reference_ligand` to be a real record name within the input SDF. A separately extracted reference ligand asset cannot be passed directly as a cross-asset reference; to strictly use an independent reference asset, the FEP interface would need to add `reference_ligand_asset_id` and have the worker merge/map it.
- The dry-run does not run OpenMM production simulations, so it does not produce real ΔG/ΔΔG values or trajectories. Formal production jobs are more time-consuming and use GPUs.
