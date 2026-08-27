# Major Project Continuation Handbook

## Treatment-Response Coupling in GRU-Based Sepsis Prediction

**Purpose:** This document is the restart guide for continuing the major project after the semester examinations. It begins at the point where Phase 1 (data engineering in BigQuery) is complete and continues through model training, controlled ablation, calibration, explainability, subgroup evaluation, backend/frontend integration, testing, report writing, and final defense preparation.

**Project boundary:** This is a retrospective research prototype. It is not a diagnostic device, a causal treatment-effect estimator, or a treatment-recommendation system.

**Current status date:** August 2026

---

## 1. The five-minute restart summary

When returning after the examinations, remember these five facts:

1. The raw MIMIC-IV data has already been transformed into three permanent, unnormalized, hourly BigQuery tables.
2. The patient-level train, validation, and test split is already fixed. Never create a new random split unless a verified data error forces reconstruction.
3. The next technical task is not more feature engineering. It is to verify and freeze the Phase 1 outputs, load one variant at a time, fit normalization parameters using training rows only, and build GRU sequences by ICU stay.
4. The primary experiment is a controlled comparison:
   - Variant A: physiology and laboratory information.
   - Variant B: Variant A information plus flat vasopressor treatment information.
   - Variant C: Variant A information plus dynamic treatment-physiology features.
5. The final system must report honest results. Variant C is being tested; it is not assumed to win.

The entire remaining pipeline is:

```text
BigQuery verification
    -> training-only normalization
    -> per-stay variable-length sequences
    -> identical GRU training for A, B, and C
    -> locked test evaluation
    -> select final model
    -> temperature scaling
    -> SHAP explanations
    -> subgroup evaluation
    -> model artifact packaging
    -> PostgreSQL/FastAPI integration
    -> React dashboard and model-behaviour what-if tool
    -> Docker testing
    -> final report and defense
```

---

## 2. Current completed state: Phase 1 handover

### 2.1 Cohort and labels already constructed

The current proposal records:

| Item | Frozen value |
|---|---:|
| Unique patients | 46,494 |
| ICU stays | 59,662 |
| Hourly observations | 4,169,659 |
| Positive hourly labels | 48,330 |
| Negative hourly labels | 4,121,329 |
| Approximate positive rate | 1.159% |
| Approximate negative-to-positive ratio | 85.27:1 |
| Training patients | 32,500 |
| Validation patients | 6,929 |
| Test patients | 7,065 |
| Prediction horizon | Sepsis onset within the next 6 hours |
| Minimum eligible stay duration | More than 24 hours |

A modeled hour is positive when Sepsis-3 onset occurs strictly after the current hour and within the following six hours. Post-onset measurements must not be used to predict that same onset.

### 2.2 Completed physiological and laboratory representation

Physiological variables:

- Heart rate
- Mean arterial pressure (MAP)
- SpO2
- Temperature
- Respiratory rate
- Glasgow Coma Scale (GCS)
- One current-hour missingness indicator for each variable

Laboratory variables:

- Lactate
- White blood cell count
- Creatinine
- Total bilirubin
- Platelet count
- One current-hour missingness indicator for each variable

Bounded Last Observation Carried Forward (LOCF):

- Heart rate, MAP, SpO2, respiratory rate: maximum 4 hours
- Temperature, GCS: maximum 6 hours
- Laboratory variables: maximum 24 hours

Training-derived residual-imputation medians currently recorded:

| Feature | Training median |
|---|---:|
| Heart rate | 83 |
| MAP | 79 |
| SpO2 | 97 |
| Temperature | 36.89 |
| Respiratory rate | 19 |
| GCS | 15 |
| Lactate | 1.5 |
| WBC | 10.4 |
| Creatinine | 0.9 |
| Bilirubin | 0.7 |
| Platelets | 198 |

Do not silently recompute these values from the combined dataset. If they must be recomputed, use training patients only and document why.

### 2.3 Completed treatment representation

Variant B flat treatment inputs:

- Hourly norepinephrine-equivalent vasopressor dose
- Vasopressor-active indicator

Variant C dynamic treatment-physiology inputs:

- Hour-to-hour change in norepinephrine-equivalent dose
- Hour-to-hour change in MAP
- Interaction = dose change multiplied by MAP change
- Antibiotic-intensification indicator

Important interpretation:

- The interaction is a predictive temporal association.
- It is not an estimate of vasopressor effectiveness.
- It is not an expected-versus-actual response equation.
- It is not causal.

Recorded Phase 1 validation values:

| Check | Frozen value |
|---|---:|
| Hours with nonzero vasopressor-dose change | 272,507 |
| Positive antibiotic-intensification hours | 55,879 |
| Hours with antibiotic treatment | 1,645,913 |
| ICU stays with antibiotic intensification | 24,037 |
| Maximum concurrent antibiotic count | 7 |
| Nulls among the four new Variant C features | 0 |

### 2.4 Exact variant rule

Keep this rule unchanged during the controlled ablation:

```text
Variant A = physiology/laboratories/missingness indicators

Variant B = Variant A
            + flat norepinephrine-equivalent dose
            + vasopressor-active indicator

Variant C = Variant A
            + delta MAP
            + delta norepinephrine-equivalent dose
            + delta-dose x delta-MAP interaction
            + antibiotic-intensification indicator
```

Variant C intentionally does not simultaneously include the flat dose and vasopressor-active fields from Variant B. Changing this rule would create a different experiment and would require updating the research question and report.

---

## 3. Fill this project-location record before the examinations

Complete this block and commit the document to the private project repository. Do not place passwords, service-account keys, or MIMIC-derived data in Git.

```yaml
BigQuery project: sepsis-gru-mimic
Dataset: sepsis_gru

Variant A: variant_a
Variant B: variant_b
Variant C: variant_c

Patient ID: subject_id
Stay ID: stay_id
Hour column: hour
Split column: dataset_split
Label column: sepsis_next_6h

Rows: 4,169,659
Patients: 46,494
ICU stays: 59,662
Positive hours: 48,330
Negative hours: 4,121,329

Split:
train = 32,500 patients
validation = 6,929
test = 7,065

Patient leakage = 0
A/B mismatch = 0
A/C mismatch = 0
Variant C dynamic nulls = 0
```

Also record:

- The Google account that can access the BigQuery project.
- Whether billing or BigQuery sandbox limitations apply.
- Whether the BigQuery Storage API is enabled.
- The exact table creation date.
- The SQL files that created each final table.
- The last successful row-count and leakage-check date.
- The person responsible for each remaining phase.

Keep `PROJECT_STATE.md` short and update it at the end of every work session:

```markdown
# Current Project State

Last updated: YYYY-MM-DD
Latest verified data manifest: FILL_ME
Latest Git commit: FILL_ME
Last completed task: FILL_ME
Current task: FILL_ME
Next exact command/notebook: FILL_ME
Known blocker: none/FILL_ME
Decision needed from supervisor: none/FILL_ME
Artifacts produced today: FILL_ME
```

---

## 4. Non-negotiable research rules

These rules protect the validity of the project.

### 4.1 Never mix patients across splits

All stays from one `subject_id` must remain in one split. Verify this again before training.

### 4.2 Never learn preprocessing parameters from validation or test data

The following must come from training data only:

- Residual imputation values
- Continuous-feature means and standard deviations
- Positive-class weight
- Hyperparameter decisions
- Early-stopping decisions
- Classification threshold
- Temperature-scaling parameter

### 4.3 Do not inspect test performance repeatedly

Use validation data during development. Open the test results only after:

- Feature lists are frozen.
- Hyperparameters are frozen.
- Random-seed protocol is frozen.
- Primary metric and model-selection rule are written down.
- Threshold-selection procedure is frozen.

### 4.4 Keep the three GRU experiments controlled

Across A, B, and C, keep identical:

- Cohort and labels
- Patient split
- Sequence construction
- Model architecture
- Hidden dimension
- Number of layers
- Dropout
- Optimizer
- Learning-rate schedule
- Batch size
- Epoch limit
- Early-stopping rule
- Loss function
- Seed list
- Evaluation code

Only the intended feature representation may differ.

### 4.5 Do not force the expected conclusion

Accept all three scientifically valid outcomes:

- C outperforms B: evidence supports added predictive value from explicit coupling.
- C and B are similar: no evidence that the explicit representation adds meaningful value under this design.
- C underperforms B: the engineered representation may discard useful treatment information or add noise.

### 4.6 Respect MIMIC-IV data-use restrictions

- Keep BigQuery datasets private.
- Never upload raw or processed patient-level data to GitHub.
- Never make a Cloud Storage bucket public.
- Never put credentials in source code.
- Display only de-identified IDs in the prototype.
- Store only the minimum data required for the research prototype.

---

## 5. Recommended repository structure

Create this structure before model development:

```text
sepsis-major-project/
├── README.md
├── PROJECT_STATE.md
├── requirements.in
├── requirements-lock.txt
├── .env.example
├── .gitignore
├── configs/
│   ├── data.yaml
│   ├── features.yaml
│   ├── model.yaml
│   └── experiment.yaml
├── sql/
│   ├── phase1/
│   ├── validation/
│   │   ├── 01_table_counts.sql
│   │   ├── 02_split_overlap.sql
│   │   ├── 03_label_distribution.sql
│   │   ├── 04_variant_consistency.sql
│   │   └── 05_feature_nulls.sql
│   └── serving/
├── notebooks/
│   ├── 01_phase1_handover.ipynb
│   ├── 02_data_loading.ipynb
│   ├── 03_gru_pilot.ipynb
│   ├── 04_ablation_results.ipynb
│   ├── 05_calibration.ipynb
│   ├── 06_shap.ipynb
│   └── 07_subgroup_evaluation.ipynb
├── src/
│   ├── data/
│   │   ├── bigquery_io.py
│   │   ├── normalization.py
│   │   ├── sequences.py
│   │   └── validation.py
│   ├── models/
│   │   ├── gru.py
│   │   └── calibration.py
│   ├── training/
│   │   ├── train.py
│   │   ├── evaluate.py
│   │   └── metrics.py
│   ├── explainability/
│   │   ├── shap_pipeline.py
│   │   └── subgroup_metrics.py
│   └── common/
│       ├── config.py
│       ├── seeds.py
│       └── logging.py
├── artifacts/
│   ├── scalers/
│   ├── checkpoints/
│   ├── metrics/
│   ├── plots/
│   └── explanations/
├── backend/
│   ├── app/
│   ├── tests/
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   ├── tests/
│   └── Dockerfile
├── database/
│   ├── schema.sql
│   └── seed_dashboard_subset.py
├── tests/
├── docker-compose.yml
└── report/
    ├── tables/
    ├── figures/
    └── notes/
```

Add this to `.gitignore`:

```gitignore
.env
*service-account*.json
credentials*.json
data/
*.csv
*.parquet
*.pkl
*.pt
*.pth
mlruns/
__pycache__/
.ipynb_checkpoints/
node_modules/
dist/
```

Model checkpoints may be stored in a private release location, but do not accidentally commit patient-derived tensors with them.

---

## 6. Phase 1 closure: verify BigQuery before modeling

Do this immediately after returning. Save every result as a screenshot and as a small non-sensitive summary file.

### 6.1 Table inventory

```sql
SELECT
  table_name,
  table_type,
  creation_time
FROM `PROJECT_ID.DATASET_ID.INFORMATION_SCHEMA.TABLES`
WHERE table_name IN (
  'VARIANT_A_TABLE',
  'VARIANT_B_TABLE',
  'VARIANT_C_TABLE'
)
ORDER BY table_name;
```

### 6.2 Counts for each variant

Run separately or combine with `UNION ALL`:

```sql
SELECT
  'A' AS variant,
  COUNT(*) AS hourly_rows,
  COUNT(DISTINCT subject_id) AS patients,
  COUNT(DISTINCT stay_id) AS stays
FROM `PROJECT_ID.DATASET_ID.VARIANT_A_TABLE`

UNION ALL

SELECT
  'B',
  COUNT(*),
  COUNT(DISTINCT subject_id),
  COUNT(DISTINCT stay_id)
FROM `PROJECT_ID.DATASET_ID.VARIANT_B_TABLE`

UNION ALL

SELECT
  'C',
  COUNT(*),
  COUNT(DISTINCT subject_id),
  COUNT(DISTINCT stay_id)
FROM `PROJECT_ID.DATASET_ID.VARIANT_C_TABLE`;
```

Expected: identical cohort counts for all variants.

### 6.3 Label distribution

```sql
SELECT
  split,
  sepsis_label,
  COUNT(*) AS hourly_rows,
  COUNT(DISTINCT subject_id) AS patients,
  COUNT(DISTINCT stay_id) AS stays
FROM `PROJECT_ID.DATASET_ID.VARIANT_C_TABLE`
GROUP BY split, sepsis_label
ORDER BY split, sepsis_label;
```

Use the actual split and label column names.

### 6.4 Patient leakage test

```sql
SELECT
  subject_id,
  COUNT(DISTINCT split) AS number_of_splits
FROM `PROJECT_ID.DATASET_ID.VARIANT_C_TABLE`
GROUP BY subject_id
HAVING COUNT(DISTINCT split) > 1;
```

Expected: zero rows.

### 6.5 Confirm that variants contain identical keys and labels

```sql
SELECT COUNT(*) AS mismatch_count
FROM `PROJECT_ID.DATASET_ID.VARIANT_A_TABLE` a
FULL OUTER JOIN `PROJECT_ID.DATASET_ID.VARIANT_B_TABLE` b
USING (subject_id, stay_id, hour_timestamp)
WHERE
  a.subject_id IS NULL
  OR b.subject_id IS NULL
  OR a.sepsis_label != b.sepsis_label;
```

Repeat for A versus C. Replace `hour_timestamp` with the actual hourly key.

Expected: zero mismatches.

### 6.6 Variant C null and event checks

```sql
SELECT
  COUNTIF(delta_map IS NULL) AS null_delta_map,
  COUNTIF(delta_dose IS NULL) AS null_delta_dose,
  COUNTIF(treatment_physiology_interaction IS NULL) AS null_interaction,
  COUNTIF(antibiotic_intensification IS NULL) AS null_antibiotic_intensification,
  COUNTIF(delta_dose != 0) AS nonzero_dose_change_hours,
  COUNTIF(antibiotic_intensification = 1) AS intensification_hours
FROM `PROJECT_ID.DATASET_ID.VARIANT_C_TABLE`;
```

### 6.7 Data freeze manifest

After the checks pass, create `artifacts/data_manifest.json` containing only metadata, not patient data:

```json
{
  "created_at": "YYYY-MM-DD",
  "bigquery_location": "FILL_ME",
  "variant_a_table": "FILL_ME",
  "variant_b_table": "FILL_ME",
  "variant_c_table": "FILL_ME",
  "rows": 4169659,
  "patients": 46494,
  "stays": 59662,
  "positive_hours": 48330,
  "negative_hours": 4121329,
  "split_method": "patient-level subject_id",
  "phase1_sql_git_commit": "FILL_ME",
  "status": "frozen"
}
```

**Phase 1 closure gate:** Do not begin model training until every check above passes or every difference has a written explanation.

---

## 7. Development environment

### 7.1 Suggested initial packages

Use a Google Colab GPU for training and a local environment for backend/frontend work.

```bash
pip install \
  google-cloud-bigquery \
  google-cloud-bigquery-storage \
  db-dtypes \
  pyarrow \
  pandas \
  numpy \
  scikit-learn \
  torch \
  shap \
  matplotlib \
  seaborn \
  mlflow \
  fastapi \
  uvicorn \
  sqlalchemy \
  psycopg2-binary \
  pydantic \
  pytest
```

After the first fully working environment:

```bash
pip freeze > requirements-lock.txt
```

Commit the lock file. Do not depend indefinitely on unpinned latest versions.

### 7.2 Configuration rather than hard-coded names

Example `configs/data.yaml`:

```yaml
project_id: YOUR_PROJECT
dataset_id: YOUR_DATASET
location: YOUR_LOCATION

tables:
  A: YOUR_VARIANT_A_TABLE
  B: YOUR_VARIANT_B_TABLE
  C: YOUR_VARIANT_C_TABLE

ids:
  patient: subject_id
  stay: stay_id
  hour: hour_timestamp
  label: sepsis_label
  split: split
```

### 7.3 Reproducibility helper

```python
import os
import random
import numpy as np
import torch

def set_seed(seed: int) -> None:
    os.environ["PYTHONHASHSEED"] = str(seed)
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)

    torch.backends.cudnn.benchmark = False
    torch.backends.cudnn.deterministic = True
```

Record:

- Python version
- PyTorch version
- CUDA version
- GPU type
- Package lock file
- Git commit
- BigQuery table IDs
- Seed

---

## 8. Phase 2A: load one BigQuery variant at a time

Do not load A, B, and C into memory simultaneously.

### 8.1 Authentication

In Colab, authenticate through the account that already has access:

```python
from google.colab import auth
auth.authenticate_user()
```

Then create the client:

```python
from google.cloud import bigquery

PROJECT_ID = "YOUR_PROJECT"
client = bigquery.Client(project=PROJECT_ID)
```

Do not upload a service-account JSON file to the notebook or repository unless the team has a formally controlled credential process.

### 8.2 Select only required columns

Define exact feature order. Model feature order must never depend on alphabetical sorting.

```python
COMMON_CONTINUOUS = [
    "heart_rate",
    "map",
    "spo2",
    "temperature",
    "respiratory_rate",
    "gcs",
    "lactate",
    "wbc",
    "creatinine",
    "bilirubin",
    "platelets",
]

MISSINGNESS_FEATURES = [
    "heart_rate_missing",
    "map_missing",
    "spo2_missing",
    "temperature_missing",
    "respiratory_rate_missing",
    "gcs_missing",
    "lactate_missing",
    "wbc_missing",
    "creatinine_missing",
    "bilirubin_missing",
    "platelets_missing",
]

VARIANT_FEATURES = {
    "A": COMMON_CONTINUOUS + MISSINGNESS_FEATURES,
    "B": COMMON_CONTINUOUS + MISSINGNESS_FEATURES + [
        "norepinephrine_equivalent_dose",
        "vasopressor_active",
    ],
    "C": COMMON_CONTINUOUS + MISSINGNESS_FEATURES + [
        "delta_map",
        "delta_dose",
        "treatment_physiology_interaction",
        "antibiotic_intensification",
    ],
}
```

Replace names with the exact BigQuery schema and save the mapping in `configs/features.yaml`.

### 8.3 Loading function

```python
from google.cloud import bigquery

ID_COLUMNS = [
    "subject_id",
    "stay_id",
    "hour_timestamp",
    "sepsis_label",
    "split",
]

def load_variant_split(
    client: bigquery.Client,
    full_table_id: str,
    feature_columns: list[str],
    split_name: str,
):
    columns = ID_COLUMNS + feature_columns
    selected = ",\n".join(f"`{c}`" for c in columns)

    sql = f"""
    SELECT
      {selected}
    FROM `{full_table_id}`
    WHERE split = @split_name
    ORDER BY stay_id, hour_timestamp
    """

    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter(
                "split_name", "STRING", split_name
            )
        ]
    )

    return (
        client.query(sql, job_config=job_config)
        .to_dataframe(create_bqstorage_client=True)
    )
```

The BigQuery Storage API can accelerate large query-result downloads. If RAM becomes insufficient:

1. Load one split at a time.
2. Save temporary Parquet files only in a private, access-controlled location.
3. Delete transient copies after experiments.
4. Do not place them in Git or a public bucket.
5. Consider reading sequences in chunks by `stay_id`.

### 8.4 Immediate assertions after loading

```python
def validate_loaded_frame(df, features):
    required = set(ID_COLUMNS + features)
    missing_columns = required - set(df.columns)
    assert not missing_columns, missing_columns

    assert df["subject_id"].notna().all()
    assert df["stay_id"].notna().all()
    assert df["hour_timestamp"].notna().all()
    assert df["sepsis_label"].isin([0, 1]).all()
    assert not df.duplicated(
        ["stay_id", "hour_timestamp"]
    ).any()

    assert df[features].notna().all().all()
```

If an assertion fails, stop. Do not "temporarily fill" the problem inside the training notebook without tracing it back to the Phase 1 pipeline.

---

## 9. Phase 2B: training-only normalization

### 9.1 What to standardize

Standardize continuous measurements and continuous treatment variables.

Normally do not standardize:

- Missingness indicators
- Vasopressor-active indicator
- Antibiotic-intensification indicator
- Label
- IDs
- Split field

For Variant C, `delta_map`, `delta_dose`, and the interaction are continuous and should normally be standardized.

### 9.2 Fit on training rows only

```python
from sklearn.preprocessing import StandardScaler
import joblib

def fit_and_apply_scaler(
    train_df,
    val_df,
    test_df,
    continuous_features,
    output_path,
):
    scaler = StandardScaler()

    train_df = train_df.copy()
    val_df = val_df.copy()
    test_df = test_df.copy()

    train_df[continuous_features] = scaler.fit_transform(
        train_df[continuous_features]
    )
    val_df[continuous_features] = scaler.transform(
        val_df[continuous_features]
    )
    test_df[continuous_features] = scaler.transform(
        test_df[continuous_features]
    )

    joblib.dump(scaler, output_path)
    return train_df, val_df, test_df
```

Important:

- Never call `fit_transform` on validation or test.
- Save one scaler per variant because feature sets differ.
- Save the exact feature order beside each scaler.
- The backend must reuse the final model's scaler exactly.

### 9.3 Normalization checks

After scaling:

```python
train_means = train_df[continuous_features].mean()
train_stds = train_df[continuous_features].std(ddof=0)

print(train_means.abs().sort_values(ascending=False).head())
print((train_stds - 1).abs().sort_values(ascending=False).head())
```

Expect training means near 0 and training standard deviations near 1. Validation and test means do not need to be zero.

---

## 10. Phase 2C: convert hourly rows into ICU-stay sequences

The GRU receives sequences, not independent rows.

For one ICU stay:

```text
X shape = (number of modeled hours, number of features)
y shape = (number of modeled hours)
```

### 10.1 Dataset class

```python
import numpy as np
import torch
from torch.utils.data import Dataset

class StaySequenceDataset(Dataset):
    def __init__(self, df, feature_columns):
        self.feature_columns = feature_columns
        self.groups = []

        df = df.sort_values(["stay_id", "hour_timestamp"])

        for stay_id, group in df.groupby("stay_id", sort=False):
            x = group[feature_columns].to_numpy(
                dtype=np.float32
            )
            y = group["sepsis_label"].to_numpy(
                dtype=np.float32
            )

            self.groups.append({
                "stay_id": int(stay_id),
                "subject_id": int(group["subject_id"].iloc[0]),
                "x": torch.from_numpy(x),
                "y": torch.from_numpy(y),
            })

    def __len__(self):
        return len(self.groups)

    def __getitem__(self, index):
        return self.groups[index]
```

If memory is too high, replace the in-memory list with:

- Parquet row groups partitioned by split/stay, or
- A BigQuery-to-Parquet preparation step followed by lazy per-stay reading.

Do not change the scientific logic merely to solve memory issues.

### 10.2 Padding and mask

```python
from torch.nn.utils.rnn import pad_sequence

def collate_stays(batch):
    xs = [item["x"] for item in batch]
    ys = [item["y"] for item in batch]

    lengths = torch.tensor(
        [len(x) for x in xs],
        dtype=torch.long,
    )

    x_padded = pad_sequence(
        xs,
        batch_first=True,
        padding_value=0.0,
    )

    y_padded = pad_sequence(
        ys,
        batch_first=True,
        padding_value=0.0,
    )

    max_length = x_padded.size(1)
    mask = (
        torch.arange(max_length)[None, :]
        < lengths[:, None]
    )

    return {
        "x": x_padded,
        "y": y_padded,
        "lengths": lengths,
        "mask": mask,
        "stay_id": [item["stay_id"] for item in batch],
        "subject_id": [item["subject_id"] for item in batch],
    }
```

The mask ensures padded hours do not contribute to loss or metrics.

---

## 11. Phase 2D: common GRU model

Use one common model class for all variants.

```python
import torch
from torch import nn
from torch.nn.utils.rnn import (
    pack_padded_sequence,
    pad_packed_sequence,
)

class HourlySepsisGRU(nn.Module):
    def __init__(
        self,
        input_dim,
        hidden_dim=64,
        num_layers=2,
        dropout=0.2,
    ):
        super().__init__()

        self.gru = nn.GRU(
            input_size=input_dim,
            hidden_size=hidden_dim,
            num_layers=num_layers,
            batch_first=True,
            dropout=dropout if num_layers > 1 else 0.0,
        )

        self.output = nn.Linear(hidden_dim, 1)

    def forward(self, x, lengths):
        packed = pack_padded_sequence(
            x,
            lengths.cpu(),
            batch_first=True,
            enforce_sorted=False,
        )

        packed_output, _ = self.gru(packed)

        padded_output, _ = pad_packed_sequence(
            packed_output,
            batch_first=True,
            total_length=x.size(1),
        )

        return self.output(padded_output).squeeze(-1)
```

Expected shapes:

```text
Input x:       [batch, time, features]
Lengths:       [batch]
Output logits: [batch, time]
Mask:          [batch, time]
Labels:        [batch, time]
```

Do not apply sigmoid inside the model during training. Use `BCEWithLogitsLoss`, which is numerically more stable than applying sigmoid and then binary cross-entropy separately.

---

## 12. Phase 2E: class imbalance and loss

Compute positive weight from training labels only.

```python
def calculate_pos_weight(train_df):
    positives = int(train_df["sepsis_label"].sum())
    negatives = len(train_df) - positives

    if positives == 0:
        raise ValueError("Training set contains no positive labels")

    return negatives / positives
```

Create masked loss:

```python
import torch
from torch import nn

def masked_bce_loss(logits, targets, mask, pos_weight):
    criterion = nn.BCEWithLogitsLoss(
        reduction="none",
        pos_weight=torch.tensor(
            [pos_weight],
            device=logits.device,
        ),
    )

    elementwise = criterion(logits, targets)
    masked = elementwise * mask.float()

    return masked.sum() / mask.sum().clamp_min(1)
```

Important:

- Record the training positive weight.
- Use the same weighting rule for all variants.
- Weighted training may worsen raw probability calibration. This is one reason calibration is evaluated after model selection.
- Do not use accuracy as the primary metric because the positive rate is near 1.16%.

---

## 13. Phase 2F: training loop

### 13.1 One epoch

```python
import torch

def train_one_epoch(
    model,
    loader,
    optimizer,
    device,
    pos_weight,
):
    model.train()
    total_loss = 0.0
    total_hours = 0

    for batch in loader:
        x = batch["x"].to(device)
        y = batch["y"].to(device)
        lengths = batch["lengths"].to(device)
        mask = batch["mask"].to(device)

        optimizer.zero_grad(set_to_none=True)

        logits = model(x, lengths)
        loss = masked_bce_loss(
            logits,
            y,
            mask,
            pos_weight,
        )

        loss.backward()
        torch.nn.utils.clip_grad_norm_(
            model.parameters(),
            max_norm=1.0,
        )
        optimizer.step()

        valid_hours = int(mask.sum().item())
        total_loss += loss.item() * valid_hours
        total_hours += valid_hours

    return total_loss / max(total_hours, 1)
```

### 13.2 Collect validation predictions

```python
import numpy as np
import torch

@torch.no_grad()
def collect_predictions(model, loader, device):
    model.eval()

    all_logits = []
    all_labels = []
    all_stay_ids = []
    all_subject_ids = []

    for batch in loader:
        x = batch["x"].to(device)
        lengths = batch["lengths"].to(device)
        mask = batch["mask"].to(device)

        logits = model(x, lengths)

        valid_logits = logits[mask]
        valid_labels = batch["y"].to(device)[mask]

        all_logits.append(valid_logits.cpu().numpy())
        all_labels.append(valid_labels.cpu().numpy())

        for i, length in enumerate(batch["lengths"].tolist()):
            all_stay_ids.extend(
                [batch["stay_id"][i]] * length
            )
            all_subject_ids.extend(
                [batch["subject_id"][i]] * length
            )

    return {
        "logits": np.concatenate(all_logits),
        "labels": np.concatenate(all_labels),
        "stay_id": np.asarray(all_stay_ids),
        "subject_id": np.asarray(all_subject_ids),
    }
```

### 13.3 Metrics

```python
import numpy as np
from sklearn.metrics import (
    roc_auc_score,
    average_precision_score,
    log_loss,
    brier_score_loss,
)

def sigmoid_np(logits):
    return 1.0 / (1.0 + np.exp(-logits))

def probability_metrics(labels, logits):
    probabilities = sigmoid_np(logits)

    return {
        "auroc": roc_auc_score(labels, probabilities),
        "auprc": average_precision_score(
            labels, probabilities
        ),
        "nll": log_loss(
            labels,
            probabilities,
            labels=[0, 1],
        ),
        "brier": brier_score_loss(
            labels,
            probabilities,
        ),
    }
```

### 13.4 Early stopping

Use validation AUPRC as the primary early-stopping measure because the event is rare. Freeze this choice before comparing variants.

Suggested pilot configuration:

```yaml
hidden_dim: 64
num_layers: 2
dropout: 0.2
optimizer: AdamW
learning_rate: 0.001
weight_decay: 0.0001
batch_size: 32
maximum_epochs: 30
early_stopping_patience: 5
gradient_clip_norm: 1.0
primary_validation_metric: AUPRC
```

These are starting values, not guaranteed final values.

Correct procedure:

1. Conduct a small pilot using training and validation data.
2. Select one feasible architecture/configuration.
3. Freeze it.
4. Apply it unchanged to all three variants.
5. Use the same seed set.

Recommended seed set:

```python
SEEDS = [42, 52, 62, 72, 82]
```

If computing resources are limited, use at least three seeds and explain the limitation.

---

## 14. Experiment tracking with MLflow

Log every run:

```python
import mlflow
import mlflow.pytorch

def log_experiment(
    variant,
    seed,
    config,
    metrics,
    model,
):
    with mlflow.start_run(
        run_name=f"variant_{variant}_seed_{seed}"
    ):
        mlflow.log_param("variant", variant)
        mlflow.log_param("seed", seed)
        mlflow.log_params(config)
        mlflow.log_metrics(metrics)
        mlflow.pytorch.log_model(
            model,
            artifact_path="model",
        )
```

Also log:

- Feature list and order
- Scaler path/hash
- BigQuery table ID
- Data-manifest hash
- Git commit
- Package lock hash
- Epoch selected by early stopping
- Training time
- GPU type
- Number of trainable parameters

Save the best checkpoint locally:

```python
torch.save(
    {
        "model_state_dict": model.state_dict(),
        "variant": variant,
        "seed": seed,
        "input_dim": len(feature_columns),
        "feature_columns": feature_columns,
        "model_config": model_config,
    },
    checkpoint_path,
)
```

---

## 15. Controlled ablation protocol

### 15.1 Development stage

For each seed:

1. Train A using the fixed configuration.
2. Train B using the same configuration.
3. Train C using the same configuration.
4. Save the best validation checkpoint for each.
5. Record validation AUROC, AUPRC, NLL, and Brier score.

### 15.2 Model-selection rule

Write this before viewing test results. Recommended:

```text
Primary selection criterion: mean validation AUPRC across seeds.
Secondary criterion: mean validation AUROC.
Tie-breaker: lower validation NLL, then simpler representation.
```

Do not select a different architecture for each variant. That would test architecture-plus-features rather than treatment representation.

### 15.3 Locked test stage

Once the protocol is frozen:

- Evaluate every variant/seed on the untouched test set.
- Report mean and standard deviation across seeds.
- Compare C versus B as the primary research comparison.
- Compare B versus A as the value of flat treatment information.
- Compare C versus A as the total value of dynamic treatment representation.

For the final prototype, choose one checkpoint using validation data only. A practical predeclared rule is: use the checkpoint from the selected variant and seed with the highest validation AUPRC. Keep the multi-seed mean and standard deviation as the scientific result, while the single validation-selected checkpoint is the deployable demonstration model. Do not choose the prototype checkpoint because it happened to obtain the highest test score.

Required result table:

| Variant | AUROC mean +/- SD | AUPRC mean +/- SD | NLL | Brier | Parameters |
|---|---:|---:|---:|---:|---:|
| A | TBD | TBD | TBD | TBD | Same model design |
| B | TBD | TBD | TBD | TBD | Same model design |
| C | TBD | TBD | TBD | TBD | Same model design |

### 15.4 Confidence intervals

Because multiple hourly rows come from one patient, do not treat all hours as independent when constructing uncertainty intervals. Prefer patient-level bootstrap:

1. Sample test `subject_id` values with replacement.
2. Include all their hourly rows.
3. Recompute the metric.
4. Repeat 1,000 times if feasible.
5. Report 2.5th and 97.5th percentiles.

Also bootstrap the paired difference `C - B` using the same sampled patients.

Basic patient-bootstrap helper:

```python
import numpy as np

def patient_bootstrap_metric(
    labels,
    probabilities,
    subject_ids,
    metric_fn,
    repetitions=1000,
    seed=42,
):
    rng = np.random.default_rng(seed)
    unique_subjects = np.unique(subject_ids)

    subject_indices = {
        subject: np.flatnonzero(subject_ids == subject)
        for subject in unique_subjects
    }

    values = []

    for _ in range(repetitions):
        sampled_subjects = rng.choice(
            unique_subjects,
            size=len(unique_subjects),
            replace=True,
        )

        sampled_indices = np.concatenate([
            subject_indices[subject]
            for subject in sampled_subjects
        ])

        sampled_labels = labels[sampled_indices]
        if np.unique(sampled_labels).size < 2:
            continue

        values.append(metric_fn(
            sampled_labels,
            probabilities[sampled_indices],
        ))

    values = np.asarray(values)

    return {
        "estimate": metric_fn(labels, probabilities),
        "lower_95": np.percentile(values, 2.5),
        "upper_95": np.percentile(values, 97.5),
        "bootstrap_values": values,
    }
```

For a paired C-versus-B comparison, use the same resampled subject list for both variants and store the metric difference within each repetition.

### 15.5 Honest interpretation

Do not write:

> Variant C proves treatment effectiveness.

Write:

> Under the same cohort, architecture, and training procedure, Variant C produced [higher/similar/lower] predictive discrimination than Variant B. This result concerns the predictive value of the engineered representation and does not establish a causal treatment effect.

### 15.6 Optional logistic-regression sanity baseline

The proposal's software requirements mention a logistic-regression baseline, although the primary methodology is the three-GRU ablation. If time permits, train a simple hourly logistic-regression model as a secondary sanity check. Use training-only normalization and the same patient splits, but label it clearly as a non-sequential baseline. It must not replace the A/B/C GRU comparison. Decide with the supervisor whether this baseline belongs in the final scope before adding it to the report.

---

## 16. Threshold selection and clinical classification metrics

AUROC and AUPRC are threshold-independent. The dashboard may still require a low/medium/high risk display.

Choose the threshold using validation data only.

Possible predefined rule:

- Maximize validation F1, or
- Maximize Youden's J, or
- Choose a threshold that achieves a target sensitivity.

Do not choose the threshold from the test set.

```python
import numpy as np
from sklearn.metrics import (
    precision_recall_curve,
    confusion_matrix,
)

def choose_f1_threshold(labels, probabilities):
    precision, recall, thresholds = (
        precision_recall_curve(labels, probabilities)
    )

    f1 = (
        2 * precision[:-1] * recall[:-1]
        / (precision[:-1] + recall[:-1] + 1e-12)
    )

    best_index = int(np.nanargmax(f1))
    return float(thresholds[best_index])

def threshold_metrics(labels, probabilities, threshold):
    predictions = (probabilities >= threshold).astype(int)

    tn, fp, fn, tp = confusion_matrix(
        labels,
        predictions,
        labels=[0, 1],
    ).ravel()

    return {
        "sensitivity": tp / max(tp + fn, 1),
        "specificity": tn / max(tn + fp, 1),
        "precision": tp / max(tp + fp, 1),
        "npv": tn / max(tn + fn, 1),
        "f1": 2 * tp / max(2 * tp + fp + fn, 1),
    }
```

Also report the number of alerts per 100 patient-hours at the chosen threshold. This makes alert burden easier to understand.

---

## 17. Phase 3: probability calibration

Only the selected final GRU is calibrated.

### 17.1 Temperature scaling rule

1. Freeze trained model weights.
2. Collect validation logits.
3. Fit one positive scalar temperature using validation labels.
4. Apply the same temperature to test logits.
5. Compare test calibration before and after scaling.
6. Do not refit the temperature on test data.

### 17.2 PyTorch temperature scaler

```python
import torch
from torch import nn

class TemperatureScaler(nn.Module):
    def __init__(self):
        super().__init__()
        self.log_temperature = nn.Parameter(
            torch.zeros(1)
        )

    @property
    def temperature(self):
        return self.log_temperature.exp()

    def forward(self, logits):
        return logits / self.temperature

def fit_temperature(validation_logits, validation_labels):
    scaler = TemperatureScaler()

    logits = torch.tensor(
        validation_logits,
        dtype=torch.float32,
    )
    labels = torch.tensor(
        validation_labels,
        dtype=torch.float32,
    )

    criterion = nn.BCEWithLogitsLoss()
    optimizer = torch.optim.LBFGS(
        scaler.parameters(),
        lr=0.1,
        max_iter=100,
    )

    def closure():
        optimizer.zero_grad()
        loss = criterion(scaler(logits), labels)
        loss.backward()
        return loss

    optimizer.step(closure)
    return scaler
```

### 17.3 Expected Calibration Error with ten equal-width bins

```python
import numpy as np

def expected_calibration_error(
    labels,
    probabilities,
    n_bins=10,
):
    boundaries = np.linspace(0.0, 1.0, n_bins + 1)
    ece = 0.0
    rows = []

    for lower, upper in zip(
        boundaries[:-1],
        boundaries[1:],
    ):
        if upper == 1.0:
            in_bin = (
                (probabilities >= lower)
                & (probabilities <= upper)
            )
        else:
            in_bin = (
                (probabilities >= lower)
                & (probabilities < upper)
            )

        count = int(in_bin.sum())
        if count == 0:
            continue

        confidence = float(probabilities[in_bin].mean())
        observed_rate = float(labels[in_bin].mean())
        weight = count / len(labels)

        ece += weight * abs(confidence - observed_rate)
        rows.append({
            "lower": lower,
            "upper": upper,
            "count": count,
            "mean_prediction": confidence,
            "observed_rate": observed_rate,
        })

    return ece, rows
```

Required calibration outputs:

- ECE before and after
- NLL before and after
- Brier score before and after
- Reliability diagram before and after
- Learned temperature
- AUROC and AUPRC before and after

Expected behavior:

- Temperature scaling should change probabilities.
- It should usually improve NLL/ECE if validation and test distributions are similar.
- It should not materially change AUROC because division by a positive temperature preserves ranking.
- Improvement is not guaranteed; report the actual result.

---

## 18. Phase 4: SHAP-based explainability

### 18.1 Freeze the explanation target

For a time-series model, "explain the patient" is incomplete. Define:

- Which prediction hour is explained?
- How many preceding hours are included?
- Is the displayed attribution per feature, per hour, or aggregated across time?

Recommended dashboard definition:

```text
Explain the calibrated risk at a selected target hour using a fixed
24-hour window ending at that hour. Display:
1. the top feature contributions at the target hour,
2. aggregated absolute contribution over the 24-hour window,
3. a temporal heatmap for advanced inspection.
```

A fixed window makes SHAP input shapes consistent. For the first implementation, explain only target hours with at least 24 valid preceding hours. This avoids treating artificial padding as real clinical history. Add masked shorter-history explanations only after verifying that the explanation wrapper respects the mask.

### 18.2 Explanation wrapper

```python
import torch
from torch import nn

class LastStepRiskWrapper(nn.Module):
    def __init__(self, gru_model, temperature=1.0):
        super().__init__()
        self.gru_model = gru_model
        self.temperature = float(temperature)

    def forward(self, x):
        lengths = torch.full(
            (x.size(0),),
            x.size(1),
            dtype=torch.long,
            device=x.device,
        )

        logits = self.gru_model(x, lengths)
        last_logits = logits[:, -1]

        return (
            last_logits / self.temperature
        ).unsqueeze(-1)
```

### 18.3 SHAP explainer

Start with a small, representative background sample from training windows.

```python
import shap

wrapper = LastStepRiskWrapper(
    gru_model,
    temperature=learned_temperature,
).to(device)

wrapper.eval()

background = training_windows[
    background_indices
].to(device)

explain_batch = test_windows[
    explanation_indices
].to(device)

explainer = shap.GradientExplainer(
    wrapper,
    background,
)

shap_values = explainer.shap_values(
    explain_batch
)
```

The proposal names DeepExplainer. In practice, recurrent models, packed sequences, custom masks, and changing SHAP versions may create compatibility issues. Use this protocol:

1. First test `shap.DeepExplainer` on 10 fixed-length windows.
2. Verify output shape and approximate additivity.
3. If it fails or produces unstable values, use `shap.GradientExplainer`.
4. Document the final explainer and version.
5. Do not switch silently.

The wrapper above explains the calibrated logit. Therefore, SHAP values should be described as contributions on the calibrated-logit scale. Do not present an individual SHAP value as a direct percentage-point change in sepsis probability.

### 18.4 Feasibility gate

The proposal says SHAP values will be precomputed for all test patients at every timestep. This may be computationally expensive.

Before promising full precomputation:

1. Benchmark 100 patients.
2. Record GPU/CPU time, memory, and output size.
3. Extrapolate to the entire test set.
4. If infeasible, obtain supervisor approval to:
   - precompute selected dashboard patients and clinically relevant hours, or
   - compute explanations on demand and cache them.

Do not quietly reduce the scope after results are due.

### 18.5 SHAP checks

For each explained batch:

- Confirm shape matches `[samples, time, features]`.
- Confirm feature order matches the saved model configuration.
- Check for NaN/infinite values.
- Check approximate additivity where supported.
- Never describe attribution as a causal effect.
- State that correlated clinical variables can share or redistribute attribution.

Required figures:

- Global mean absolute SHAP feature ranking
- Temporal SHAP heatmap
- One positive patient waterfall/bar explanation
- One negative patient explanation
- One false-positive explanation
- One false-negative explanation

---

## 19. Phase 5: subgroup and fairness evaluation

Evaluate the final selected model across:

- Sex groups available in MIMIC-IV
- Prespecified age groups
- Insurance groups, if retained and ethically justified

Recommended age bins:

```text
18-39
40-64
65-79
80+
```

For each subgroup report:

- Number of patients
- Number of ICU stays
- Positive hours
- Negative hours
- AUROC
- AUPRC
- Sensitivity and specificity at the frozen validation threshold
- Calibration error
- Brier score
- 95% patient-bootstrap confidence interval

Skeleton:

```python
from sklearn.metrics import (
    roc_auc_score,
    average_precision_score,
)

def subgroup_report(
    frame,
    group_column,
    probability_column,
    label_column,
):
    rows = []

    for group, part in frame.groupby(group_column):
        positives = int(part[label_column].sum())
        negatives = len(part) - positives

        row = {
            "group": group,
            "patients": part["subject_id"].nunique(),
            "hours": len(part),
            "positives": positives,
            "negatives": negatives,
        }

        if positives > 0 and negatives > 0:
            row["auroc"] = roc_auc_score(
                part[label_column],
                part[probability_column],
            )
            row["auprc"] = average_precision_score(
                part[label_column],
                part[probability_column],
            )
        else:
            row["auroc"] = None
            row["auprc"] = None

        rows.append(row)

    return rows
```

Interpretation rules:

- A difference in subgroup performance is an audit signal, not proof of discrimination.
- Small subgroups produce unstable estimates.
- Always show group sample size and event count.
- Insurance is not a direct measurement of race or socioeconomic status.
- Do not claim fairness because aggregate AUROC is high.

---

## 20. Phase 6: package the final model

The backend needs a complete inference bundle, not only `model.pt`.

Required artifacts:

```text
final_model/
├── checkpoint.pt
├── model_config.json
├── feature_config.json
├── scaler.joblib
├── temperature_scaler.pt
├── threshold.json
├── metrics.json
├── data_manifest.json
├── package_versions.txt
└── README.md
```

`feature_config.json` must contain:

```json
{
  "variant": "C",
  "feature_order": ["FILL_ME"],
  "continuous_features": ["FILL_ME"],
  "binary_features": ["FILL_ME"],
  "sequence_window_for_explanation": 24,
  "label_definition": "onset strictly after current hour and within six hours"
}
```

Before packaging, test that reloading the bundle produces identical logits to the original notebook within a small numerical tolerance.

---

## 21. Phase 7: practical PostgreSQL design

Recommended architecture:

- BigQuery remains the source of truth for the full research dataset.
- PostgreSQL stores only the de-identified test/dashboard subset, predictions, explanations, and model metadata required by the prototype.
- If the proposal is interpreted as requiring the entire processed cohort in PostgreSQL, discuss the duplication and privacy cost with the supervisor before implementing it.

Example schema:

```sql
CREATE TABLE patients (
    subject_id BIGINT PRIMARY KEY,
    sex TEXT,
    age_group TEXT,
    insurance_group TEXT
);

CREATE TABLE stays (
    stay_id BIGINT PRIMARY KEY,
    subject_id BIGINT NOT NULL REFERENCES patients(subject_id),
    split TEXT NOT NULL,
    onset_hour TIMESTAMP NULL
);

CREATE TABLE hourly_predictions (
    stay_id BIGINT NOT NULL REFERENCES stays(stay_id),
    hour_timestamp TIMESTAMP NOT NULL,
    raw_probability DOUBLE PRECISION NOT NULL,
    calibrated_probability DOUBLE PRECISION NOT NULL,
    risk_category TEXT NOT NULL,
    PRIMARY KEY (stay_id, hour_timestamp)
);

CREATE TABLE shap_values (
    stay_id BIGINT NOT NULL REFERENCES stays(stay_id),
    hour_timestamp TIMESTAMP NOT NULL,
    feature_name TEXT NOT NULL,
    feature_value DOUBLE PRECISION,
    shap_value DOUBLE PRECISION NOT NULL,
    PRIMARY KEY (
        stay_id,
        hour_timestamp,
        feature_name
    )
);

CREATE TABLE model_metadata (
    model_version TEXT PRIMARY KEY,
    variant TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    metrics JSONB NOT NULL,
    feature_config JSONB NOT NULL,
    disclaimer TEXT NOT NULL
);

CREATE INDEX idx_predictions_stay_hour
ON hourly_predictions(stay_id, hour_timestamp);

CREATE INDEX idx_shap_stay_hour
ON shap_values(stay_id, hour_timestamp);
```

Do not store real names, medical record numbers, addresses, or any new identifying information.

---

## 22. Phase 8: FastAPI backend

### 22.1 Minimum endpoints

```text
GET  /health
GET  /model/metrics
GET  /patients
GET  /patients/{stay_id}/trajectory
GET  /patients/{stay_id}/explanations?hour=...
POST /what-if
```

### 22.2 Request model

```python
from pydantic import BaseModel, Field

class WhatIfRequest(BaseModel):
    stay_id: int
    hour_timestamp: str
    modified_delta_dose: float = Field(
        ge=-10.0,
        le=10.0,
    )
    modified_antibiotic_intensification: int = Field(
        ge=0,
        le=1,
    )
```

The numeric bounds above are placeholders. Replace them with defensible ranges derived from training data and clinical plausibility. Record the chosen ranges.

### 22.3 Endpoint skeleton

```python
from fastapi import FastAPI, HTTPException

app = FastAPI(
    title="Sepsis Model Research Prototype",
)

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/what-if")
def what_if(request: WhatIfRequest):
    try:
        result = run_model_behaviour_simulation(
            stay_id=request.stay_id,
            hour_timestamp=request.hour_timestamp,
            modified_delta_dose=(
                request.modified_delta_dose
            ),
            modified_antibiotic_intensification=(
                request.modified_antibiotic_intensification
            ),
        )
    except KeyError as error:
        raise HTTPException(
            status_code=404,
            detail=str(error),
        )

    return {
        "stay_id": request.stay_id,
        "hour_timestamp": request.hour_timestamp,
        "original_probability": result[
            "original_probability"
        ],
        "modified_probability": result[
            "modified_probability"
        ],
        "disclaimer": (
            "Research model-behaviour simulation only. "
            "This result is not a causal treatment effect "
            "or treatment recommendation."
        ),
    }
```

### 22.4 What-if inference logic

The simulator must:

1. Retrieve the historical input window.
2. Copy it; never overwrite stored history.
3. Modify only allowed fields.
4. Recompute every dependent feature.
5. Preserve exact feature order.
6. Apply the saved scaler.
7. Run the frozen model.
8. Apply the saved temperature.
9. Return original and modified model outputs.
10. Log no identifying data.

For Variant C, changing treatment information may require recomputing:

```text
delta dose
interaction = modified delta dose x existing delta MAP
```

If changing the absolute dose rather than `delta_dose`, recompute consecutive deltas consistently. Do not let the UI change one value while leaving dependent features mathematically inconsistent.

Use the phrase **model-behaviour simulation** in the interface. The term counterfactual can incorrectly imply a causal estimate.

### 22.5 Performance target

The proposal specifies a response target below 500 ms. Measure this after model warm-up:

- Median latency
- 95th percentile latency
- Maximum observed latency
- Number of requests
- Hardware used

Do not claim the target is achieved without measurement.

---

## 23. Phase 9: React dashboard

Required screens:

### 23.1 Patient/stay list

Display:

- De-identified stay ID
- Age group, not exact age if not needed
- Sex group
- Available timeline range
- Outcome/onset marker only if appropriate for retrospective review

### 23.2 Risk trajectory

Display:

- Calibrated risk over time
- Sepsis-onset marker
- Selected alert threshold
- Optional vasopressor-dose/MAP overlay
- Clear units and legend

### 23.3 Explanation view

Display:

- Selected prediction hour
- Top positive and negative SHAP contributions
- Feature value and unit
- A statement that attribution is associative, not causal

### 23.4 What-if view

Display:

- Original values
- Allowed input controls
- Original probability
- Modified model probability
- Difference in model output
- Persistent disclaimer

Never display:

- "Recommended dose"
- "Expected patient response"
- "Give this medication"
- "The treatment will reduce risk by X%"

### 23.5 Frontend API pattern

```javascript
async function runWhatIf(payload) {
  const response = await fetch(
    "http://localhost:8000/what-if",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify(payload)
    }
  );

  if (!response.ok) {
    throw new Error("What-if request failed");
  }

  return response.json();
}
```

Keep the UI usable without internet after Docker images and dependencies are prepared.

---

## 24. Phase 10: Docker integration

Example `docker-compose.yml`:

```yaml
services:
  database:
    image: postgres:16
    environment:
      POSTGRES_DB: sepsis_research
      POSTGRES_USER: sepsis_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    environment:
      DATABASE_URL: ${DATABASE_URL}
    depends_on:
      - database
    ports:
      - "8000:8000"

  frontend:
    build: ./frontend
    depends_on:
      - backend
    ports:
      - "3000:3000"

volumes:
  postgres_data:
```

Keep secrets in `.env`, which is ignored by Git. Provide `.env.example` with placeholder values only.

Integration test:

```text
docker compose up --build
GET http://localhost:8000/health
Open http://localhost:3000
Select one stay
Open trajectory
Open explanation
Run one what-if request
Restart containers
Confirm stored data remains available
```

---

## 25. Testing plan

### 25.1 Data tests

- All three variants have identical keys and labels.
- No patient appears in multiple splits.
- No post-onset row is used for onset prediction.
- No model feature is null.
- Feature ranges are plausible.
- Variant C first-hour deltas equal zero.
- Variant B contains flat treatment fields.
- Variant C contains dynamic fields and excludes Variant B-only fields.

### 25.2 Model tests

- Forward output shape is `[batch, time]`.
- Padding does not affect valid-hour logits.
- Padded hours do not contribute to loss.
- The same input produces the same output in evaluation mode.
- Saved and reloaded checkpoints reproduce logits.
- All variants use the same common configuration.
- Validation metrics are computed only on valid masked hours.

### 25.3 Calibration tests

- Temperature is positive.
- Test calibration uses validation-fitted temperature.
- AUROC is unchanged within numerical tolerance.
- No probability is NaN or outside `[0, 1]`.

### 25.4 API tests

- Unknown stay returns 404.
- Invalid input range returns 422.
- Valid request returns original and modified probabilities.
- Response includes disclaimer.
- Repeated identical requests return identical values.
- Dependent treatment features are recomputed.

### 25.5 UI tests

- Patient list loads.
- Empty/error states display clearly.
- Timeline labels and units are visible.
- SHAP chart matches selected hour.
- What-if controls cannot exceed allowed ranges.
- Disclaimer remains visible.
- No treatment recommendation language appears.

### 25.6 End-to-end scientific test

Select one frozen test stay and store:

- Raw historical input window
- Normalized tensor
- Original logit
- Calibrated probability
- Top SHAP values
- API response
- UI screenshot

All stages must agree on the patient, hour, feature order, and probability.

---

## 26. Required result tables and figures

### 26.1 Tables

1. Final cohort and split summary
2. Feature list for A, B, and C
3. Missingness and imputation summary
4. Ablation metrics across seeds
5. Pairwise metric differences with confidence intervals
6. Calibration before/after
7. Threshold-dependent metrics
8. Subgroup performance with support counts
9. Model and inference runtime
10. Software/hardware configuration

### 26.2 Figures

1. Final end-to-end pipeline
2. Class distribution
3. Example patient MAP/dose trajectory
4. Training and validation loss curves
5. ROC curves for A/B/C
6. Precision-recall curves for A/B/C
7. Reliability diagram before/after scaling
8. Global SHAP summary
9. Patient-level SHAP explanation
10. Subgroup metric plot with confidence intervals
11. Full system architecture
12. Dashboard screenshots

### 26.3 Keep report language conditional until results exist

Before results:

> The study will evaluate whether Variant C improves prediction.

After results:

> Variant C achieved [result] relative to Variant B under the controlled protocol.

Never write "proved" unless the evidence and statistical analysis genuinely support the exact claim.

### 26.4 Final report update sequence

When experiments finish, update the report in this order:

1. **Methodology:** replace planned language with the exact implemented feature names, package versions, hyperparameters, seeds, explainer, threshold rule, and SHAP scope.
2. **Results:** present cohort verification, ablation results, uncertainty intervals, calibration, threshold metrics, SHAP outputs, subgroup results, latency, and system tests.
3. **Discussion:** answer the research question directly, compare findings with the reviewed literature, explain unexpected outcomes, and separate predictive associations from causal claims.
4. **Limitations:** include single-center retrospective data, label uncertainty, missingness, class imbalance, treatment confounding, lack of external/prospective validation, SHAP limitations, and prototype-only status.
5. **Conclusion:** state only what was supported by the completed experiment.
6. **Appendices:** include feature dictionary, final configuration, API contract, additional tables, and reproducibility commands without exposing patient-level data or credentials.

---

## 27. Expected outcomes: what each result would mean

### Scenario A: Variant C clearly improves AUPRC over B

Interpretation:

- Explicit dynamic treatment-physiology representation added predictive information under the controlled setup.
- Report paired patient-bootstrap uncertainty.
- Do not claim causal treatment response.

### Scenario B: C and B are nearly identical

Interpretation:

- The GRU may already learn relevant interactions from flat features, or the explicit feature may not add measurable information.
- This is still a valid answer to the research question.
- Discuss sample, feature design, and architecture limitations.

### Scenario C: C performs worse

Interpretation:

- Excluding the absolute dose/active indicator may remove useful context.
- Delta/product features may amplify noise.
- The engineered interaction may be too simple.
- Do not alter the primary experiment after seeing the test result.
- Additional combined variants may be labeled exploratory only.

### Calibration result

Expected:

- ECE/NLL may improve.
- AUROC should remain essentially unchanged.
- If test calibration worsens, report possible validation-test shift or overfitting of the calibration step.

### SHAP result

Expected:

- The method should identify influential physiological and treatment-related features.
- Feature importance does not prove clinical causality.
- Correlated features may share importance.

### Subgroup result

Expected:

- Some variation across groups is normal.
- Wide confidence intervals may indicate insufficient subgroup events.
- Report uncertainty rather than ranking groups as "fair" or "unfair" from one metric.

---

## 28. Suggested ten-week continuation schedule

### Week 0: Restart and verification

- Restore accounts and repository.
- Fill the location record.
- Run all BigQuery closure queries.
- Freeze manifest.
- Re-read proposal methodology.
- Confirm responsibilities.

**Output:** signed-off Phase 1 handover.

### Week 1: Data loading and normalization

- Finalize feature configuration.
- Load each split safely.
- Fit training-only scalers.
- Build per-stay sequences.
- Validate masks and labels.

**Output:** reproducible DataLoaders for A/B/C.

### Week 2: GRU pilot

- Implement common model.
- Run overfit test on a tiny subset.
- Run one full pilot.
- Fix architecture and training configuration.
- Set MLflow tracking.

**Output:** frozen experiment configuration.

### Weeks 3-4: Controlled ablation

- Train all variants across the fixed seeds.
- Apply early stopping.
- Save checkpoints and logs.
- Lock model selection.
- Run test evaluation once.

**Output:** ablation table, curves, checkpoints.

### Week 5: Calibration and statistical analysis

- Fit temperature on validation logits.
- Evaluate test calibration.
- Generate reliability diagrams.
- Run patient-level bootstrap.
- Freeze threshold.

**Output:** calibrated final inference bundle.

### Week 6: Explainability and subgroup evaluation

- Benchmark SHAP feasibility.
- Generate agreed explanation scope.
- Compute subgroup tables and intervals.
- Write limitations immediately.

**Output:** SHAP artifacts and subgroup report.

### Week 7: Database and backend

- Create PostgreSQL schema.
- Load dashboard subset.
- Build FastAPI endpoints.
- Implement exact inference/scaling/calibration.
- Add tests and latency measurement.

**Output:** tested API.

### Week 8: Frontend

- Build patient list.
- Build risk chart.
- Build explanation chart.
- Build what-if view.
- Add persistent boundaries/disclaimer.

**Output:** working React interface.

### Week 9: Integration and Docker

- Connect all services.
- Run end-to-end test.
- Repair feature-order or API mismatches.
- Create offline demo backup.
- Capture screenshots/video.

**Output:** reproducible local prototype.

### Week 10: Report and final defense

- Update methodology to actual implementation.
- Insert real results.
- Write discussion and limitations.
- Finalize figures/tables.
- Prepare slides, script, questions, and demo backup.

**Output:** report, source code, prototype, final presentation.

Add buffer weeks if the academic calendar allows them. Do not compress scientific validation merely to add visual dashboard features.

---

## 29. Suggested four-member division

### Member 1: Data handover and modeling

- BigQuery closure checks
- Data loading
- Normalization
- Sequence dataset
- GRU implementation

### Member 2: Evaluation and responsible AI

- Metrics and bootstrap
- Calibration
- SHAP
- Subgroup evaluation
- Result tables

### Member 3: Backend and database

- Model bundle loading
- PostgreSQL schema
- FastAPI endpoints
- What-if inference consistency
- Backend tests

### Member 4: Frontend, integration, and documentation

- React views
- Charts
- Docker integration
- Report figures
- Demo and presentation assets

Team rules:

- At least two members review every scientific result.
- All members must understand A/B/C and the two research gaps.
- No component should be understood by only one member.
- Merge code through branches and review, not file sharing through chat.
- Hold one weekly integration meeting with written decisions.

---

## 30. The first two hours after examinations

Follow this exact sequence:

### First 15 minutes

- Open this handbook.
- Open the private repository.
- Open BigQuery Studio.
- Confirm the three tables still exist.
- Fill missing project-location values.

### Next 30 minutes

- Run table counts.
- Run patient leakage query.
- Run label distribution query.
- Run Variant C null/event query.
- Compare with frozen proposal counts.

### Next 30 minutes

- Create/activate the Python environment.
- Authenticate BigQuery.
- Load 1,000 ordered rows from Variant A.
- Validate schema and feature names.
- Update `features.yaml` with exact names.

### Final 45 minutes

- Load a small group of complete stays.
- Build `StaySequenceDataset`.
- Run the collate function.
- Print tensor shapes.
- Run one GRU forward pass.
- Record the next action in `PROJECT_STATE.md`.

At the end of the session, commit code and metadata only.

---

## 31. Common failure modes and recovery

### "The three tables have different row counts"

Stop training. Compare key coverage and joins. Do not drop rows independently by variant.

### "The model reports excellent accuracy"

Ignore accuracy initially. With approximately 98.84% negative hours, a trivial negative predictor looks accurate. Examine AUPRC, sensitivity, specificity, and alert rate.

### "Training loss becomes NaN"

Check:

- Infinite or extreme feature values
- Incorrect scaling
- Excessive positive weight
- Learning rate
- Gradient norm
- NaN logits
- Incorrect padding mask

Reduce learning rate only after data checks.

### "GPU runs out of memory"

Try in this order:

1. Reduce batch size.
2. Bucket stays by similar sequence length.
3. Use mixed precision after correctness is verified.
4. Load one split/variant at a time.
5. Use lazy Parquet sequence loading.
6. Reduce hidden dimension only through the frozen shared configuration.

### "Variant C wins only for one seed"

Report mean, standard deviation, and paired uncertainty. Do not highlight a single favorable seed.

### "Temperature scaling does not improve ECE"

Check fitting code and validation logits. If correct, report the result. Do not refit on test.

### "SHAP crashes on the GRU"

Use fixed-length windows, remove packed-sequence behavior from the explanation wrapper, test small batches, and use GradientExplainer if DeepExplainer is incompatible. Record the decision.

### "The what-if result looks clinically strange"

The tool shows model behavior, not patient response. Verify feature scaling and dependent-feature recomputation. Keep the disclaimer visible.

### "The dashboard takes too long"

Prioritize:

1. Correct experiment
2. Calibration/evaluation
3. Correct backend inference
4. Minimal functional dashboard
5. Visual polish

A scientifically valid simple dashboard is better than a polished invalid experiment.

---

## 32. Changes that require supervisor approval

Obtain written approval before:

- Changing the six-hour label definition
- Rebuilding patient splits
- Adding post-onset observations
- Including flat dose inside Variant C
- Selecting different architectures for each variant
- Reducing the planned SHAP population/scope
- Removing subgroup evaluation
- Replacing SHAP with a different explanation family
- Turning model-behaviour simulation into treatment recommendation
- Changing the primary metric
- Using test results for model selection
- Publicly hosting any MIMIC-derived data

---

## 33. Final definition of done

The project is complete only when all relevant boxes are checked.

### Data

- [ ] A/B/C tables verified and frozen
- [ ] Patient leakage check passes
- [ ] Label counts verified
- [ ] Feature lists/version recorded
- [ ] Training-only scaler saved
- [ ] Data-use requirements followed

### Modeling

- [ ] Common GRU implementation tested
- [ ] Common hyperparameters frozen
- [ ] At least three fixed seeds run
- [ ] Test set used only after lock
- [ ] A/B/C comparison reported
- [ ] Checkpoints reproducibly reload

### Evaluation

- [ ] AUROC and AUPRC reported
- [ ] NLL and Brier score reported
- [ ] Threshold selected on validation
- [ ] Sensitivity/specificity/precision reported
- [ ] Patient-bootstrap intervals reported
- [ ] Conclusions match observed results

### Responsible AI

- [ ] Temperature fitted on validation only
- [ ] ECE/reliability before and after reported
- [ ] SHAP method and scope documented
- [ ] SHAP limitations stated
- [ ] Subgroup support and metrics reported
- [ ] No causal or treatment claims made

### System

- [ ] Final inference bundle includes scaler and feature order
- [ ] PostgreSQL schema works
- [ ] API endpoints tested
- [ ] What-if dependent features recomputed
- [ ] Latency measured
- [ ] React views work
- [ ] Persistent disclaimer visible
- [ ] Docker demo survives restart

### Documentation and defense

- [ ] Report methodology matches actual code
- [ ] Real results replace expected-results language
- [ ] Figures/tables reproducible
- [ ] Git repository organized
- [ ] README includes exact setup
- [ ] Demo backup video/screenshots available
- [ ] All four members can explain the full pipeline

---

## 34. Official technical references

Use these when APIs change after the examination break:

- BigQuery Python libraries and Storage API:
  https://docs.cloud.google.com/bigquery/docs/python-libraries
- BigQuery Studio:
  https://docs.cloud.google.com/bigquery/docs/bigquery-web-ui
- PyTorch GRU:
  https://docs.pytorch.org/docs/stable/generated/torch.nn.GRU.html
- PyTorch packed sequences:
  https://docs.pytorch.org/docs/stable/generated/torch.nn.utils.rnn.pack_padded_sequence.html
- PyTorch BCEWithLogitsLoss:
  https://docs.pytorch.org/docs/stable/generated/torch.nn.BCEWithLogitsLoss.html
- SHAP GradientExplainer:
  https://shap.readthedocs.io/en/latest/generated/shap.GradientExplainer.html
- SHAP DeepExplainer:
  https://shap.readthedocs.io/en/latest/generated/shap.DeepExplainer.html
- FastAPI request bodies:
  https://fastapi.tiangolo.com/tutorial/body/
- FastAPI response models:
  https://fastapi.tiangolo.com/tutorial/response-model/

---

## 35. Final reminder

The purpose of the project is not to manufacture a positive result. It is to perform a controlled and reproducible test of whether explicit treatment-physiology representation adds predictive value, and then to evaluate and communicate the selected model responsibly.

If the result does not match the original expectation, the project has not failed. A carefully controlled negative or neutral result is still a valid research finding. The project fails only if the experimental controls are broken, the test set is repeatedly used for development, the reported conclusion exceeds the evidence, or the prototype is presented as a clinical treatment system.

The next action after the examinations is:

> Verify the frozen BigQuery tables, freeze the experiment configuration, and obtain one correct end-to-end GRU training run on Variant A before attempting the complete ablation.
