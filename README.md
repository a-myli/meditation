# Brain Activity Patterns Across Meditation Experience Levels

Neurotech@USC · Offline preprocessing and Welch features for EEG.

## Install

Open a terminal in the project folder. The pinned dependencies were exercised on macOS Apple Silicon with Python 3.14.

```sh
cd "$HOME/Desktop/Meditation"
python3.14 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pytest -q
```

Open `Meditation_processing.ipynb` in your notebook editor with the `.venv` kernel and run all cells from the project folder. The notebook and terminal use the same pipeline. The default terminal path runs headlessly. For optional desktop marking, add `--interactive` and use an interactive Matplotlib backend. In a notebook select a GUI backend before setting `interactive=True`. In the browser, click a channel label to toggle its bad flag and press A to mark `BAD_...` intervals. Save these edits by letting the run finish. The desktop browser interaction needs local validation.

## Process a recording

1. Put the EDF in `data/` or edit `paths.data_dir` in `config.toml`. Use pseudonymous filenames and IDs and keep the linkage separately. Add one row to `examples/participants.csv` with the actual filename and pseudonymous IDs. Its columns are `participant_id,session_id,file,experience_level,eyes_condition,time_of_day`. Leave unknown fields blank. Use a distinct session ID per input recording and keep each person's participant ID consistent across sessions.
2. Review headset labels, channel roles, EEG units and actual per-channel rates against the acquisition setup. Enter explicit renames and channel types. Set `expected_eeg` to the verified standardized layout to catch missing or extra EEG channels. Channels with names such as Trigger, Event, or Status need a verified type. Mark known auxiliary channels as `misc` or their true type so the EEG filter and features omit them. Assign `stim` only to verified integer digital triggers. MNE reads these as EDF digital codes, so compare the exported codes against the acquisition log before trusting them. If two channels both carry codes, both are inspected separately. Select one explicitly for verified stage mapping.
3. Enter the documented acquisition reference if known. Keep `reference = "preserve"` unless the study protocol calls for average reference with a consistent channel set. Add a named montage only if its coordinates match the headset. A suffix such as `-REF` is part of the vendor label and does not identify the hardware reference.
4. Run the signal-only path on the first recording. Substitute its actual filename for `recording.edf`.

```sh
mkdir -p data
python -m meditation_eeg.pipeline --config config.toml --file recording.edf
```

With no `--file`, the runner processes sorted supported files in the configured folder. Paths resolve relative to the config file. Each successful run publishes to `processed/<participant_id>/<session_id>/`. Use a fresh output directory for reruns. A failure leaves previously completed sessions available.

If you have the original 19-column MATLAB files from the supplied notebook, enable `legacy_mat_enabled` after verifying `variable`, EEG column order, EEG units, timestamp conversion and sampling rate. The importer checks sample spacing, converts EEG to volts and retains both condition columns as separate stimulus channels. It records the native timestamp offset in the source description. The MATLAB importer cannot recover an acquisition date absent from the matrix. Use signal-only mode for motor imagery in either file format.

## Configure authentic meditation timing

Set `mode = "meditation"`. Choose one verified timing source and set `stages.provenance` to a protocol or timing-log identifier and version. Verify onset offsets against the recording's time zero. Provenance documents your verification. The software can check consistency and cannot determine whether an experimental label is true.

- **CSV** — Copy the header-only `examples/stage_annotations.csv` and fill rows from the actual session log. Use the same IDs and filename as the manifest. Enter `stage`, `onset_seconds` and `duration_seconds` relative to the first recorded sample. Each row covers one continuous interval.
- **EDF annotations** — Set `source = "annotations"` and map the exact acquisition descriptions in `stages.annotation_map`. Original annotations are retained. Mapped annotations must have real durations.
- **Trigger pairs** — Set `source = "triggers"`, choose the verified `stim_channel` and configure distinct positive integer start/end codes for each stage. The decoder records every transition into a nonzero value, including an initial nonzero sample. Confirm that sustained values and direct code transitions carry those semantics in your export. Absent pairs are allowed. A marker present without its partner, reversed pairs, overlaps, invalid timing, or no matched intervals fails validation.

Use `pre_meditation`, `meditation` and `post_meditation` only for observed stages. Omit stages that were skipped. Original measurement dates and annotations are retained. Stage windows use seconds from the first sample. Timing and metadata in FIF and QC reports may be sensitive even when IDs are pseudonymous.

## Cleaning and quality checks

- The default zero-phase FIR filter passes 0.5–41 Hz on EEG channels. The filter separates spans at annotations beginning with `BAD` or `EDGE`. Review filter-length warnings and edges around bad or short spans. EEG bypasses any extra line-noise notch by default. The original files remain the full-rate archive.
- Stimulus channels bypass the EEG filter. For rates above 256 Hz, MNE downsamples the cleaned raw and maps native events jointly. Inspect `triggers.csv`, which retains native and mapped sample numbers and times. Timing is quantized to the new grid and closely spaced pulses can merge in the resampled stimulus trace. Stage decisions always use the original event times. The pipeline keeps native rate if the first-sample time would shift, and never upsamples. [MNE Raw API](https://mne.tools/stable/generated/mne.io.Raw.html)
- The EEG channel report gives peak-to-peak, standard deviation and the 95th minus 5th percentile range in µV before filtering. Channels with full-recording peak-to-peak below 0.5 µV are excluded by default. Robust range over 150 µV flags review. Inspect intermittent flatness, clipping, line interference, electrode changes and visually noisy segments before setting study thresholds.
- Optional interpolation requires a valid montage, unique finite electrode positions and at least four good EEG channels with more good than bad channels. Interpolated features carry an explicit status. Channels left bad are excluded from epoching. Sparse montages and session-specific bad/interpolated channels can change the effective average reference.
- ICA defaults off. With a verified EOG and adequate fit data, inspection mode fits FastICA on a per-recording copy high-passed at 1 Hz. It reports rank, EOG correlations and candidates, and saves source traces and the fitted solution for review. No components are subtracted. A missing EOG, inadequate rank, inadequate samples, or iteration limit yields a skip reason. Visually confirm putative ocular components before designing a reviewed removal step.
- Complete five-second candidates start on the sample grid inside each verified interval, or tile the recording in signal-only mode. Inward rounding and discarded partial tails protect stage boundaries. `BAD` overlaps are logged and rejected. Remaining candidates use mean removal and EEG peak-to-peak rejection above 150 µV or below 0.5 µV, configurable. Counts use all complete candidates as the denominator. A dropped epoch can have multiple reasons. [MNE Epochs API](https://mne.tools/stable/generated/mne.Epochs.html)

## Outputs and feature definitions

| Output | Contents |
| --- | --- |
| `cleaned_raw.fif`, `cleaned-epo.fif` | Cleaned EEG and auxiliary channels in Raw; retained EEG epochs in Epochs. The epoch FIF is omitted when there are zero retained epochs. |
| `qc.json`, `triggers.csv` | Acquisition and cleaned channel inventory, source EDF units and rates, configured reference, original and cleaned annotations, timing provenance, settings, artifact review and epoch totals. |
| `channel_qc.csv`, `epoch_qc.csv` | Pre-cleaning channel statistics and flags; one row per candidate with stage, time and drop reasons. |
| `features.csv` | One row per retained epoch × included EEG channel × band. Stage and stage provenance are blank in signal-only mode. Rejected candidates appear in epoch QC. |
| `session_summary.csv` | Mean features and contributing epoch counts per participant, session, stage, channel, band and QC status. |
| PNG plots | Sample traces, channel spectra before/after cleaning, power trends, and stage comparisons when available. Optional ICA source traces show the first 15 seconds of fitted component time courses. |

Welch PSD uses two-second Hann segments with 50% overlap, mean averaging, DC removal and typically 0.5 Hz bin spacing. The PSD has units V²/Hz. Absolute power in V² is the sum of PSD bins in the band multiplied by bin width. Theta uses [4, 8) Hz, alpha [8, 13) Hz and beta [13, 30) Hz. Relative power is the band power divided by the sum of these three bands. Theta/beta and alpha/beta use absolute powers. Relative powers and ratios are dimensionless. A zero denominator leaves the value blank. Both ratios are repeated on the three band rows for easy filtering. QC trace/PSD figures use µV and µV²/Hz and may include BAD spans for visual inspection.

## Exploratory research analysis and baseline

Run these only after real labeled sessions exist. Substitute a confirmed common channel for `F3`.

```sh
python -m meditation_eeg.analysis --processed processed --participants examples/participants.csv --output exploration --channel F3
python -m meditation_eeg.modeling --processed processed --participants examples/participants.csv --channels F3 --folds 3
```

Analysis averages retained non-interpolated epochs within each session and then averages sessions within participant, stage, channel and band. It reports the number of contributing participants for each experience and stage cell. The scatterplot shows participant-stage values. It provides descriptive summaries without significance tests. Review individual variability, movement and muscle artifact, eyes-open/closed condition, time of day, headset or session changes, acquisition and analysis reference, and stage order. If stage and eyes condition always change together, their effects cannot be separated from this design.

The baseline is logistic regression on preselected common-channel relative band powers, using one row per participant and stage. Grouped folds hold out entire participants. Scaling and fitting occur inside each training fold. The baseline reports class balance, exclusions for missing features, balanced accuracy and a confusion matrix if the grouped folds cover all stages. Insufficient coverage or missing verified stages skips fitting. Set channel selection and processing rules before looking at CV results. The baseline predicts protocol stage from participant-stage averages, with potential confounding by stage order and acquisition conditions. [Scikit-learn grouped cross-validation API](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.StratifiedGroupKFold.html)

## Audit and validation

See [AUDIT.md](AUDIT.md) for the original code and saved-output findings, the changes made, and the boundary between synthetic tests and acquisition validation.
