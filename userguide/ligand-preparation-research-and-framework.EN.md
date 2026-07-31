> [中文文档](ligand-preparation-research-and-framework.md)

# Ligand Preparation Requirements and Development Framework

This document targets the ligand preparation module of WA-DD. The goal is to cover the common ligand preparation needs of CADD researchers in molecular docking, virtual screening, and downstream SAR/FEP analysis.

This document defines the ligand-preparation capability required by WA-DD for CADD workflows: docking, virtual screening, SAR, and FEP. The implementation should not be a thin web-only SMILES converter. The production path uses a dedicated ligand-prep worker so that chemistry dependencies, future commercial tool adapters, and compute isolation can be managed independently from the web server.

## English executive summary

The ligand workflow must be docking-first. For Uni-Dock / AutoDock Vina docking, the primary ligand output is a `prepared_ligand` with 3D coordinates and proper charges; SDF is the primary interchange format, while PDBQT is the docking-ready format for Vina-family engines. The web application should manage upload, table mapping, Ketcher editing, asset lineage, job submission, and file management. The `wa-dd-ligand-prep-worker` should run the chemistry pipeline.

Minimum production capabilities:

- Inputs: SMILES lists, SDF/MOL/MOL2/PDB upload, tabular files with a SMILES column, empty ligand libraries, and Ketcher drawing/editing.
- Editing: edit a single molecule from an uploaded file or molecule library; save as a new asset/version by default; keep source asset, row index, molecule name, and edit reason.
- Standardization: RDKit sanitize, salt stripping, largest-fragment selection, metal disconnection, neutralization/reionization, canonical SMILES, InChIKey deduplication, and failed-molecule reports.
- Enumeration: pH-aware protonation when the backend is available, tautomer enumeration, undefined stereocenter enumeration, E/Z enumeration, and per-input variant limits.
- 3D preparation: conformer generation, MMFF/UFF optimization, conformer pruning, failure reporting, and optional retention of supplied 3D coordinates.
- Docking output: prepared SDF with 3D coordinates, Meeko PDBQT for Vina-family engines, manifest with molecule metadata.
- Optional compatibility: MOL2 for exchange, Open Babel fallback conversion.

Current implementation direction:

```text
WebApp / FastAPI
  -> uploads, Ketcher, tables, assets, API docs, task orchestration
wa-dd-ligand-prep-worker
  -> RDKit + OpenBabel + Meeko + Dimorphite-DL chemistry execution
assets
  -> ligand, prepared_ligand, prepared_ligand_library, optional docking_ready_ligand
```

## 1. Conclusion

The first version must be Uni-Dock docking-first. The core ligand input for docking is a ligand file with reasonable 3D structure, correct charges, and rotatable bond information. SDF is the primary interchange format, and PDBQT is the docking-ready format for Vina-family docking engines.

Recommended approach:

```text
WebApp / FastAPI
  Responsible for: upload, table column mapping, molecule editing, parameter setup,
       job submission, job status, asset/file management

ligand-prep-worker
  Responsible for: RDKit main pipeline + 3D conformer generation + Meeko PDBQT preparation
       + optional Open Babel fallback

Output assets
  prepared_ligand / prepared_ligand_library /
  ligand_conformer_set / optional docking_ready_ligand
```

Do not cram heavy computation and complex chemistry handling into the WebServer image. The WebServer should stay lightweight; ligand preparation should run in a separate worker, so that CPU/GPU/commercial-software-license deployments can be split later.

Open-source-first route:

1. `RDKit` as the core engine for chemistry objects, standardization, salt stripping, deduplication, SMILES/SDF read/write, and stereo/conformer handling.
2. `Meeko` as the PDBQT preparer for Uni-Dock / AutoDock Vina, the first-priority docking format output.
3. `Gypsum-DL` or `molscrub` as reference implementations or optional backends for state enumeration.
4. `Open Babel` as a format conversion and fallback tool, but do not deeply embed the Open Babel Python API into the main program; GPL propagation issues need separate evaluation.
5. Reserve a commercial adapter layer for `Schrödinger LigPrep/Epik` and `OpenEye OMEGA/QUACPAC`, but not as default dependencies.

## 2. CADD ligand preparation capabilities to cover

### 2.0 Docking-first compatibility principle

In Uni-Dock / AutoDock Vina docking tasks, the primary small-molecule input is a PDBQT file with 3D coordinates, correct charges, and rotatable bond information. SDF serves as the primary format for auditing, preview, and cross-tool exchange.

Therefore, the production goals of the ligand preparation module should be:

1. Obtain a trustworthy ligand with reasonable 3D structure from user input.
2. Generate PDBQT directly usable by Uni-Dock / Vina.
3. Keep SDF as the structural asset for auditing, preview, cross-tool exchange, and downstream FEP/SAR.
4. Keep canonical SMILES / InChIKey for deduplication and identification.

PDBQT is the primary output for the docking route; SDF is the universal interchange format.

### 2.1 Input capabilities

Must support:

- Single-molecule `SDF/MOL/MOL2/PDB` upload.
- Multi-molecule `SDF` upload.
- `SMILES` list input.
- `CSV/TSV/XLSX` table upload, allowing selection of:
  - SMILES column;
  - molecule name column;
  - compound ID column;
  - activity / batch / series and other reserved fields.
- Manual molecule drawing:
  - output molblock;
  - synchronously generate canonical SMILES;
  - support saving as a ligand asset.
- Single-molecule editing within a molecule library:
  - after uploading a multi-molecule SDF or SMILES list, the user can select any single molecule in the table;
  - click `Edit` to enter the 2D molecule editor;
  - on save, do not overwrite the original molecule; by default generate a new edited ligand asset or a new version;
  - keep `source_asset_id`, `source_row_index`, `source_molecule_id`, `edit_parent_id`.

Recommended to support:

- PubChem CID / ChEMBL ID / vendor ID import.
- Extract bound ligands from a protein PDB and save as a ligand asset.
- Copy ligands from prepared protein / docking result to the ligand preparation page.
- Reverse-load ligand SMILES from docking input files.

### 2.1.1 Molecule editor requirements

The frontend should embed a 2D molecule editor; Ketcher is recommended. It must cover three entry points:

1. Draw a molecule from scratch
   - user draws from zero;
   - save as a new ligand asset;
   - simultaneously save molblock, canonical SMILES, and a 2D preview image.

2. Edit a single uploaded molecule
   - after the user uploads a single-molecule SDF/MOL, click `Edit`;
   - the editor loads the molblock of that molecule;
   - on save, generate an edited ligand asset.

3. Edit a row in a molecule library
   - user uploads a multi-molecule SDF or SMILES table;
   - the molecule table is displayed row by row;
   - any row can be clicked `Edit`;
   - on save, generate a new molecule version and record the source in the manifest.

Editor save behavior must satisfy:

- Do not overwrite the original molecule by default.
- The new molecule after saving must go through RDKit sanitize again.
- If the structure is invalid, the frontend reports an error and does not create an asset.
- The new molecule after saving can be used directly in docking tasks.
- Keep edit history:
  - `source_asset_id`
  - `source_molecule_id`
  - `source_row_index`
  - `edit_parent_id`
  - `edit_reason` or user note.

### 2.2 Standardization and cleanup

Must support:

- Parse failure detection, and write failed molecules to `failed.sdf/csv`.
- RDKit sanitize.
- Salt stripping / largest organic fragment selection.
- Metal bond disconnection / metal disconnector.
- Functional group normalization.
- Charge normalization / uncharge / reionize.
- canonical SMILES / InChIKey deduplication.
- Molecule name, original row number, input file name retention.

Recommended to support:

- Option to keep salt forms.
- Keep the mapping between original molecule and prepared molecule.
- Support deduplication by InChIKey first block or full InChIKey.
- Rule-based annotation of reactive groups, PAINS, covalent warheads, organometallics, without forced removal.

### 2.3 State enumeration

Must support:

- pH range setting, e.g. `7.4 ± 1.0`.
- Protonation state enumeration.
- tautomer enumeration.
- Undefined stereocenter enumeration.
- Undefined E/Z double bond enumeration.
- Maximum variant count limit per input molecule.
- Each variant keeps genealogy / variant reason.

Recommended to support:

- Medicinal-chemistry rules to filter unreasonable tautomers.
- Do not flip defined chirality by default.
- User can choose:
  - strictly preserve input stereo;
  - enumerate undefined stereo;
  - enumerate all stereo.
- Display variant tree hierarchically by pH, tautomer, stereo.

### 2.4 3D conformer generation and optimization

Must support:

- Generate 3D from 1D/2D.
- Multi-conformer generation.
- MMFF94 or UFF preliminary optimization.
- Conformer deduplication.
- Maximum conformer count limit.
- Separate output for failed molecules.

Recommended to support:

- Ring conformer handling, especially 6-membered ring chair/boat.
- Macrocycle-specific strategy.
- Option to retain original 3D coordinates.
- Output only one low-energy conformer for docking; output a multi-conformer library for virtual screening.

### 2.5 Docking-compatible output

Must support:

- Each prepared ligand outputs an SDF with 3D coordinates.
- Generate PDBQT usable by Uni-Dock / AutoDock Vina (via Meeko).
- Keep metadata such as molecule name, original row number, input file.
- Generate manifest.csv to track the processing status of each molecule.
- If a pocket is used, support associating a pocket asset in the project.

Recommended to support:

- Automatically generate a unique ID for each ligand.
- Display 2D structure preview.
- Batch download prepared SDF / PDBQT.
- Output `manifest.csv`, `report.json`.
- Support one protein + multiple ligands batch docking preparation.

### 2.6 Other format output

Optional support:

- MOL2 output.
- AM1-BCC or other higher-quality charge backends as optional workers.
- Retain non-rotatable bond configuration.
- Issue explicit warnings for special elements such as covalent docking / metal coordination / boron / silicon.

Note: PDBQT is the primary input format for Uni-Dock / Vina family engines. SDF is the universal interchange format.

### 2.7 Quality control and reporting

Must support:

- Processing status of each input molecule:
  - success;
  - warning;
  - failed.
- Failure reason.
- Number of variants generated.
- Number of conformers generated.
- canonical SMILES.
- InChIKey.
- formal charge.
- heavy atom count.
- rotatable bond count.
- molecular weight.
- clogP / TPSA / HBD / HBA and other basic properties.

Recommended to support:

- Lipinski / Veber / lead-like / fragment-like rules.
- PAINS / Brenk / reactive group annotation.
- 2D structure thumbnails.
- 3D conformer preview.
- Retain/delete list after batch filtering.

### 2.8 Output assets

Each ligand preparation job produces at least:

```text
prepared_ligands.sdf
prepared_ligands.pdbqt
manifest.csv
failed.csv
report.json
```

At the asset level, it is recommended to split into:

| Asset type | Purpose |
| --- | --- |
| `ligand` | Original single ligand |
| `ligand_library` | Original multi-molecule library |
| `prepared_ligand` | Single prepared ligand |
| `prepared_ligand_library` | Prepared multi-molecule library |
| `ligand_conformer_set` | Multi-conformer output |
| `docking_ready_ligand` | PDBQT or other docking-specific output |

All outputs must be able to:

- Download;
- Preview;
- Rename;
- Copy to another project;
- Be selected as input assets by docking, FEP, and SAR pages.

## 3. Recommended tool stack

### 3.1 RDKit: default core

Uses:

- Molecule parsing and sanitize.
- Standardization, salt stripping, metal disconnect, normalize, reionize, tautomer canonicalization.
- SMILES/SDF read/write.
- canonical SMILES / InChIKey.
- Stereo recognition and enumeration.
- ETKDG 3D conformer generation.
- MMFF/UFF optimization.
- Basic descriptors and filtering.

Advantages:

- BSD 3-Clause, suitable for commercial product integration.
- Mature Python/C++ ecosystem.
- Low integration cost with Pandas, FastAPI, worker queues.

Limitations:

- pH-related protonation is not RDKit's strength; needs Dimorphite-DL, Gypsum-DL, molscrub, OpenEye/Schrödinger supplementation.
- Very complex macrocycles, metal complexes, and special elements require failure diversion.

### 3.2 Meeko PDBQT generator: docking main output

Uses:

- Convert prepared ligand 3D structure to PDBQT usable by Uni-Dock / Vina.
- Assign AutoDock atom types.
- Compute partial charges.
- Define rotatable bonds / torsion tree.
- Convert docking output back to RDKit/SDF.

Suitable for:

- Uni-Dock docking.
- AutoDock Vina.
- AutoDock-GPU.
- Large-scale docking workflows.

Limitations:

- Input should already have explicit hydrogens and 3D coordinates; therefore Meeko should be placed after RDKit/Gypsum-DL/other 3D preparation.
- Not a universal ligand preparation full-pipeline replacement.

### 3.3 Gypsum-DL: open-source state enumeration backend

Uses:

- Generate 3D-ready small molecules from SMILES or flat SDF.
- Enumerate ionization, tautomer, chiral, cis/trans, ring conformer states.

Suitable for:

- Open-source virtual screening.
- Batch preparation with strong pH-variant and ring-conformer enumeration needs.

Limitations:

- The project is relatively old; throughput, failure recovery, and modern Python compatibility need empirical testing.
- May be slow for very large, non-drug-like molecules.

### 3.4 molscrub: batch preparation in the AutoDock ecosystem

Uses:

- RDKit ETKDGv3 + UFF.
- tautomer enumeration.
- pH correction.
- ring chair enumeration.
- Batch processing for AutoDock docking.

Suitable for:

- Building docking-ready pipelines together with Meeko/Vina/AutoDock-GPU.

Limitations:

- GPL-3.0; be careful with commercial distribution.
- Documentation and API stability need empirical testing.

### 3.5 Open Babel: format conversion and fallback

Uses:

- Multi-format interconversion.
- `--gen3d` 3D generation.
- `-p <pH>` pH hydrogen addition.
- partial charge.
- Minimization.

Suitable for:

- Format fallback.
- Some formats inconvenient for RDKit read/write.
- Command-line fallback.

Limitations:

- GPL; if deeply linked or distributed, license evaluation is needed.
- Recommended to use first as an optional CLI worker, not directly embedded in the WebServer main pipeline.

### 3.6 Commercial benchmark backends

Adapters can be added later, not as default dependencies:

| Tool | Main capabilities | Adapter approach |
| --- | --- | --- |
| Schrödinger LigPrep + Epik | High-quality ionization, tautomer, stereo, ring conformation, 3D preparation | Worker calls command line under license environment |
| OpenEye OMEGA + QUACPAC | High-speed conformer generation, tautomer/protonation, charges | Worker calls toolkit/app under license environment |

The value of commercial backends is quality and throughput, but deployment, licensing, and cost are more complex. Architecturally, only a backend adapter needs to be reserved; do not bind product logic to a single commercial tool.

## 4. Recommended development framework

### 4.1 Backend data model

Add or extend:

```text
Job
  job_type = ligand_preparation
  input_asset_ids = [ligand or ligand_library]
  options_json = preparation parameters
  output_asset_ids = [prepared_ligand_library, optional docking_ready_ligand]
  result_json = stats, report paths, failed counts

Asset
  kind = ligand / ligand_library / prepared_ligand / prepared_ligand_library /
         docking_ready_ligand
  metadata_json = source mapping, generation options, chemistry summary

AssetFile
  role = input_sdf / prepared_sdf / pdbqt /
         manifest / failed / report / preview
```

### 4.2 Worker entry point

Recommended stable CLI definition:

```bash
wa-dd-ligand-prep \
  --input input.sdf \
  --input-format sdf \
  --output-dir /data/jobs/<job_id>/outputs \
  --ph 7.4 \
  --ph-tolerance 1.0 \
  --enumerate-tautomers true \
  --enumerate-protomers true \
  --enumerate-undefined-stereo true \
  --max-variants-per-mol 16 \
  --max-conformers-per-variant 20 \
  --output-sdf true \
  --output-pdbqt true
```

The worker outputs fixed files:

```text
prepared.sdf
prepared.pdbqt
manifest.csv
failed.csv
report.json
events.jsonl
```

### 4.3 Pipeline stages

```text
1. Load
   Read SDF/SMILES/CSV/XLSX/molblock

2. Validate
   sanitize, element check, duplicate ID check, failure logging

3. Standardize
   normalize, salt stripping, metal disconnect, uncharge/reionize, canonical identifiers

4. Enumerate
   pH/protomer, tautomer, stereo, ring state

5. Generate 3D
   ETKDG/MMFF/UFF, multi-conformer, failure fallback

6. Filter / Rank
   energy, duplicate conformer, max variant count, drug-like rules

7. Docking Format
   Meeko writes PDBQT, keep SDF mapping

8. Report
   manifest, failed, report, preview

9. Asset Commit
   outputs written to prepared_ligand_library / optional docking_ready_ligand
```

### 4.4 Frontend page design

The ligand preparation page is recommended to be split into the following areas:

1. Input area
   - Upload SDF/MOL2/CSV/XLSX.
   - Paste SMILES.
   - Draw molecules manually.
   - Import from project assets.

2. Molecule table and single-molecule editing area
   - After uploading SDF or SMILES list, parse the molecule table.
   - Each row shows name, SMILES, 2D thumbnail, status, source row number.
   - Each molecule has `Preview`, `Edit`, `Copy as new molecule`, `Delete/Keep`.
   - Click `Edit` to open a 2D molecule editor such as Ketcher.
   - After editing and saving, generate a new ligand asset or a new version in the library; do not directly overwrite the original input.

3. Column mapping area
   - SMILES column.
   - ID column.
   - Name column.
   - activity/series passthrough columns.

4. Preparation parameters area
   - pH.
   - State enumeration switches.
   - Stereo strategy.
   - Conformer count.
   - Output format: SDF / PDBQT.
   - Failure handling strategy.

5. Preview and QC area
   - Input molecule table.
   - 2D diagrams.
   - 3D conformers.
   - Warning/failed molecules.

6. Job and output area
   - Real-time progress.
   - Success/failure counts.
   - manifest.
   - prepared SDF download.
   - PDBQT download.
   - Copy to project.
   - Send to docking/FEP/SAR.

### 4.5 API draft

```text
POST /api/v1/assets/ligands/table
POST /api/v1/assets/ligands/smiles
POST /api/v1/assets/ligands/draw
POST /api/v1/assets/ligands/{asset_id}/molecules/{molecule_id}/edit

POST /api/v1/preparations/ligand
GET  /api/v1/jobs/{job_id}
GET  /api/v1/jobs/{job_id}/events
POST /api/v1/jobs/{job_id}/retry
POST /api/v1/jobs/{job_id}/cleanup

GET  /api/v1/assets/{asset_id}/files/{file_id}/download
POST /api/v1/assets/{asset_id}/copy
```

### 4.6 Docker image recommendations

WebServer:

```text
python:3.11-slim
FastAPI + SQLAlchemy + Redis client
Do not install heavy chemistry computation dependencies
```

ligand-prep-worker:

```text
python:3.11-slim or micromamba
RDKit
Meeko (PDBQT generation)
Optional: Gypsum-DL / molscrub / Open Babel CLI
```

Commercial worker:

```text
schrodinger-worker
openeye-worker
Enabled via license server or local license file
Only expose the same worker input/output protocol
```

## 5. MVP development order

### Phase 1: Usable

- SDF upload.
- SMILES list.
- CSV/XLSX SMILES/name/id column selection.
- Molecule table display.
- Open single molecule in editor, modify, and save as a new asset.
- RDKit sanitize.
- Salt stripping, standardization, deduplication.
- Generate canonical SMILES.
- 3D conformer generation.
- Output prepared SDF.
- Output PDBQT (Meeko).
- manifest / failed / report.
- Job status, rerun, clean up outputs.

### Phase 2: Batch docking usable

- One protein + multiple ligands batch docking preparation.
- Docking page can directly select prepared ligand.
- Multi-molecule library batch docking jobs.
- Docking output written back to ligand manifest for SAR sorting.

### Phase 3: State enumeration

- pH/protomer.
- tautomer.
- undefined stereo.
- ring conformer.
- max variants control.
- variant tree and warning display.

### Phase 4: Professional QC

- PAINS/Brenk/reactive group.
- Lipinski/Veber/lead-like/fragment-like.
- macrocycle strategy.
- metal/organometallic diversion.
- 3D conformer viewer.

### Phase 5: Commercial backend

- LigPrep/Epik adapter.
- OpenEye OMEGA/QUACPAC adapter.
- per-project backend selection.
- license/worker health check.

## 6. Key risks

1. pH/tautomer is not "a single correct answer"
   - The UI must show which states were generated; it cannot just emit one result.

2. Stereo must not be changed carelessly
   - Defined chirality is preserved by default.
   - Undefined chirality is enumerated only, unless the user explicitly requests enumerating all.

3. File and molecule mapping must not be lost
   - Each output molecule must keep source row, source ID, variant index, conformer index.

4. Open Babel / molscrub license
   - GPL components must not be mixed directly into the main WebServer.
   - If images are to be distributed, a clear license strategy is needed.

5. Large-scale libraries must be processed in streaming fashion
   - Do not load hundreds of thousands of molecules into the Web process memory at once.
   - The worker must write manifest and events in chunks.

6. CADD results must be explainable
   - Every deletion, enumeration, failure, and warning must be traceable.

## 7. Research sources

- RDKit MolStandardize docs: <https://www.rdkit.org/docs/source/rdkit.Chem.MolStandardize.rdMolStandardize.html>
- RDKit Overview / license: <https://www.rdkit.org/docs/Overview.html>
- Open Babel 3D generation: <https://openbabel.github.io/docs/3DStructureGen/Overview.html>
- Open Babel `obabel` command docs: <https://open-babel.readthedocs.io/en/latest/Command-line_tools/babel.html>
- Open Babel license FAQ: <https://openbabel.github.io/docs/Introduction/faq.html>
- Gypsum-DL GitHub: <https://github.com/durrantlab/gypsum_dl>
- Gypsum-DL paper: <https://pmc.ncbi.nlm.nih.gov/articles/PMC6534830/>
- molscrub GitHub: <https://github.com/forlilab/molscrub>
- Meeko ligand preparation: <https://meeko.readthedocs.io/en/develop/lig_prep_basic.html>
- Meeko overview: <https://meeko.readthedocs.io/en/develop/lig_overview.html>
- Uni-Dock GitHub: <https://github.com/dptech-corp/Uni-Dock>
- AutoDock Vina basic docking: <https://autodock-vina.readthedocs.io/en/latest/docking_basic.html>
- Schrödinger LigPrep: <https://www.schrodinger.com/platform/products/ligprep/>
- Schrödinger Epik: <https://www.schrodinger.com/platform/products/epik/>
- OpenEye OMEGA: <https://www.eyesopen.com/omega>
- OpenEye QUACPAC: <https://www.eyesopen.com/quacpac>
