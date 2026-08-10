> [中文文档](gromacs-md.md)

# GROMACS / MD

## Role

The GROMACS / MD page submits CUDA-accelerated molecular dynamics jobs. It currently supports energy minimization, NVT, NPT, production MD, aMD, Metadynamics, Umbrella, binding-stability analysis, cryptic-pocket discovery, and trajectory post-processing entrypoints.

## Recommended workflow

1. Select an existing receptor/protein structure asset in the project. With only `.pdb/.gro`, the default "auto prepare GROMACS system" option generates topology, box, solvent, and ion steps; with an existing `.tpr` or `.gro/.pdb + .top`, WA-DD reuses the prepared system directly.
2. Open `GROMACS / MD` and select input assets.
3. Choose a workflow template or protocol, then set force field, water model, integrator, time, temperature, pressure, output frequency, and GPU.
4. Confirm that the right-side step inspector says the workflow validation passed. If it has errors, fix inputs or switch to dry-run.
5. Submit the job and follow events and output assets from the task center.
6. Open the MD output asset and inspect grouped trajectory, energy, analysis, structure, checkpoint, parameter, and log files.

## Starting from ligand / receptor assets

- Receptor-only MD: select a `.pdb/.gro` asset from protein preparation or upload. The page previews `pdb2gmx -> editconf -> solvate -> genion -> grompp -> mdrun` automatically.
- Parameterized complex MD: select the complex structure and matching `.top/.itp/.prm` assets. WA-DD then generates the `grompp/mdrun` path directly.
- Unparameterized ligand: selecting only `.sdf/.mol2` ligand structures without ligand topology blocks real execution. Generate or upload ligand topology first, then include it in the same MD job.
- Trajectory analysis: select `.xtc/.trr`, preferably with the matching `.tpr` and `.edr`; the page previews RMSD/RMSF/energy analysis commands.

## Advanced parameters

- `.mdp` files JSON: writes one or more MDP files.
- Command JSON: executes custom commands one by one for non-dry-run custom flows.
- Extra text files JSON: writes PLUMED, index, or helper text configs.
- Custom workflow JSON: stores step configuration for parameter-package and command-preview auditing.

## Checkpoint continuation

If a completed MD output asset contains both `.cpt` and `.tpr`, its output detail shows "Resume from checkpoint / extend production". Enter an extension time and WA-DD creates a new `md_production` job using `gmx mdrun -cpi ... -append`.

## Outputs

- Trajectory: `.xtc`, `.trr`
- Energy: `.edr`
- Analysis: `.xvg`, `.csv`, `.json`, `.xpm`
- Structure: `.gro`, `.pdb`, `.cif`
- Checkpoint: `.cpt`
- Parameters: `.mdp`, `.top`, `.itp`, `.ndx`, `.tpr`
- Logs: `.log`, `.txt`

`.xvg`, `.csv`, `.json`, and log files can be previewed in the page; `.xvg` files render a lightweight line chart.
