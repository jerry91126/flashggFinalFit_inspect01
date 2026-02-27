# flashggFinalFit Architecture Documentation

## System Architecture Overview

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CMS Analysis Chain                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CMS Detector → RECO → NanoAOD → HiggsDNA → flashggFinalFit     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    flashggFinalFit Package                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Input: ROOT files with diphoton candidates                     │
│          (Events, weights, categories, systematics)              │
│                                                                   │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │  Trees2WS    │  │              │  │              │          │
│   │  Converter   │  │              │  │              │          │
│   └──────┬───────┘  │              │  │              │          │
│          │          │              │  │              │          │
│          ↓          │              │  │              │          │
│   ┌──────────────┐  │              │  │              │          │
│   │ RooWorkspace │  │              │  │              │          │
│   │   Storage    │  │              │  │              │          │
│   └──────┬───────┘  │              │  │              │          │
│          ↓          │              │  │              │          │
│   ┌─────────────────┴──────┐    ┌─┴──────────────┐  │          │
│   │   Signal Modelling     │    │   Background   │  │          │
│   │   (Python + ROOT)      │    │   Modelling    │  │          │
│   │                        │    │   (C++/ROOT)   │  │          │
│   │ • F-Test              │    │                │  │          │
│   │ • Photon Systematics  │    │ • F-Test       │  │          │
│   │ • Signal Fitting      │    │ • MultiPdf     │  │          │
│   │ • Packaging           │    │                │  │          │
│   └────────────┬───────────┘    └────────┬───────┘  │          │
│                │                         │          │          │
│                └────────┬────────────────┘          │          │
│                         ↓                           │          │
│                ┌──────────────────┐                 │          │
│                │    Datacard      │                 │          │
│                │    Generator     │                 │          │
│                │                  │                 │          │
│                │ • Yields         │                 │          │
│                │ • Systematics    │                 │          │
│                │ • Text Format    │                 │          │
│                └────────┬─────────┘                 │          │
│                         ↓                           │          │
│                ┌──────────────────┐                 │          │
│                │  Combine Fits    │                 │          │
│                │                  │                 │          │
│                │ • Limits         │                 │          │
│                │ • Significance   │                 │          │
│                │ • Impacts        │                 │          │
│                │ • Scans          │                 │          │
│                └────────┬─────────┘                 │          │
│                         ↓                           │          │
│                ┌──────────────────┐                 │          │
│                │  Result Plots    │                 │          │
│                │                  │                 │          │
│                │ • Signal Models  │                 │          │
│                │ • S+B Models     │                 │          │
│                │ • Limit Plots    │                 │          │
│                └──────────────────┘                 │          │
│                                                                   │
│   Output: Publication-ready results and plots                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Detailed Component Architecture

### 1. Signal Module Architecture

```
Signal/
├── RunSignalScripts.py (Orchestrator)
│   │
│   ├─→ Mode: fTest
│   │   └─→ scripts/fTest.py
│   │       └─→ tools/fTest.py
│   │           • Determines optimal # of Gaussians
│   │           • Minimizes reduced χ²
│   │           • Outputs: JSON with nGaussians per (proc, cat)
│   │
│   ├─→ Mode: calcPhotonSyst
│   │   └─→ scripts/calcPhotonSyst.py
│   │       └─→ tools/calcPhotonSyst.py
│   │           • Calculates systematic effects
│   │           • Outputs: PKL DataFrames with constants
│   │
│   ├─→ Mode: getDiagProc
│   │   └─→ scripts/getDiagProc.py
│   │       • Identifies highest sum-of-weights process
│   │       • Outputs: JSON with diagonal process per cat
│   │
│   └─→ Mode: signalFit
│       └─→ scripts/signalFit.py
│           └─→ tools/simultaneousFit.py
│           └─→ tools/finalModel.py
│               • Simultaneous fit across mass points
│               • Polynomial interpolation
│               • RV/WV splitting
│               • Outputs: ROOT files with RooWorkspace
│
├── RunPackager.py
│   └─→ scripts/mergeWS.py
│       • Merges individual (proc,cat) workspaces
│       • Can merge across years
│       • Outputs: Single workspace per category
│
└── RunPlotter.py
    └─→ scripts/makePlot.py
        • Signal model visualization
        • Overlays different processes
        • Outputs: PDF/PNG plots
```

### 2. Background Module Architecture

```
Background/
├── src/
│   ├── FTest.cc              # Background function testing
│   ├── ProfileMultiplePdfs.cc # Multipdf creation
│   └── ResultContainer.cc     # Results storage
│
├── interface/
│   └── [Header files]
│
├── RunBackgroundScripts.py (Orchestrator)
│   │
│   └─→ Mode: fTestParallel
│       └─→ scripts/fTestParallel.py
│           └─→ bin/fTest (compiled C++)
│               • Tests multiple function families:
│               │   - Exponential
│               │   - Bernstein polynomials
│               │   - Laurent series
│               │   - Power law
│               • Goodness-of-fit criteria
│               • Outputs: RooMultiPdf per category
│
└── makefile
    └─→ Compiles C++ code
        Links with ROOT, RooFit, RooStats
```

### 3. Datacard Module Architecture

```
Datacard/
├── makeDatacard.py (Main Entry Point)
│   │
│   ├─→ inputs:
│   │   ├─ Signal workspaces (from Signal/)
│   │   ├─ Background workspaces (from Background/)
│   │   └─ Data workspace
│   │
│   ├─→ makeYields.py
│   │   └─→ tools/yields.py
│   │       • Calculates expected yields per bin
│   │       • Applies normalizations
│   │
│   ├─→ systematics.py
│   │   ├─→ theory_uncertainties/
│   │   │   └─→ [LHCHXSWG uncertainties]
│   │   └─→ Defines systematic structure
│   │
│   └─→ Output: Datacard_*.txt
│       Format:
│       ┌──────────────────────────┐
│       │ imax N categories        │
│       │ jmax M processes         │
│       │ kmax K systematics       │
│       │                          │
│       │ bin    cat1  cat2  ...   │
│       │ process ggH  VBF  ...    │
│       │ rate   X.XX  Y.YY  ...   │
│       │                          │
│       │ systematic_name shape... │
│       └──────────────────────────┘
│
└── tools/
    └─→ datacardClass.py
        • Object-oriented datacard manipulation
        • Add/remove/modify systematics
```

### 4. Combine Module Architecture

```
Combine/
├── RunFits.py (Main Orchestrator)
│   │
│   ├─→ Fit Types:
│   │   ├─ MaxLikelihoodFit (bestfit)
│   │   ├─ AsymptoticLimits
│   │   ├─ Significance
│   │   ├─ MultiDimFit (scans)
│   │   └─ Impacts
│   │
│   └─→ Calls combine tool:
│       $ combine -M [method] datacard.txt [options]
│
├── RunText2Workspace.py
│   └─→ Converts text datacard to binary workspace
│       $ text2workspace.py datacard.txt -o workspace.root
│
├── models.py
│   └─→ Physics model definitions
│       • POI definitions
│       • Parameter freezing
│       • Custom models
│
└── Checks/
    ├─→ Bias_nominal/
    │   └─→ Bias validation studies
    └─→ Bias_in_significance/
        └─→ Significance bias studies
```

### 5. Tools Module (Common Infrastructure)

```
tools/
├── commonObjects.py
│   └─→ Global Constants:
│       • Luminosity map
│       • Production modes
│       • Workspace names
│       • Branching ratios
│
├── commonTools.py
│   └─→ Utility Functions:
│       • rooiter(): Iterate over RooFit collections
│       • extractWSFileNames(): Glob workspace files
│       • extractListOfProcs(): Parse process names
│       • extractListOfCats(): Parse categories
│       • signalFromFileName(): Extract production mode
│       • procToData() / dataToProc(): Name conversions
│       • procToDatacardName(): Convert to datacard format
│
├── replacementMap.py
│   └─→ Replacement Strategy:
│       • Maps (proc, cat) → replacement (proc, cat)
│       • Used when statistics < threshold
│       • Analysis-specific mappings
│
├── XSBRMap.py
│   └─→ Normalization Factors:
│       • σ × BR from theory
│       • Process-specific factors
│       • Mass-dependent calculations
│
├── simultaneousFit.py
│   └─→ Signal Fitting Engine:
│       • Multi-mass-point fitting
│       • scipy.optimize.minimize
│       • Constraint handling
│
└── finalModel.py
    └─→ Signal Model Builder:
        • Constructs RooWorkspace
        • Adds systematics
        • Polynomial interpolation
        • RV/WV splitting
```

## Data Flow Architecture

### Signal Data Flow

```
Input ROOT Files
    │
    ├─ output_GluGluHToGG_M125.root
    ├─ output_VBFHToGG_M125.root
    └─ ...
    │
    ↓
[Extract from RooWorkspace]
    │
    ├─ Process: ggh_125_13TeV_cat0
    ├─ Process: vbf_125_13TeV_cat0
    └─ ...
    │
    ↓
[F-Test]
    │
    └─ Determine nGaussians per process
       → JSON: fTest_results.json
    │
    ↓
[Photon Systematics]
    │
    └─ Calculate systematic effects
       → PKL: photonSystematics_cat0.pkl
    │
    ↓
[Signal Fit]
    │
    ├─ Build simultaneous likelihood
    ├─ Fit parameters: μ, σ, fractions
    ├─ Polynomial interpolation for mass
    └─ Add systematic variations
       → ROOT: output_ggh_cat0.root
    │
    ↓
[Packaging]
    │
    └─ Merge all processes for category
       → ROOT: signal_cat0.root
```

### Background Data Flow

```
allData.root
    │
    ├─ Data_13TeV_cat0
    ├─ Data_13TeV_cat1
    └─ ...
    │
    ↓
[Background F-Test]
    │
    ├─ Test function families:
    │   ├─ Exponential: exp(-k*x)
    │   ├─ Bernstein: Σ bᵢBᵢ(x)
    │   ├─ Laurent: Σ aᵢx^(-i)
    │   └─ Power: x^(-α)*exp(-β*x)
    │
    ├─ Goodness-of-fit tests
    │
    └─ Select passing functions
       → ROOT: RooMultiPdf per category
    │
    ↓
[Discrete Profiling in Combine]
    │
    └─ Treat function choice as nuisance
       with uniform prior
```

### Datacard Creation Data Flow

```
Inputs:
    ├─ Signal workspaces
    ├─ Background workspaces
    └─ Data workspace
    │
    ↓
[makeDatacard.py]
    │
    ├─ [1] Read signal normalizations
    │      • Get (σ × BR) from XSBRMap
    │      • Get (ε × A) from workspace
    │      • Calculate yield = (σ×BR)×(ε×A)×L
    │
    ├─ [2] Read background normalizations
    │      • Extract from RooMultiPdf
    │
    ├─ [3] Construct rate table
    │
    ├─ [4] Add systematics
    │      ├─ Theory: from theory_uncertainties/
    │      ├─ Photon: from signal workspaces
    │      ├─ Experimental: from systematics.py
    │      └─ Background: flatParam for norm
    │
    └─ [5] Write text datacard
       → TXT: Datacard_analysis_year_cat.txt
```

## Execution Flow

### Signal Workflow Sequence

```
Step 1: Configuration
    ├─ Edit config_*.py
    └─ Set: inputWSDir, procs, cats, ext, year, massPoints

Step 2: F-Test
    $ python3 RunSignalScripts.py --inputConfig config.py --mode fTest
    ├─ Submit jobs (one per category)
    ├─ Wait for completion
    └─ Check: outdir_ext/fTest/json/fTest_results.json

Step 3: Photon Systematics
    $ python3 RunSignalScripts.py --inputConfig config.py --mode calcPhotonSyst
    ├─ Submit jobs (one per category)
    ├─ Wait for completion
    └─ Check: outdir_ext/calcPhotonSyst/dat/*.pkl

Step 4: Diagonal Process (optional)
    $ python3 RunSignalScripts.py --inputConfig config.py --mode getDiagProc
    └─ Check: outdir_ext/getDiagProc/json/diagonal_procs.json

Step 5: Signal Fit
    $ python3 RunSignalScripts.py --inputConfig config.py --mode signalFit
    ├─ Submit jobs (one per category or per proc×cat)
    ├─ Wait for completion
    └─ Check: outdir_ext/signalFit/output_*.root

Step 6: Packaging
    $ python3 RunPackager.py --cats auto --inputWSDir /path --exts ext1,ext2 [--mergeYears]
    └─ Output: outdir_packaged/CMS-HGG_sigfit_cat*.root

Step 7: Plotting
    $ python3 RunPlotter.py --procs all --cats cat0 --years 2016,2017,2018
    └─ Output: outdir_plots/*.pdf
```

### Background Workflow Sequence

```
Step 1: Compilation
    $ cd Background
    $ make clean && make

Step 2: Configuration
    ├─ Edit config_*.py
    └─ Set: inputWSDir (path to allData.root), cats, ext

Step 3: F-Test
    $ python3 RunBackgroundScripts.py --inputConfig config.py --mode fTestParallel
    ├─ Submit jobs (one per category)
    ├─ Wait for completion
    └─ Check: outdir_ext/CMS-HGG_multipdf_*.root

Step 4: Copy to Combine
    $ cp outdir_ext/CMS-HGG_multipdf_*.root ../Combine/
```

### Full Analysis Sequence

```
[Parallel Execution]
    ├─ [Signal Workflow] → Signal workspaces
    └─ [Background Workflow] → Background workspaces
        │
        ↓
[Datacard Creation]
    $ cd Datacard
    $ python3 makeDatacard.py --config config.py
        │
        ↓ Datacard_*.txt
        │
[Text2Workspace]
    $ cd Combine
    $ python3 RunText2Workspace.py --inputConfig inputs.json
        │
        ↓ workspace.root
        │
[Run Fits]
    $ python3 RunFits.py --inputConfig inputs.json --mode all
        │
        ├─ MaxLikelihoodFit → bestfit results
        ├─ AsymptoticLimits → limit tree
        ├─ Significance → significance value
        └─ Impacts → impact plots
        │
        ↓
[Result Plots]
    $ cd ../Plots
    $ python3 makePlots.py [various scripts]
        │
        └─ Publication-quality plots
```

## Job Submission Architecture

### Batch System Integration

```
┌─────────────────────────────────────────┐
│         RunSignalScripts.py             │
│                                         │
│  1. Parse config                        │
│  2. Determine jobs needed               │
│  3. Create job scripts                  │
│     └─→ jobs/sub_{job_id}.sh           │
│  4. Create submission script            │
│     └─→ jobs/sub_all.sh                │
└─────────────┬───────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────┐
│      Batch System Selection             │
├─────────────────────────────────────────┤
│                                         │
│  if batch == 'condor':                  │
│    └─→ Create .sub file                │
│        └─→ condor_submit jobs.sub      │
│                                         │
│  elif batch == 'SGE':                   │
│    └─→ qsub -q {queue} job.sh          │
│                                         │
│  elif batch == 'IC':                    │
│    └─→ qsub -l h_vmem=12G job.sh       │
│                                         │
│  elif batch == 'local':                 │
│    └─→ bash job.sh                     │
│                                         │
└─────────────┬───────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────┐
│        Job Execution                    │
├─────────────────────────────────────────┤
│                                         │
│  Setup CMSSW environment                │
│  cd $CMSSW_BASE/src/flashggFinalFit    │
│  python3 scripts/{mode}.py {args}       │
│                                         │
│  Output → outdir_{ext}/{mode}/output/  │
│  Logs   → outdir_{ext}/{mode}/logs/    │
│                                         │
└─────────────────────────────────────────┘
```

## Error Handling and Debugging

### Common Issues Flow

```
Issue: Job fails
    │
    ├─→ Check log file
    │   └─ outdir_ext/{mode}/logs/log_{job_id}.log
    │
    ├─→ Check job script
    │   └─ outdir_ext/{mode}/jobs/sub_{job_id}.sh
    │
    ├─→ Run locally for debugging
    │   $ bash outdir_ext/{mode}/jobs/sub_{job_id}.sh
    │
    └─→ Common errors:
        ├─ Missing input file → Check inputWSDir
        ├─ RooFit errors → Check workspace contents
        ├─ Memory issues → Increase memory allocation
        └─ Timeout → Increase wall time / split jobs
```

## Performance Considerations

### Parallel Execution Strategy

1. **F-Test**: One job per category (fully parallelizable)
2. **Photon Systematics**: One job per category (fully parallelizable)
3. **Signal Fit**: 
   - Default: One job per (process × category)
   - With `--groupSignalFitJobsByCat`: One job per category
   - Trade-off: More jobs = more parallelism, but more overhead

### Resource Requirements

```
Typical Job Resources:
├─ F-Test:
│   ├─ CPU: 1 core
│   ├─ Memory: 2-4 GB
│   └─ Time: 5-30 minutes per category
│
├─ Signal Fit:
│   ├─ CPU: 1 core
│   ├─ Memory: 1-2 GB
│   └─ Time: 1-10 minutes per (proc,cat)
│
└─ Background F-Test:
    ├─ CPU: 1 core
    ├─ Memory: 4-8 GB
    └─ Time: 30-120 minutes per category
```

## Summary

The flashggFinalFit architecture is designed around:

1. **Modularity**: Clear separation of signal, background, datacard, and fitting stages
2. **Scalability**: Parallel job submission for all major steps
3. **Flexibility**: Configuration-driven, supports multiple analyses
4. **Robustness**: Error handling, logging, debugging support
5. **Integration**: Seamless connection with CMS combine tool

The package transforms raw diphoton data into publication-ready physics results through a well-defined pipeline with clear data flows and execution sequences.
