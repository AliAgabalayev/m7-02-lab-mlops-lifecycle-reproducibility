# ETA Model Lifecycle

```mermaid
flowchart TD
    A[Raw Data\nS3 Bucket] -->|dataset_hash SHA-256| B[Feature Pipeline\nProcessed Dataset]
    B -->|dataset_uri + dataset_hash| C[Training Job\ndocker_image_digest + git_sha]
    E[Experiment Config\nrun_id + hyperparams] -->|config.yaml| C
    C -->|model.onnx + run_id| D[Model Artifact\nS3 URI]
    D -->|model_uri| F[Evaluation Job\nFrozen Holdout Set]
    F -->|eval_report.json + metrics| G[Auto Gate Check\nMAE + latency]
    G -->|PASS — automatic| H[Registry: Staging\nstage=staging]
    G -->|FAIL| X[Rejected\nno promotion]
    H -->|manual — ML Lead + Ops| I[Registry: Production\nstage=production]
    H -->|superseded by newer version| J[Registry: Archived\nstage=archived]
    I -->|model_uri + version tag| K[Canary Deployment\n10% traffic — 24h]
    K -->|canary_metrics OK — manual approval| L[Production Deployment\n100% traffic]
    K -->|canary FAIL — rollback| H
    L -->|deployed_version| M[Prediction Service\n~200 RPS online inference]
    M -->|prediction_distribution| N[Drift Monitor\nPSI on output]
    N -->|drift_signal > threshold| O[Retrain Trigger\nscheduled alert]
    O -->|new training cycle| B
    I -->|new version promoted to prod| J
```

## Transition Legend

| Transition | Type |
|---|---|
| Feature Pipeline → Training Job | Automatic |
| Evaluation → Registry Staging | Automatic (gate must pass) |
| Staging → Production | **Manual** (ML Lead + Ops approval) |
| Production → Canary Deployment | Automatic |
| Canary → Production Deployment | **Manual** (canary metrics review) |
| Canary FAIL → Rollback to Staging | Automatic |
| Production → Archived | Automatic (on new version promotion) |
| Drift Monitor → Retrain Trigger | Automatic (threshold breach) |

## Artifacts on Each Arrow

| Arrow | Artifact |
|---|---|
| Raw Data → Feature Pipeline | `dataset_hash` (SHA-256) |
| Feature Pipeline → Training Job | `dataset_uri`, `dataset_hash` |
| Experiment Config → Training Job | `run_id`, `git_sha`, `config.yaml` |
| Training Job → Model Artifact | `model.onnx`, `run_id` |
| Model Artifact → Evaluation Job | `model_uri` |
| Evaluation Job → Gate Check | `eval_report.json`, `mae`, `p95_latency_ms` |
| Gate Check → Staging | `model_uri`, `eval_report_uri` |
| Staging → Production | `approved_version`, `approver_id` |
| Production → Canary | `model_uri`, `version_tag` |
| Canary → Production | `canary_report.json` |
| Prediction Service → Drift Monitor | `prediction_distribution` (rolling 1h window) |
| Drift Monitor → Retrain Trigger | `drift_signal`, `psi_score` |