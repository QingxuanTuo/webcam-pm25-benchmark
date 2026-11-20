# Nanjing workflow replicability evaluation

This directory contains the Nanjing evaluation reported in the journal manuscript. Models are trained locally using the image-derived descriptor definitions and fixed regression configurations implemented for Milan. The experiment assesses workflow replicability and consistency of findings across camera and urban settings, rather than direct transfer of Milan-fitted model parameters. It does not include CNN representations or local ARPA meteorological observations.

## Setup and inputs

Run [`01_nanjing_external_validation_cv_holdout.ipynb`](01_nanjing_external_validation_cv_holdout.ipynb) from top to bottom, with the working directory set to either the repository root or `external validation/`. The notebook imports the shared `src` package. Install the root dependencies and start Jupyter from the repository root:

```bash
python -m pip install -r requirements.txt
python -m jupyter lab
```

Prepare this input layout:

```text
external validation/
├── 01_nanjing_external_validation_cv_holdout.ipynb
├── Nanjing/
│   ├── NJ_224/                       # Supplied 224 × 224 JPEG images
│   ├── NJ_Image_AQI.xlsx              # Image identifiers, dates and PM2.5 labels
│   └── nanjing_era5_hourly.csv        # Hourly ERA5 predictors, UTC timestamps
└── results/                          # Created automatically
```

The source image/label dataset is available at [Mendeley Data, version 1](https://doi.org/10.17632/3hnztk9fk8.1). Use its Nanjing fixed-surveillance-camera subset and arrange the inputs as above. The spreadsheet must contain `Image Name`, `Year`, `Month`, `Day`, `Hour`, and `PM2.5`. JPEG filename stems must match the 12-digit identifiers extracted from `Image Name`.

To download ERA5 through the notebook, configure CDS credentials, accept the relevant dataset terms, and set `DOWNLOAD_ERA5 = True`. The configured coordinates are 32.11613° N, 118.91002° E, with pressure-level area `[32.25, 118.75, 32.00, 119.00]` in north/west/south/east order. Leave the flag `False` when the ERA5 CSV is already available.

ERA5 is optional for a reduced image-plus-timestamp demonstration, but **reproducing the manuscript results requires all 18 configured ERA5 variables**. Otherwise, the notebook uses whichever configured ERA5 columns are available, or skips ERA5 if the file is absent; these reduced runs are not equivalent to the manuscript experiment.

## Processing

1. Link labels to images using the spreadsheet date components and image identifiers. Retain PM2.5 values in the implemented 0–500 µg/m³ validity range; remove missing/invalid targets, missing image links and duplicate timestamps. AQI, AQL and PM10 are not predictors.
2. Extract the same 18 image-derived descriptors used in Milan from each complete supplied 224 × 224 image, without a further ROI crop. Screen non-finite descriptors, flat images and invalid descriptor ranges.
3. Construct 12 timestamp-derived predictors. Calendar variables and date groups use Nanjing local time (`Asia/Shanghai`); convert timestamps to UTC for ERA5 matching. The daytime indicator uses local hours 06–18 inclusive.
4. Remove ERA5 cloud-base height (`CBH`), screen physical ranges and non-finite values, and retain complete records with valid hourly ERA5 matches. ERA5 values are not imputed in this notebook.

Image descriptors are cached at `results/nanjing_handcrafted_features.csv`, relative to this directory. Remove the cache before rerunning extraction if the source images or descriptor implementation change.

With the manuscript inputs, the final dataset contains **2,687 observations from 132 dates**, with 48 predictors: 18 image-derived descriptors, 12 timestamp-derived predictors and 18 ERA5 predictors. Timestamp and PM2.5 target columns are additional to the predictors.

## Evaluation protocol

- **Fixed partition:** allocate approximately 80%/20% of complete dates within each observed year–month to development/common hold-out sets, using split seed `24`. The manuscript partition contains 2,133 development observations from 104 dates and 554 hold-out observations from 28 dates.
- **Development CV:** use five-fold `GroupKFold`, grouped by local calendar date. All observations from a date remain in one fold. The model random state is `42`.
- **Predictor combinations:** compare Image, Image + Temporal, and Image + Temporal + ERA5 using the shared Ridge and Random Forest configurations. When ERA5 is present, all combinations use the same final complete-case dataset and partitions.
- **Model comparison:** compare Ridge, Decision Tree, Random Forest, Gradient Boosting and MLP using the richest available predictor set. Import configurations unchanged from `src.models.benchmark`; rank models primarily by mean development-CV R², with RMSE and MAE as supporting measures. Refit each fixed configuration on development data for common hold-out evaluation. The hold-out is not used for tuning or model selection.
- **Repeated-split sensitivity:** refit the fixed Gradient Boosting configuration over 50 year–month-balanced, date-disjoint splits, using split seeds `0`–`49`. Only date allocation changes; predictor set, model settings and 20% test proportion remain fixed. This analysis does not select models. Empirical percentiles describe variation across date allocations, rather than a formal population confidence interval.

MAE and RMSE are in µg/m³, R² is dimensionless, and MAPE, SMAPE and mean-normalised NRMSE are percentages. Metric conventions are implemented in `src.features.evaluation.evaluate_regression`. CV statistics are calculated across folds; repeated-split summaries use sample standard deviations across splits.

## Outputs and reference results

Outputs are written to **`external validation/results/nanjing_cv_holdout/`**, relative to the repository root. Older results directories are not the reference outputs for this notebook.

| File | Contents |
| --- | --- |
| `nanjing_final_dataset.csv` | Final analysis-ready observations and predictors |
| `quality_control_summary.csv` | Screening and matching counts |
| `split_summary.csv` | Dataset, development and common hold-out sizes |
| `notebook3_external_validation.csv` | Ridge/Random Forest predictor-combination CV and hold-out metrics |
| `notebook4_external_validation.csv` | Five-model CV and hold-out comparison |
| `gradient_boosting_repeated_holdout_results.csv` | Six metrics and sample counts for each of the 50 split seeds |
| `gradient_boosting_repeated_holdout_summary.csv` | Repeated-split means, sample SDs, medians and empirical percentiles |
| `best_model_holdout_predictions.csv` | Hold-out predictions and residuals for the CV-selected model |
| `best_model_concentration_summary.csv` | Additional concentration-range diagnostic |

The notebook also produces diagnostic figures. Reference results are:

- Fixed-split Gradient Boosting: hold-out MAE **8.822 µg/m³**, RMSE **12.056 µg/m³**, R² **0.598**.
- Across 50 splits: mean hold-out R² **0.638 ± 0.097** (sample SD).

Exact numerical agreement may depend on library versions and numerical implementations. The root `requirements.txt` lists dependencies but does not pin an exact environment.

## Data sources and licensing

The Nanjing images and PM2.5 labels originate from Xiaochu Wang (2026), *Air Quality Estimation from Sequential Surveillance Images Using a Unified CNN-RNN Framework*, version 1, Mendeley Data, [doi:10.17632/3hnztk9fk8.1](https://doi.org/10.17632/3hnztk9fk8.1), licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

ERA5 data were obtained from the Copernicus Climate Change Service (C3S), Climate Data Store: [ERA5 hourly data on single levels](https://doi.org/10.24381/cds.adbb2d47) and [ERA5 hourly data on pressure levels](https://doi.org/10.24381/cds.bd0915c6), under CC BY 4.0.

The processed ERA5 CSV contains location-specific data and derived meteorological variables. The final analytical dataset combines screened Nanjing PM2.5 labels, image-derived descriptors computed for this study, timestamp-derived predictors and matched ERA5 variables. These processing steps were performed by the authors of this repository. No endorsement by the original data providers is implied.

The repository's MIT licence applies to the project code. Third-party data and their adapted portions retain their applicable CC BY 4.0 terms and attribution requirements. When redistributing these data, retain the source credits and licence links above and indicate any further changes. Obtain the original images and labels through the linked dataset where they are not distributed with this directory.
