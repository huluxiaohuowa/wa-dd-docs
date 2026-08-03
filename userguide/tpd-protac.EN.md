> [中文文档](tpd-protac.md)

# TPD / PROTAC

## Role

The TPD/PROTAC module supports POI-E3 ternary-complex modeling, warhead, E3 ligand, and linker design workflows.

## Input asset chain

TPD is not a simple "protein in, protein out" workflow. DeepTernary inputs come from three source types:

- POI protein asset: `protein`, `prepared_protein`, or a complex structure.
- E3 ligase protein asset: `protein`, `prepared_protein`, or a complex structure.
- Degrader / MGD ligand asset: `ligand` or `prepared_ligand`.

PROTAC jobs also require four auxiliary PDB assets:

- POI-side binary ligand PDB.
- E3-side binary ligand PDB.
- POI-side mask PDB.
- E3-side mask PDB.

The UI presents the selection as "source → asset → file". POI, E3, and degrader/MGD are reusable upstream assets; binary ligand and mask PDB assets are auxiliary inputs for the DeepTernary run.

## Outputs and reuse

The reusable DeepTernary output asset is `ternary_complex`. It represents one ternary-complex prediction result and usually contains:

- PDB ensemble: ternary-complex structures that can enter interaction analysis or be reused as complex structures.
- Summary CSV: candidate conformations, seeds, and run summaries.
- Run logs: useful for diagnosing input, GPU, or ligand-correction issues.

Downstream workflows should reference the `ternary_complex` asset itself instead of asking users to manually copy PDB or CSV file paths.

## Result inspection

Recommended interaction order:

1. Confirm the `ternary_complex` source chain: POI, E3, and ligand should match the intended inputs.
2. Open or download the PDB ensemble to inspect the POI/E3/degrader relative conformation.
3. Inspect the summary CSV for candidate count, seed details, and failures.
4. For binding-interface interpretation, send the ternary-complex asset to the interaction-analysis page.

## API / Automation

Job submission and result lookup reuse the unified automation model of `project_id`, `asset_id`, and `job_id`. Automation should persist and pass asset IDs, not container file paths.
