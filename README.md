# Electrode drift in extracellular recordings

A pipeline that quantifies and corrects electrode drift in spike-sorting data from a single input, the 2D spike locations, so it can sit in front of any spike sorter.

Undergraduate dissertation, BSc Computer Science, School of Informatics, University of Edinburgh, 2021. Supervised by Dr Matthias Hennig. Full report: [ug4_report-RaresTudorache.pdf](ug4_report-RaresTudorache.pdf).

## Problem

High-density silicon probes record thousands of neurons at once. Spike sorting assigns each detected spike to the neuron that produced it. During long recordings the probe moves relative to the tissue, so one neuron's spikes appear at shifting positions and their waveforms change. Most sorters either ignore this electrode drift or rely on a manual curation step at the end.

The recording used for development has a drift motion imposed on purpose, which shows up as a zig-zag in the spike positions over time. The goal is to straighten that pattern and remove the manual step.

## Pipeline

```
2D spike locations -> clustering -> cluster matching -> drift profile -> data cleaning -> correction -> reclustering
```

1. **Clustering.** The recording is split into 20 s time frames. Spikes in each frame are clustered with MeanShift (bandwidth 6.5, bin seeding), which needs no cluster count up front. The improved variant discards clusters with fewer than 30 spikes, about 20 percent of the groups per frame, because they are noise rather than a neuron.
2. **Matching.** Consecutive frames are paired with a greedy nearest-pair search over the distance matrix of cluster centres, stopping once the smallest remaining distance passes a threshold. It behaves like a thresholded Hungarian match at a fraction of the cost.
3. **Drift profile.** The average displacement of matched pairs per frame, split into vertical and horizontal components, is the drift profile of the recording.
4. **Data cleaning.** Pairs whose displacement falls outside the frame average plus or minus half a standard deviation are discarded as mismatches. The displacement spread drops from a 4 to 22 µm range to 0 to 4 µm.
5. **Cumulative displacement.** The running sum of average displacements quantifies the drift and is the metric every correction is judged on. Quantifying the drift this way was not part of earlier approaches.
6. **Correction.** Two methods: shift every spike in a frame by that frame's average displacement, or interpolate a per-spike correction inside each frame. Interpolation resolves the imposed zig-zag almost completely. A section-wise variant runs the pipeline on probe sections independently to handle drift that differs along the probe.
7. **Reclustering.** MeanShift runs again on the corrected locations to produce the sorted output.

## Results

Absolute average vertical displacement along the probe's long axis, in µm, before and after correction (Table 4.1 of the report):

| Dataset | Before | Interpolation, hard threshold | Interpolation, soft threshold | Sections |
|---|---|---|---|---|
| Imposed drift | 19.38 | 0.50 | 4.67 | 2.40 |
| Allen Mouse | 2.85 | 4.40 | 0.61 | 1.45 |
| Svoboda | 3.28 | 0.37 | 1.67 | 0.35 |

On the imposed-drift recording the pipeline removes about 80 percent of the drift measured as cumulative displacement and replaces the final manual curation step. The two datasets without imposed drift show the trade-off in the cleaning threshold: a strict threshold suits recordings with small, smooth movement, a softer one is needed to keep the sudden movements in the Allen Mouse recording.

Limitations: the matching and cleaning thresholds need tuning per dataset, and because the correction accumulates displacements, a single mismatch that survives cleaning propagates to every later frame.

## Notebooks

| Notebook | Contents |
|---|---|
| `MeanShift_Baseline_Clustering.ipynb` | Baseline clustering, matching, drift profile, cleaning, correction by average displacement |
| `MeanShift_Improved_Clustering.ipynb` | Improved clustering with small clusters dropped, correction by average displacement and by interpolation, reclustering |
| `Probe_sections.ipynb` | The pipeline refactored into functions (`clustering`, `matching`, `avg_disp`, `clean_data`, `cum_disp`, `correction`, `align_spikes`, `cluster_corrected`) and applied per probe section |

## Data

The notebooks read a HerdingSpikes output file (`HS2_sorted.hdf5`) with h5py. Spikes were detected and localised with the method of Muthmann et al. (2015): for each event the channel, the 2D location estimated from the amplitudes on neighbouring sites, and the timestamp. The recordings themselves (the imposed-drift recording, Allen Mouse and Svoboda) are not included. Point the `h5py.File(...)` call at the top of each notebook at your own HerdingSpikes sorted file.

## Running

```bash
pip install numpy pandas scipy scikit-learn h5py matplotlib jupyter
jupyter notebook
```

Python 3, no GPU needed. Each notebook runs top to bottom once the data path is set.
