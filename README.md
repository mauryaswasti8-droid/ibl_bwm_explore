# ibl_bwm_explore
Exploratory pilot on the IBL Brain Wide Map (DANDI:000409): can the block prior be decoded from pre-stimulus spikes beyond drift and choice history? One session, drift-aware pseudosession null.

# Does neural activity carry block-prior information beyond choice history? A pilot on the IBL Brain Wide Map

**Status:** exploratory pilot on a single session. Not a finding. Written to document the method, the controls, and every variant I tried.

## Summary

In the International Brain Laboratory (IBL) visual decision task, the probability that the stimulus appears on the left switches between 0.2 and 0.8 in uncued blocks. Published work (Findling et al., 2025) shows that mice use this prior and that it can be decoded from neural activity in many brain regions.

A decoder trained on slowly drifting neural recordings can look "significant" just because both the recordings and the block labels change slowly over a session. A second problem is that a mouse's recent choices already track the block, so neural activity could be reflecting choice history instead of a prior.

This repository asks, on one session: **can the block be decoded from pre-stimulus spike counts, and does it still decode after accounting for drift and for behavioral history?**

**Short answer for this session:** the block decodes above a drift-aware null, but trial history alone decodes it far better, adding neurons to history gives no gain, and after regressing out history the residual neural signal is at chance. In this session the neural signal is consistent with tracking recent choices. With one session, one probe and a strong history/block correlation, this is a pilot and says nothing about other regions.

## Data

- **Dataset:** IBL Brain Wide Map, DANDI:000409, version 0.260309.1324 (CC-BY-4.0), in NWB format
- **Session:** `sub-CSH-ZAD-001_ses-3e7ae7c0-fe8b-487c-9354-036236fa1010_desc-processed_behavior+ecephys.nwb`, streamed from DANDI (not downloaded in full)
- **Recording:** one Neuropixels probe (Probe00); units recorded mostly in midbrain structures (e.g., SNr, VTA, Fields of Forel)
- **Trials:** 854 total. I dropped the initial 90 unbiased trials (P(left) = 0.5), the last block (3 trials), and trials without a stimulus onset time, leaving **761 trials**
- **Units:** 50 units with `ibl_quality_score >= 0.99`

## Methods

**Neural features.** For each trial, spike counts per unit in a window from 600 ms to 100 ms before stimulus onset (the inter-trial interval, when the mouse is mostly still).

**Target.** Block identity: P(left) = 0.8 vs 0.2 (binary).

**Decoder.** L1-regularized logistic regression (C = 0.1) on standardized counts, with interleaved 5-fold cross-validation (trial *i* goes to fold *i* mod 5). Score: ROC AUC on pooled out-of-fold predictions.

**Drift-aware null (pseudosessions).** Following the logic of Findling et al. (2025) and Harris (2020), I generated fake block sequences from the task's generative process (first 90 trials at 0.5, then alternating 0.2/0.8 blocks with lengths drawn from an exponential with scale 60, truncated to 20-100 trials), ran the identical decoding on the real neural data, and compared the real AUC to this null distribution. With 200 pseudosessions, the smallest reportable p-value is about 0.005.

**History baseline.** Features from previous-trial behavior: previous choice, previous reward, previous stimulus side, and an exponentially weighted average of past choices (decay constant 5.45 trials, the median reported for mice in Findling et al.).

**Residual analysis.** Neural counts were linearly regressed on the history features, and block was decoded from the residuals (same pipeline and null).

## Results (one session, 761 trials, 50 units)

| Features | Real AUC | Null median | Null 95th percentile |
|---|---|---|---|
| History only | 0.948 | 0.574 | 0.727 |
| Neural only | 0.754 | 0.651 | 0.714 |
| Neural + history | 0.952 | 0.673 | 0.764 |
| Neural residual (history regressed out) | 0.479 | 0.637 | 0.703 |

Note that the null median for neural features is well above 0.5. Fake labels alone score about 0.65 because of slow drift, which is why a raw AUC alone would overstate the result.

**Behavior check.** At zero contrast, the mouse chose "right" about 62% of the time in blocks where the right side was more likely (P(left) = 0.2) and about 12% in the opposite blocks, and the curves converge at the highest contrasts. This matches the behavioral effect reported by the IBL.

## Analysis log (everything I tried)

1. **Block-wise (GroupKFold) cross-validation** with plain accuracy. Result: AUC ~0.30, below chance. Cause: alternating blocks plus slow drift make a held-out block resemble neighboring training blocks of the opposite class. This was an artifact of the evaluation design, so I abandoned it in favor of interleaved folds plus a pseudosession null.
2. **Interleaved CV + pseudosession null** (neural only): real AUC 0.754, above all 200 pseudosessions in the first run (null median 0.653, 95th percentile 0.722). The table above is from a later run with a different random seed.
3. **History-only and neural + history models** (table above).
4. **Residual analysis** (table above).

No other time windows, unit filters, or targets were tried.

## Limitations

- **One session, one probe, 50 units.** Nothing here generalizes to other regions or mice. The paper estimates about 10 sessions per region are needed to detect its effects.
- **History and block are strongly correlated** (choices follow stimuli, which follow the block), so history alone nearly reproduces the target. This leaves little headroom for neurons to add anything, and a null on the residual has low power.
- **Linear residualization only.** Nonlinear dependence on history is not removed, and the history regression was fit on all trials before cross-validation (it should be done inside each fold in a final analysis).
- **True block label as the target**, not the Bayes-optimal subjective prior used in the paper, which decodes better.
- **200 pseudosessions**, so p-values are coarse; a final analysis should use 1000.
- **Midbrain only.** The paper reports prior-related signals in other regions, and my results do not contradict it.

## Reproducing

Run `ibl_bwm_explore.ipynb` top to bottom in Google Colab (Runtime, then Run all). It installs `dandi`, `pynwb`, `h5py` and `remfile`, then streams the session file from DANDI. Streaming the spike times takes a few minutes.

`pilot_session.npz` (neural counts, labels, trial indices) and `pilot_trials.csv` (trials table) are saved copies so the decoding can be re-run without streaming.

## Next steps

- Extend to many sessions and decode within each, combining per-region results with Fisher's method as in Findling et al.
- Use 1000 pseudosessions and fit history regressions within each cross-validation fold.
- Use the Bayes-optimal prior as the target.

## References

- Findling, C. et al. (International Brain Laboratory). Brain-wide representations of prior information in mouse decision-making. *Nature* 645 (2025). https://doi.org/10.1038/s41586-025-09226-1
- International Brain Laboratory et al. A brain-wide map of neural activity during complex behaviour. *Nature* (2025). https://doi.org/10.1038/s41586-025-09235-0
- Harris, K. D. Nonsense correlations in neuroscience. bioRxiv (2020). https://doi.org/10.1101/2020.11.29.402719
- DANDI Archive, dandiset 000409: https://dandiarchive.org/dandiset/000409

## Author

Swasti Maurya. Exploratory work toward an application to the NeuroData AI Summer School 2027.
