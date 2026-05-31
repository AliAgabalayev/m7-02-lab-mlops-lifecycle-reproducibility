# ADR 0001: Reproducibility Strategy for NorthStar Models

## Context

NorthStar Logistics currently has no model registry: models are stored in S3 with informal names like `eta_v2_FINAL.onnx`, with no record of which code, dataset version, or environment produced them. This makes it impossible to audit a production model, reproduce a training run, or safely roll back after a regression.

## Decision

**Environment.** Every training job runs inside a Docker image whose exact `sha256` digest is recorded in the model lineage. Images are built from a pinned `Dockerfile` committed to git and published to an internal container registry (ECR). No "install at runtime" — all dependencies are baked into the image at build time.

**Data.** Each training and evaluation dataset is written to S3 as an immutable, versioned object. A SHA-256 hash of the dataset manifest (file list + S3 ETags) is computed at write time and stored as the `dataset_hash` lineage field. The frozen evaluation split is committed once per model and never overwritten; its S3 URI and hash are gates for every staging promotion.

**Code.** Every training job is triggered from a tagged git commit. The `git_sha` (full 40-character hash, not a branch name) is recorded in the model lineage at job start. No training job may be promoted to staging if `git_sha` is absent or points to a dirty working tree.

**Randomness.** All random seeds (Python `random`, NumPy, framework-level) are set explicitly from a single `seed` field in `config.yaml`, which is versioned in git. The seed value is logged in the training run metadata and stored in the model lineage.

## Alternatives Rejected

- **S3 timestamps as dataset versioning** — timestamps are mutable (object can be overwritten silently) and do not detect partial writes or concurrent modifications; SHA-256 hashing is strictly stronger at negligible cost.
- **conda environment files instead of Docker** — conda envs do not capture system-level libraries (BLAS, CUDA drivers) and diverge across developer machines; Docker image digests are byte-for-byte reproducible across any host.
- **Branch name instead of git SHA in lineage** — branch heads move; a branch name recorded today points to a different commit tomorrow, making the lineage record meaningless for future audits.

## Consequences

- The ML team must maintain a Docker image build pipeline (CI job) and an internal container registry; image builds add ~5 minutes to the training kick-off time.
- Every dataset write must compute and store a SHA-256 manifest hash; the data pipeline team owns this step and must not skip it for large datasets.
- All training jobs must be launched via the approved CI trigger (not ad-hoc from a developer laptop) to guarantee that `git_sha` is clean and the Docker image digest is recorded.

## Revisit If

- Training data exceeds ~10 TB per run, at which point manifest hashing overhead becomes non-trivial and a dedicated dataset versioning tool (e.g. DeltaLake with built-in versioning, or LakeFormation with governed tables) should replace the SHA-256 convention.