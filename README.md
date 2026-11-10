# IEEE-CIS Fraud Detection

A data mining course project investigating how model choice, transaction
features, identity information, and feature engineering affect online
transaction fraud detection.

## Project overview

We compare logistic regression and CatBoost using the IEEE-CIS Fraud
Detection dataset. The project includes exploratory analysis, chronological
validation, threshold selection, final holdout evaluation, feature importance,
and error analysis.

This project uses local evaluation only. No Kaggle submission was made.

## Dataset

Download the dataset from:

https://www.kaggle.com/competitions/ieee-fraud-detection/data

The analysis requires:

- `train_transaction.csv`
- `train_identity.csv`

Place these files inside the `data/` folder. Raw dataset files are not included
in this repository. Obtain access through Kaggle and follow the dataset's
applicable terms.

The labeled dataset contains 590,540 transactions, including 20,663 fraudulent
transactions (approximately 3.50%). Identity information is available for
approximately 24.42% of transactions.

The tables are joined on `TransactionID` using a left join, preserving
transactions without identity records.

The Kaggle test files and sample submission are not required for this analysis.

## Project structure

```text
ieee-fraud-project/
├── README.md
├── data/
│   ├── train_transaction.csv
│   └── train_identity.csv
├── notebooks/
│   ├── 01_experiments_documented.ipynb
│   └── 02_saved_model_evaluation.ipynb
├── models/
│   ├── selected_catboost.cbm
│   └── selected_model_metadata.json
└── results/
    ├── validation_experiments.csv
    ├── holdout_results.csv
    ├── holdout_predictions.csv
    ├── report_model_comparison.csv
    ├── feature_importance.csv
    ├── holdout_errors_by_product.csv
    ├── environment_versions.json
    └── figures/
        ├── holdout_confusion_matrix.png
        ├── holdout_roc_curve.png
        ├── holdout_precision_recall_curve.png
        └── feature_importance_top20.png
```

The `data/` directory is populated locally after downloading the dataset.
The saved model and metadata must be available to run evaluation without
retraining.

## Environment setup

The recorded environment used Python 3.12.10 with:

| Package | Version |
|---|---|
| pandas | 2.2.3 |
| NumPy | 2.2.4 |
| scikit-learn | 1.8.0 |
| CatBoost | 1.2.10 |
| Matplotlib | 3.10.6 |

Create an environment using Python 3.12:

```bash
python -m venv .venv
```

Activate it on Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

Or on macOS/Linux:

```bash
source .venv/bin/activate
```

Install the dependencies:

```bash
python -m pip install pandas==2.2.3 numpy==2.2.4 scikit-learn==1.8.0 catboost==1.2.10 matplotlib==3.10.6 ipykernel
```

In VS Code, install the Python and Jupyter extensions, open the project folder,
and select this environment as the notebook kernel.

## Configure the project path

The notebooks currently contain the original author's local project path.
Before execution, update every `PROJECT_DIR` definition to your own project
folder.

For example:

```python
from pathlib import Path

PROJECT_DIR = Path("C:/path/to/ieee-fraud-project")
```

If your notebook's working directory is the `notebooks/` folder, you can
instead use:

```python
PROJECT_DIR = Path.cwd().parent
assert (PROJECT_DIR / "data").is_dir()
```

## Running the notebooks

### Review the completed experiments

Open:

```text
notebooks/01_experiments_documented.ipynb
```

This notebook documents data exploration, preprocessing, model training,
validation comparisons, and model selection.

Historical outputs are preserved for review. Execution counts were cleared
when the notebook was organized; those outputs do not represent a fresh
end-to-end execution of the organized copy.

Training is disabled by default:

```python
RUN_TRAINING = False
```

The first code cell deliberately stops execution unless this flag is changed.
There is no need to retrain models to review the existing results.

To deliberately reproduce all experiments, set the flag to `True` and execute
the notebook in order. Training can take several hours. This workflow writes
model and result files, so use a separate project copy if preserving the
original artifacts.

### Evaluate the saved model without training

Open:

```text
notebooks/02_saved_model_evaluation.ipynb
```

This notebook:

1. Loads the saved CatBoost model and metadata.
2. Reconstructs the original chronological holdout.
3. Prepares features using the recorded column definitions.
4. Generates predictions.
5. Evaluates the previously selected threshold.
6. Saves metrics, figures, feature importance, and category-level errors.

It contains no model training.

Required files include the two training CSVs, both files in `models/`, and
`results/validation_experiments.csv`.

The CSVs require substantial memory when loaded. Reading data and generating
predictions may take time even though no model is being fitted.

## Evaluation design

Transactions are sorted by `TransactionDT` and split chronologically:

| Partition | Transactions | Fraud cases |
|---|---:|---:|
| Training | 413,378 | 14,538 |
| Validation | 88,581 | 3,042 |
| Holdout | 88,581 | 3,083 |

Transactions with the same boundary time are kept in the same partition.

- Preprocessing is learned from training data.
- Validation ROC-AUC is used for CatBoost early stopping and model selection.
- Classification thresholds are selected by maximum validation F1.
- The selected model and threshold are frozen before holdout evaluation.
- Holdout error analysis is descriptive and is not used for further tuning.

## Experiments

The project evaluates:

1. A dummy baseline.
2. Logistic regression with 17 selected transaction features.
3. CatBoost with the same selected features.
4. CatBoost with all 392 transaction features.
5. CatBoost with transaction and identity features, totaling 432 inputs.
6. CatBoost with three additional engineered features.
7. A longer training budget for the transaction-plus-identity model.

The engineered features are log transaction amount, transaction missing-field
count, and training-derived product-category frequency.

The largest validation improvement came from expanding the transaction
feature set. Identity information provided a modest additional improvement.
The engineered-feature group did not improve the primary validation metrics.

## Selected model

The selected model is CatBoost with transaction and identity features.

- Maximum training budget: 2,000 iterations
- Trees retained: 1,999
- Tree depth: 6
- Learning rate: 0.05
- Training loss: Logloss
- Early-stopping metric: validation AUC
- Early-stopping patience: 100 iterations
- Random seed: 42
- CPU threads: 4
- Frozen classification threshold: approximately 0.173891

The exact threshold and feature definitions are stored in
`models/selected_model_metadata.json`.

## Results

### Validation comparisons

Each model's threshold-dependent metrics use its own validation-F1-selected
threshold.

| Model | ROC-AUC | Average precision | F1 |
|---|---:|---:|---:|
| Logistic regression, selected features | 0.7972 | 0.1685 | 0.2554 |
| CatBoost, selected features | 0.8239 | 0.2185 | 0.2988 |
| CatBoost, all transaction features | 0.9270 | 0.6248 | 0.6043 |
| CatBoost, transaction + identity | 0.9300 | 0.6363 | 0.6093 |
| CatBoost, combined + engineered features | 0.9289 | 0.6351 | 0.6045 |
| CatBoost, combined, longer budget | 0.9334 | 0.6505 | 0.6186 |

These are validation selection results, not independent final performance
estimates.

### Final holdout results

| Metric | Selected CatBoost |
|---|---:|
| ROC-AUC | 0.9096 |
| Average precision | 0.5621 |
| Precision | 0.5683 |
| Recall | 0.5196 |
| F1 | 0.5429 |
| Fraud caught | 1,602 |
| Fraud missed | 1,481 |
| False alarms | 1,217 |
| Legitimate transactions correctly cleared | 84,281 |

Final holdout metrics are reported only for the selected CatBoost model.
Baseline comparisons are limited to validation results.

Category W accounted for 970 of the 1,481 missed fraud cases, identifying an
important limitation of the model at the selected threshold.

## Interpretation and limitations

- Accuracy alone is misleading because fraud is rare.
- Validation performance was higher than final holdout performance.
- Results come from one chronological split and one random seed.
- Chronological separation does not ensure disjoint customer or card entities.
- Feature importance describes model behavior, not causation.
- Several features are anonymized, limiting domain interpretation.
- No business-specific error costs or probability calibration were evaluated.
- No Kaggle leaderboard performance or production readiness is claimed.

## References

- [IEEE-CIS Fraud Detection dataset](https://www.kaggle.com/competitions/ieee-fraud-detection/data)
- [CatBoost: Unbiased Boosting with Categorical Features](https://proceedings.neurips.cc/paper/2018/hash/14491b756b3a51daac41c24863285549-Abstract.html)
- [The Relationship Between Precision-Recall and ROC Curves](https://doi.org/10.1145/1143844.1143874)
- [CatBoost feature importance documentation](https://catboost.ai/docs/en/concepts/fstr)

## Contributors

- [Add team member names and contributions]
