# MyTruthAnalyzer

A CMSSW `EDAnalyzer` that validates the CMS **Truth Graph** and studies the
matching between truth-level particles and reconstructed **TICL Tracksters**,
using detector-hit overlap and association scores.

> **Note:** This is a CMSSW plugin, not a standalone program. It depends on
> CMSSW data formats, the CMS Truth Graph producers, TICL reconstruction, and
> the CMSSW build/run system (`scram`, `cmsRun`). It cannot be compiled or run
> outside a CMSSW release/checkout — see [Requirements](#requirements) and
> [Usage](#usage) below.

## Contents

- [`MyTruthAnalyzer.cc`](#) — the analyzer plugin source
- a configuration (# MyTruthAnalyzer

A CMSSW `EDAnalyzer` that validates the CMS **Truth Graph** and studies the
matching between truth-level particles and reconstructed **TICL Tracksters**,
using detector-hit overlap and association scores.

> **Note:** This is a CMSSW plugin, not a standalone program. It depends on
> CMSSW data formats, the CMS Truth Graph producers, TICL reconstruction, and
> the CMSSW build/run system (`scram`, `cmsRun`). It cannot be compiled or run
> outside a CMSSW release/checkout — see [Requirements](#requirements) and
> [Usage](#usage) below.

## Contents

- [`MyTruthAnalyzer.cc`](#) — the analyzer plugin source
- a configuration (`myTruthAnalyzer_cfg.py`) showing how the module is wired into a
  CMSSW job (see [Configuration](#configuration))

## What it does

The analyzer reads the CMS Truth Graph (generator + simulation history merged
into a single navigable structure, see [CMS Truth Graph
docs](https://cms-truth.docs.cern.ch/)) together with reconstructed
Tracksters and detector RecHits, and fills a set of ROOT histograms used to:

1. **Validate the Truth Graph** — particle multiplicities, mother/ancestor
   relationships, kinematics, and decay classification, cross-checked against
   known Standard Model expectations (mainly using a dedicated `TenTau`
   sample).
2. **Study detector response** — link truth particles to the calorimeter
   SimHits/RecHits they produced, and compare truth energy to reconstructed
   energy (overall, and split by ECAL/HCAL/HGCAL component).
3. **Evaluate truth-to-reconstruction matching** — using
   `ticl::AssociationMap` products, match truth pions (from τ decays) to TICL
   Tracksters in both directions (Truth → RECO and RECO → Truth), and derive
   matching efficiencies as a function of `pT`, using an association-score
   threshold to define a "good match".

The module supports two independent run modes, controlled by configuration
flags:

- `doTenTau` — validation and matching studies on a dedicated `TenTau`
  sample (ten τ leptons per event).
- `doDYtoLL` — validation of a Drell–Yan (Z → ℓℓ) sample, used as an
  additional cross-check of decay classification and matching-score
  distributions.

Both can in principle be enabled in the same job, but they are intended to be
run one at a time against the matching sample.

## Requirements

- A working **CMSSW** release/checkout (the plugin was developed against a
  Phase-2 CMSSW release with Truth Graph support enabled).
- The following data products must be available in the input files:
  - `truth::Graph` (Truth Graph)
  - `truth::LogicalGraphHitIndex` (SimHit index per logical truth particle)
  - `std::vector<ticl::Trackster>` (TICL Tracksters)
  - `ticl::AssociationMap<ticl::mapWithSharedEnergyAndScore>` — one Truth →
    RECO map and up to three RECO → Truth maps (nominal / tight / loose)
  - `EcalRecHitCollection`, `HBHERecHitCollection`, and one or more
    `HGCRecHitCollection`s (EE, HEF, HEB)
- These products are produced upstream by the Truth Graph producers and the
  TICL/truth-association sequences (see the example configuration).

## Configuration

The analyzer is configured through the following `EDAnalyzer` parameters:

| Parameter | Type | Description |
|---|---|---|
| `src` | `InputTag` | The `truth::Graph` product |
| `hitIndex` | `InputTag` | The `truth::LogicalGraphHitIndex` product |
| `tracksters` | `InputTag` | TICL Trackster (or Candidate) collection |
| `truthToTrackster` | `InputTag` | Truth → RECO association map |
| `tracksterToTruthAdaptiveNominal` | `InputTag` | RECO → Truth map, nominal working point |
| `tracksterToTruthAdaptiveTight` | `InputTag` | RECO → Truth map, tight working point |
| `tracksterToTruthAdaptiveLoose` | `InputTag` | RECO → Truth map, loose working point |
| `hgcalRecHits` | `VInputTag` | HGCAL RecHit collections (EE, HEF, HEB) |
| `ecalRecHits` | `InputTag` | ECAL barrel RecHits |
| `hcalRecHits` | `InputTag` | HCAL barrel/endcap (HBHE) RecHits |
| `doTenTau` | `bool` | Enable TenTau validation + matching studies |
| `doDYtoLL` | `bool` | Enable DYtoLL validation studies |

See `myTruthAnalyzer_cfg.py` for a complete job configuration, including
the Truth Graph prevalidation sequence
(`Validation.Configuration.truthPrevalidation_cff`) and the Phase-2 geometry
(`GeometryExtendedRun4D122Reco_cff`) that must be loaded before the analyzer
runs.

⚠️ The `truthToTrackster` `InputTag` instance label is direction-sensitive —
double check you are pointing at the Truth→RECO product
(`ticlCandidateTruthToReco`) and not one of the RECO→Truth products
(`ticlCandidateRecoToTruthFixed`/`...Adaptive*`), which are configured
separately.

## Usage

This plugin must be built and run inside a CMSSW area:

```bash
# from within $CMSSW_BASE/src, with this plugin's package checked out
scram b -j8

cmsRun path/to/`myTruthAnalyzer_cfg.py
```

The job produces a single ROOT file (`TFileService`, default name set in the
`myTruthAnalyzer_cfg.py`, e.g. `myTenTau.root`) containing all booked histograms.

## Output histograms (selected)

Grouped by analysis stage (see source for the complete list):

- **Multiplicities & kinematics**: `GenTauNum`, `SimTauNum`, `TruthTauNum`,
  `GenPhtNum`, `GenEleNum`, `*Pt`, `*Eta`, `TauDecayVtxRadius`
- **Ancestry**: `GenTauMother`, `SimTauMother`, `GenPhtMother`, `SimPhtMother`,
  `GenEleMother`, `SimEleMother`
- **Detector response**: `TauSimHitE`, `TauRecHitE`, `TauEResponse`,
  `TauEResponseVsEta`, `TauEResponseVsPt`
- **EM/hadronic energy sharing** (τ → π± π0 ντ): `1ProngBFECALVsEta`,
  `1ProngBFHCALVsEta`, `1ProngECALoverHCALVsEta`, `1Pi0BFECALVsEta`, and the
  corresponding `VsPt` variants
- **Truth → RECO matching**: `AssociationScoreOfPionBestMatch`,
  `AssociationScoreOfPionBestMatchVsEta/Pt`, `TruthPionPtAll`,
  `TruthPionPtMatched`, `TruthToRecoEfficiency`
- **RECO → Truth matching**: `TrackesterToPionBestMatchNominal/Tight/Loose`,
  `TracksterPtAll`, `TracksterPtMatched`, `RecoToTruthEfficiency`
- **DYtoLL cross-checks**: `GenZMass`, `GenZPt`, `GenZEta`,
  `GenEleFromZPt/Eta`, `GenMuFromZPt/Eta`, `GenTauFromZPt/Eta`

Tau decay-mode branching fractions and matching-score bin summaries are also
printed to the log at `endJob()`.

## Known limitations / caveats

- Matching efficiency studies restrict truth pions and Tracksters to
  `1.5 < |η| < 3.0` (HGCAL acceptance); results outside this range are not
  covered.
- The "good match" threshold (association score `< 0.6`) is currently
  hardcoded rather than configurable.

## References

- CMS Truth Graph documentation: <https://cms-truth.docs.cern.ch/data-model/>
- Identification of hadronic tau lepton decays using a deep neural network,
  *JINST* 17 (2022) P07023cfg.py`) showing how the module is wired into a
  CMSSW job (see [Configuration](#configuration))

## What it does

The analyzer reads the CMS Truth Graph (generator + simulation history merged
into a single navigable structure, see [CMS Truth Graph
docs](https://cms-truth.docs.cern.ch/)) together with reconstructed
Tracksters and detector RecHits, and fills a set of ROOT histograms used to:

1. **Validate the Truth Graph** — particle multiplicities, mother/ancestor
   relationships, kinematics, and decay classification, cross-checked against
   known Standard Model expectations (mainly using a dedicated `TenTau`
   sample).
2. **Study detector response** — link truth particles to the calorimeter
   SimHits/RecHits they produced, and compare truth energy to reconstructed
   energy (overall, and split by ECAL/HCAL/HGCAL component).
3. **Evaluate truth-to-reconstruction matching** — using
   `ticl::AssociationMap` products, match truth pions (from τ decays) to TICL
   Tracksters in both directions (Truth → RECO and RECO → Truth), and derive
   matching efficiencies as a function of `pT`, using an association-score
   threshold to define a "good match".

The module supports two independent run modes, controlled by configuration
flags:

- `doTenTau` — validation and matching studies on a dedicated `TenTau`
  sample (ten τ leptons per event).
- `doDYtoLL` — validation of a Drell–Yan (Z → ℓℓ) sample, used as an
  additional cross-check of decay classification and matching-score
  distributions.

Both can in principle be enabled in the same job, but they are intended to be
run one at a time against the matching sample.

## Requirements

- A working **CMSSW** release/checkout (the plugin was developed against a
  Phase-2 CMSSW release with Truth Graph support enabled).
- The following data products must be available in the input files:
  - `truth::Graph` (Truth Graph)
  - `truth::LogicalGraphHitIndex` (SimHit index per logical truth particle)
  - `std::vector<ticl::Trackster>` (TICL Tracksters)
  - `ticl::AssociationMap<ticl::mapWithSharedEnergyAndScore>` — one Truth →
    RECO map and up to three RECO → Truth maps (nominal / tight / loose)
  - `EcalRecHitCollection`, `HBHERecHitCollection`, and one or more
    `HGCRecHitCollection`s (EE, HEF, HEB)
- These products are produced upstream by the Truth Graph producers and the
  TICL/truth-association sequences (see the example configuration).

## Configuration

The analyzer is configured through the following `EDAnalyzer` parameters:

| Parameter | Type | Description |
|---|---|---|
| `src` | `InputTag` | The `truth::Graph` product |
| `hitIndex` | `InputTag` | The `truth::LogicalGraphHitIndex` product |
| `tracksters` | `InputTag` | TICL Trackster (or Candidate) collection |
| `truthToTrackster` | `InputTag` | Truth → RECO association map |
| `tracksterToTruthAdaptiveNominal` | `InputTag` | RECO → Truth map, nominal working point |
| `tracksterToTruthAdaptiveTight` | `InputTag` | RECO → Truth map, tight working point |
| `tracksterToTruthAdaptiveLoose` | `InputTag` | RECO → Truth map, loose working point |
| `hgcalRecHits` | `VInputTag` | HGCAL RecHit collections (EE, HEF, HEB) |
| `ecalRecHits` | `InputTag` | ECAL barrel RecHits |
| `hcalRecHits` | `InputTag` | HCAL barrel/endcap (HBHE) RecHits |
| `doTenTau` | `bool` | Enable TenTau validation + matching studies |
| `doDYtoLL` | `bool` | Enable DYtoLL validation studies |

See the  `# MyTruthAnalyzer

A CMSSW `EDAnalyzer` that validates the CMS **Truth Graph** and studies the
matching between truth-level particles and reconstructed **TICL Tracksters**,
using detector-hit overlap and association scores.

> **Note:** This is a CMSSW plugin, not a standalone program. It depends on
> CMSSW data formats, the CMS Truth Graph producers, TICL reconstruction, and
> the CMSSW build/run system (`scram`, `cmsRun`). It cannot be compiled or run
> outside a CMSSW release/checkout — see [Requirements](#requirements) and
> [Usage](#usage) below.

## Contents

- [`MyTruthAnalyzer.cc`](#) — the analyzer plugin source
- an example configuration (`cfg.py`) showing how the module is wired into a
  CMSSW job (see [Configuration](#configuration))

## What it does

The analyzer reads the CMS Truth Graph (generator + simulation history merged
into a single navigable structure, see [CMS Truth Graph
docs](https://cms-truth.docs.cern.ch/)) together with reconstructed
Tracksters and detector RecHits, and fills a set of ROOT histograms used to:

1. **Validate the Truth Graph** — particle multiplicities, mother/ancestor
   relationships, kinematics, and decay classification, cross-checked against
   known Standard Model expectations (mainly using a dedicated `TenTau`
   sample).
2. **Study detector response** — link truth particles to the calorimeter
   SimHits/RecHits they produced, and compare truth energy to reconstructed
   energy (overall, and split by ECAL/HCAL/HGCAL component).
3. **Evaluate truth-to-reconstruction matching** — using
   `ticl::AssociationMap` products, match truth pions (from τ decays) to TICL
   Tracksters in both directions (Truth → RECO and RECO → Truth), and derive
   matching efficiencies as a function of `pT`, using an association-score
   threshold to define a "good match".

The module supports two independent run modes, controlled by configuration
flags:

- `doTenTau` — validation and matching studies on a dedicated `TenTau`
  sample (ten τ leptons per event).
- `doDYtoLL` — validation of a Drell–Yan (Z → ℓℓ) sample, used as an
  additional cross-check of decay classification and matching-score
  distributions.

Both can in principle be enabled in the same job, but they are intended to be
run one at a time against the matching sample.

## Requirements

- A working **CMSSW** release/checkout (the plugin was developed against a
  Phase-2 CMSSW release with Truth Graph support enabled).
- The following data products must be available in the input files:
  - `truth::Graph` (Truth Graph)
  - `truth::LogicalGraphHitIndex` (SimHit index per logical truth particle)
  - `std::vector<ticl::Trackster>` (TICL Tracksters)
  - `ticl::AssociationMap<ticl::mapWithSharedEnergyAndScore>` — one Truth →
    RECO map and up to three RECO → Truth maps (nominal / tight / loose)
  - `EcalRecHitCollection`, `HBHERecHitCollection`, and one or more
    `HGCRecHitCollection`s (EE, HEF, HEB)
- These products are produced upstream by the Truth Graph producers and the
  TICL/truth-association sequences (see the example configuration).

## Configuration

The analyzer is configured through the following `EDAnalyzer` parameters:

| Parameter | Type | Description |
|---|---|---|
| `src` | `InputTag` | The `truth::Graph` product |
| `hitIndex` | `InputTag` | The `truth::LogicalGraphHitIndex` product |
| `tracksters` | `InputTag` | TICL Trackster (or Candidate) collection |
| `truthToTrackster` | `InputTag` | Truth → RECO association map |
| `tracksterToTruthAdaptiveNominal` | `InputTag` | RECO → Truth map, nominal working point |
| `tracksterToTruthAdaptiveTight` | `InputTag` | RECO → Truth map, tight working point |
| `tracksterToTruthAdaptiveLoose` | `InputTag` | RECO → Truth map, loose working point |
| `hgcalRecHits` | `VInputTag` | HGCAL RecHit collections (EE, HEF, HEB) |
| `ecalRecHits` | `InputTag` | ECAL barrel RecHits |
| `hcalRecHits` | `InputTag` | HCAL barrel/endcap (HBHE) RecHits |
| `doTenTau` | `bool` | Enable TenTau validation + matching studies |
| `doDYtoLL` | `bool` | Enable DYtoLL validation studies |

See `myTruthAnalyzer_cfg.py` for a complete job configuration, including
the Truth Graph prevalidation sequence
(`Validation.Configuration.truthPrevalidation_cff`) and the Phase-2 geometry
(`GeometryExtendedRun4D122Reco_cff`) that must be loaded before the analyzer
runs.

⚠️ The `truthToTrackster` `InputTag` instance label is direction-sensitive —
double check you are pointing at the Truth→RECO product
(`ticlCandidateTruthToReco`) and not one of the RECO→Truth products
(`ticlCandidateRecoToTruthFixed`/`...Adaptive*`), which are configured
separately.

## Usage

This plugin must be built and run inside a CMSSW area:

```bash
# from within $CMSSW_BASE/src, with this plugin's package checked out
scram b -j8

cmsRun path/to/cfg.py
```

The job produces a single ROOT file (`TFileService`, default name set in the
`cfg.py`, e.g. `myTenTau.root`) containing all booked histograms.

## Output histograms (selected)

Grouped by analysis stage (see source for the complete list):

- **Multiplicities & kinematics**: `GenTauNum`, `SimTauNum`, `TruthTauNum`,
  `GenPhtNum`, `GenEleNum`, `*Pt`, `*Eta`, `TauDecayVtxRadius`
- **Ancestry**: `GenTauMother`, `SimTauMother`, `GenPhtMother`, `SimPhtMother`,
  `GenEleMother`, `SimEleMother`
- **Detector response**: `TauSimHitE`, `TauRecHitE`, `TauEResponse`,
  `TauEResponseVsEta`, `TauEResponseVsPt`
- **EM/hadronic energy sharing** (τ → π± π0 ντ): `1ProngBFECALVsEta`,
  `1ProngBFHCALVsEta`, `1ProngECALoverHCALVsEta`, `1Pi0BFECALVsEta`, and the
  corresponding `VsPt` variants
- **Truth → RECO matching**: `AssociationScoreOfPionBestMatch`,
  `AssociationScoreOfPionBestMatchVsEta/Pt`, `TruthPionPtAll`,
  `TruthPionPtMatched`, `TruthToRecoEfficiency`
- **RECO → Truth matching**: `TrackesterToPionBestMatchNominal/Tight/Loose`,
  `TracksterPtAll`, `TracksterPtMatched`, `RecoToTruthEfficiency`
- **DYtoLL cross-checks**: `GenZMass`, `GenZPt`, `GenZEta`,
  `GenEleFromZPt/Eta`, `GenMuFromZPt/Eta`, `GenTauFromZPt/Eta`

Tau decay-mode branching fractions and matching-score bin summaries are also
printed to the log at `endJob()`.

## Known limitations / caveats

- Matching efficiency studies restrict truth pions and Tracksters to
  `1.5 < |η| < 3.0` (HGCAL acceptance); results outside this range are not
  covered.
- The "good match" threshold (association score `< 0.6`) is currently
  hardcoded rather than configurable.
- `doDYtoLL` matching-score summary printouts assume `allmatched > 0`; running
  on a sample with no matches will produce a division-by-zero in the log
  summary.
- Several `std::cout` debug statements are left in `analyze()`; consider
  routing these through `edm::LogInfo`/`edm::LogVerbatim` before wider use.

## References

- CMS Truth Graph documentation: <https://cms-truth.docs.cern.ch/data-model/>
- Identification of hadronic tau lepton decays using a deep neural network,
  *JINST* 17 (2022) P07023
