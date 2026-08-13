> [中文文档](tpd-protac.md)

# TPD / PROTAC

For a complete hands-on case, see [TPD / PROTAC 5T35 case: BRD4-MZ1-VHL ternary complex](tpd-protac-5t35-case.EN.md).

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

These auxiliary PDB files are not new protein outputs. A binary ligand usually comes from the cocrystal small molecule in a POI-warhead or E3-ligand binary structure. A mask PDB is the substructure used to match the full PROTAC to the anchor/warhead atoms on each side. When users need to provide them, first create a TPD PDB asset from a PDB component in the protein-processing page, or upload a ligand/mask PDB asset in the ligand-processing page, then select that asset in the TPD page. The backend accepts asset IDs, not container file paths.

Auxiliary PDB selection rules:

- **Do not select a full protein or full cocrystal complex** as a binary ligand or mask. A file that contains both protein chains and a ligand is not the file required by these fields.
- **Binary ligand PDB** should be a ligand-only PDB extracted from a binary cocrystal structure, such as the warhead ligand from a POI-warhead structure or the E3 ligand from an E3-ligand structure.
- **Mask PDB** is the substructure used to align each end of the full PROTAC. If the cocrystal ligand exactly matches the PROTAC end group, the mask can reuse the same PDB as the corresponding binary ligand. If the cocrystal ligand contains extra atoms outside the PROTAC end group, export only the shared anchor substructure.
- **Degrader / MGD ligand** is the full PROTAC small molecule. Prefer a 3D SDF that RDKit can parse. If ligand preparation fails or logs show RDKit chirality, hydrogen, or bond-order errors, fix the full PROTAC file before changing protein assets.
- If the uploaded full PROTAC already has reasonable 3D coordinates and users want to preserve them, enable "Disable ligand correction". This avoids DeepTernary trying to rebuild ligand coordinates from an RCSB `{ligand_id}_ideal.sdf` file.

POI, E3, and degrader/MGD are reusable upstream assets; binary ligand and mask PDB files are traceable auxiliary PDB assets. WA-DD can save, associate, and pass these files, but it does not decide which cocrystal ligand or mask substructure is scientifically correct for the user.

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
