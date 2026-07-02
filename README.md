# DeepLearning Zone MPPT

This repository is my working notebook for exploring a hybrid maximum power point tracking (MPPT) method for photovoltaic (PV) systems, especially when partial shading makes the power-voltage curve harder to track.

The main idea is simple: instead of asking a controller to blindly chase the nearest peak, the notebook takes a small 12-point scan of the PV curve, uses a deep learning model to guess the best MPP region, and then lets a local Perturb and Observe (P&O) step refine the final operating point.

I wrote this as an engineer trying to understand the behavior of the method, not as a polished black-box package. The notebook includes the data cleaning, model training, evaluation, plots, and notes that helped me reason through what the MPPT controller is doing.

## Why this project exists

Solar PV curves are usually straightforward when the panel has uniform irradiance. Under partial shading, though, the P-V curve can have more than one peak. A traditional MPPT method can sometimes lock onto a local maximum and miss the real global maximum power point.

This project experiments with a hybrid approach:

1. **Sample the PV curve sparsely** using only 12 voltage points.
2. **Use a Physics-Informed TCNformer model** to predict the likely MPP zone from those sparse measurements.
3. **Start P&O near that predicted zone** so the local search has a better chance of refining toward the global MPP.
4. **Compare the result** against the true dense-curve MPP and a coarse 12-point baseline.

The goal is not just to get a number at the end. The goal is to understand whether the model is actually helping the MPPT process start in the right region when the curve has multiple peaks.

## Repository contents

```text
.
├── README.md
└── TCN_with_P&O_For_Partial_Shading_MPPT.ipynb
```

The notebook is the main project file. It contains the full pipeline from dataset loading through final evaluation.

## What the notebook does

The notebook walks through the MPPT workflow step by step:

- Loads PV curve data from `.mat` or `.npz` files.
- Extracts voltage-current or voltage-power curves from different dataset layouts.
- Cleans and validates PV curves so the endpoints and interpolation behave realistically.
- Converts direct P-V data into pseudo I-V form when needed, so the same downstream pipeline can still use `P = V × I`.
- Builds a 12-point sparse scan feature set.
- Trains a Physics-Informed TCNformer model to predict:
  - the MPP voltage zone,
  - the offset inside that zone,
  - and a useful P&O step size.
- Applies local P&O refinement after the neural prediction.
- Evaluates the seed-only prediction, the refined hybrid result, and the coarse 12-point baseline.
- Plots example shaded curves with markers for the true MPP, neural seed, refined point, and sparse samples.
- Optionally saves the trained model bundle.
- Runs inference on a separate reduced PV dataset, `PV_Data_Reduced.mat`, when available.

## Main approach

At a high level, the controller being tested looks like this:

```text
PV curve -> 12-point sparse scan -> TCNformer MPP seed -> local P&O refinement -> final MPPT point
```

The deep learning part is not meant to replace MPPT completely. It is used as a guide. The model gives the controller a better starting point, and P&O still performs the final local adjustment.

That split is important because it keeps the method grounded in actual curve measurements while still using the model to handle the hard part: choosing the right region under partial shading.

## Datasets expected

The notebook expects PV datasets to be provided at runtime. In Colab, it will ask for an upload if `DATASET_PATH` is not set manually.

The main training/evaluation dataset can be a `.mat` or `.npz` file containing simulated and experimental PV curves. The notebook has explicit support for MATLAB keys such as:

- `full_curvesOk_simulated`
- `full_curvesSh_simulated`
- `full_curvesSh_experimental`
- `full_curvesOk_experimental`

For unseen inference, the notebook expects:

- `PV_Data_Reduced.mat`
- key: `PV_Data_Reduced`
- format: direct `[V, P]` data

Because `PV_Data_Reduced.mat` is treated as P-V data, the notebook converts it to pseudo I-V internally before using the existing MPPT pipeline.

## Recommended environment

This notebook was written with Google Colab in mind.

Recommended setup:

- Python 3
- GPU runtime if available
- A100 GPU preferred for faster training
- NumPy
- pandas
- matplotlib
- SciPy
- scikit-learn
- PyTorch

In Colab, use:

```text
Runtime -> Change runtime type -> GPU
```

The notebook can run on CPU, but training will be slower.

## How to run

1. Open `TCN_with_P&O_For_Partial_Shading_MPPT.ipynb` in Google Colab or Jupyter.
2. Select a GPU runtime if one is available.
3. Run the cells from top to bottom.
4. Upload the main PV dataset when prompted, or set `DATASET_PATH` manually in the configuration cell.
5. Review the preprocessing summary to make sure valid simulated and shaded experimental curves were found.
6. Let the model train.
7. Review the evaluation tables and plots.
8. Upload `PV_Data_Reduced.mat` when running the unseen inference section.

## Outputs to look at

The most useful outputs are:

- cleaning and preprocessing statistics,
- train and validation loss curves,
- zone accuracy,
- predicted MPP voltage error,
- final refined power ratio,
- final power regret,
- dense peak count,
- visual examples of shaded P-V curves.

When I review results, I mainly look for this behavior:

- the neural seed lands near the correct high-power region,
- P&O improves or preserves the seed result,
- the refined point is close to the true dense-curve MPP,
- the method does better than the coarse 12-point baseline on difficult shaded curves.

## Current project status

This is still a research and understanding notebook. It is not packaged as a production MPPT controller yet.

Some parts are intentionally verbose because I wanted the notebook to explain what each step is doing. The code also keeps several checks and summaries in place so it is easier to see when a dataset is being interpreted incorrectly.

Things I would improve next:

- separate the notebook logic into reusable Python modules,
- add a small public sample dataset for quick testing,
- save standard result tables to disk,
- add automated tests for curve cleaning and P-V to I-V conversion,
- compare against more classical MPPT baselines,
- document the exact dataset source and expected MATLAB structure more formally.

## Notes from the engineering side

The part I paid the most attention to is data interpretation. A PV curve can be stored as `[V, I]`, `[I, V]`, `[V, P]`, transposed arrays, nested MATLAB objects, or grouped keys. If that format is misunderstood, the model can look like it is working while actually learning from the wrong power calculation.

That is why the notebook spends time cleaning curves, checking endpoints, handling direct P-V input carefully, and printing summaries before training. For this kind of MPPT experiment, the preprocessing is just as important as the neural network.

## License

No license file is currently included. Add one before reusing or distributing this work outside the project context.
