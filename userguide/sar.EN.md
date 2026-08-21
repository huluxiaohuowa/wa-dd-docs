> [中文文档](sar.md)

# SAR / Structure-Activity Relationship

## Role

The SAR module combines activity data, molecular structures, docking/FEP results, and computed properties for medicinal chemistry decisions and the next round of design.

Design feedback / failure attribution is the closed-loop analysis entry inside SAR. It combines candidate ligands, interaction profiles, docking results, FEP results, and optional pocket context into one evidence table, attributes the current round's failure modes, and produces constraints for the next design round.

## Inputs

- SMILES/SDF ligand assets
- activity table
- docking or FEP result assets
- interaction profile assets
- pocket assets

Candidate ligands, interaction profiles, docking results, and FEP results use the same grouped asset picker as ligand processing. Users can expand assets by source, select multiple entries, and submit evidence from several rounds in one run. The pocket asset is a single optional input used to preserve structural context in the report.

Recommended input set:

- Candidate ligand assets: generated, screened, or FEP-annotated SDF assets from the current round.
- Interaction profiles: profiles from interaction analysis, or docking interaction backfill results.
- Docking results: WA-DD Docking or Uni-Dock output assets.
- FEP results: FEP analysis outputs with ligand-level or edge-level relative energy, error, and ranking evidence.
- Required interactions: semicolon-separated labels, for example `ASP189 hydrogen bond; SER195 proximity`.

## Outputs

- SAR table
- R-group/MMPA views
- next-design candidates

Design feedback creates a `design_feedback` asset with these standard files:

| File | Purpose |
| --- | --- |
| `design_feedback_report.json` | Full structured report with input assets, thresholds, candidate evidence, failure modes, constraints, and recommended candidates. |
| `candidate_evidence.csv` | Candidate-level evidence table. Each row represents a collapsed ligand candidate with docking, FEP, interaction, and recommendation fields. |
| `failure_modes.csv` | Failure-mode table. A candidate may have multiple labels for current-round triage. |
| `next_round_constraints.json` | Next-round design constraints, including required interactions, failure modes to avoid, optimization priorities, and thresholds. |
| `recommended_candidates.sdf` | Candidate structures recommended for the next round. This file is omitted when no candidate passes the recommendation rules. |

## Failure Attribution Rules

The algorithm fuses evidence around ligand candidates. If a candidate has multiple poses or multiple external evidence rows, rows are collapsed by ligand name while keeping the best available docking score, FEP value, and interaction support.

Built-in failure modes include:

- `weak_docking_score`: docking score does not pass the configured threshold.
- `fep_unfavorable`: FEP prediction is unfavorable and exceeds the bad-result threshold.
- `fep_uncertain`: FEP error exceeds the uncertainty threshold.
- `missing_required_interaction`: one or more required interactions are missing.
- `low_interaction_support`: interaction evidence count is below the minimum count.
- `series_outlier`: the candidate is inconsistent with its series trend.
- `insufficient_evidence`: docking, FEP, or interaction evidence is not sufficient.

A recommended candidate must satisfy:

- Docking score passes the threshold, or docking evidence is absent while other evidence supports the candidate.
- FEP does not exceed the bad-result threshold and error does not exceed the uncertainty threshold.
- Required interactions and the minimum interaction count are satisfied.
- No critical failure mode remains.

## Constraint Feedback

`next_round_constraints.json` is designed for the next generation, screening, or medicinal chemistry design round. It turns attribution results into actionable constraints:

- Preserve required interactions that are already matched.
- Send `required_interactions` back for candidates missing key interactions.
- Tighten shape complementarity, pocket occupancy, and docking score requirements when docking is weak.
- Feed energy and error thresholds back when FEP is unfavorable or uncertain.
- Flag series outliers for per-series review instead of merging them into one global rank.

## Verification Metrics

A completed design feedback run should satisfy these checks:

- The job type is `design_feedback_analysis` and the job status is `completed`.
- The output asset has `source_type` set to `design_feedback` and metadata schema `wa_dd.design_feedback.v1`.
- `candidate_evidence.csv` contains at least `candidate_name`, `docking_score`, `fep_delta_g`, `interaction_count`, and `recommended`.
- `failure_modes.csv` contains traceable `candidate_name` and `failure_modes` fields.
- `next_round_constraints.json` uses schema `wa_dd.next_round_constraints.v1`.
- If at least one candidate passes, `recommended_candidates.sdf` can be downloaded and reused as a downstream ligand asset.
- The frontend asset picker supports grouped expansion and multi-select; it should not fall back to raw multi-select boxes.

## API / Automation

Design feedback API:

```http
POST /api/v1/design-feedback
```

Request fields:

| Field | Description |
| --- | --- |
| `project_id` | Current project ID. |
| `ligand_asset_ids` | List of candidate ligand asset IDs. |
| `interaction_profile_asset_ids` | List of interaction profile asset IDs. |
| `docking_asset_ids` | List of docking result asset IDs. |
| `fep_asset_ids` | List of FEP result asset IDs. |
| `pocket_asset_id` | Optional pocket asset ID. |
| `required_interactions` | Required interactions to preserve. |
| `name` | Output asset name, defaults to `design_feedback`. |
| `docking_good_threshold` | Docking recommendation threshold, default `-8.0`. |
| `fep_bad_threshold` | FEP bad-result threshold, default `1.0`. |
| `fep_error_threshold` | FEP uncertainty threshold, default `2.0`. |
| `min_interaction_count` | Minimum interaction evidence count, default `1`. |

The response is a Job object. After completion, download the standard files from the output asset.

The current implementation runs directly in the Web/API process and does not add a worker. This path fits lightweight evidence fusion and asset generation. A separate worker should only be added later if the workflow grows into long-running model inference, bulk recomputation, or queued scheduling.
