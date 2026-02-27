# flashggFinalFit Package Investigation

## Executive Summary

This document provides a comprehensive investigation and understanding of the **flashggFinalFit** package, which is used for the final stages of CMS Higgs to two photons (Hgg) analyses. The package handles signal modeling, background modeling, datacard creation, statistical interpretation, and result plotting.

## 1. Package Overview

### 1.1 Purpose
The flashggFinalFit package is designed for:
- **Signal Modelling**: Fitting signal processes to determine shape and normalization
- **Background Modelling**: Determining background shapes using data-driven methods
- **Datacard Creation**: Generating combine datacards for statistical analysis
- **Statistical Analysis**: Running final fits with the CMS combine tool
- **Visualization**: Producing publication-quality plots

### 1.2 Context
- **Branch**: `higgsdnafinalfit` - adapted for HiggsDNA output (vs. flashgg)
- **CMS Experiment**: Part of the Large Hadron Collider (LHC) physics program
- **Physics Target**: Higgs boson decaying to two photons (H→γγ)
- **Analysis Framework**: Integrates with CMS combine tool and CombineHarvester

## 2. Architecture and Directory Structure

### 2.1 Main Components

```
flashggFinalFit/
├── Signal/          # Signal model fitting and packaging
├── Background/      # Background model determination (multipdf)
├── Datacard/        # Datacard generation and yields calculation
├── Combine/         # Running combine fits and statistical tests
├── Plots/           # Visualization scripts for results
├── Trees2WS/        # Convert ROOT trees to RooWorkspace
├── tools/           # Common utilities and objects
└── tdrStyle/        # CMS TDR plotting style
```

### 2.2 Directory Purposes

#### Signal Directory
- **Purpose**: Parametric signal modeling
- **Input**: RooWorkspace files from HiggsDNA
- **Output**: Signal model workspaces with shape and normalization
- **Key Scripts**:
  - `RunSignalScripts.py`: Main orchestrator for signal workflows
  - `RunPackager.py`: Packages individual signal models
  - `RunPlotter.py`: Creates signal model plots

#### Background Directory
- **Purpose**: Data-driven background estimation
- **Input**: Data workspace (`allData.root`)
- **Output**: RooMultiPdf background models
- **Key Scripts**:
  - `RunBackgroundScripts.py`: Orchestrates background workflows
- **Technology**: C++ with ROOT, requires compilation

#### Datacard Directory
- **Purpose**: Generate combine datacards
- **Input**: Signal models + background models + data
- **Output**: Text datacards for combine
- **Key Scripts**:
  - `makeDatacard.py`: Main datacard creation
  - `makeYields.py`: Calculate expected yields
  - `systematics.py`: Define systematic uncertainties

#### Combine Directory
- **Purpose**: Statistical inference
- **Input**: Datacards from Datacard directory
- **Output**: Fit results, limits, significance, impacts
- **Key Scripts**:
  - `RunFits.py`: Execute various fit types
  - `RunText2Workspace.py`: Convert datacards to workspaces

#### Plots Directory
- **Purpose**: Result visualization
- **Output**: Publication-quality plots

## 3. Workflow and Data Flow

### 3.1 Standard Analysis Workflow

```
HiggsDNA Output (ROOT files)
        ↓
Trees2WS: Convert to RooWorkspace
        ↓
    ┌───────┴───────┐
    ↓               ↓
Signal Modeling   Background Modeling
    ↓               ↓
    └───────┬───────┘
            ↓
    Datacard Creation
            ↓
    Combine Fitting
            ↓
    Result Plots
```

### 3.2 Detailed Signal Workflow

1. **F-Test** (`mode=fTest`): Determine optimal number of Gaussians
2. **Photon Systematics** (`mode=calcPhotonSyst`): Calculate systematic effects
3. **Diagonal Process** (`mode=getDiagProc`): Identify dominant processes per category
4. **Signal Fit** (`mode=signalFit`): Perform the actual signal fitting
5. **Packaging** (`RunPackager.py`): Combine individual fits into category files
6. **Plotting** (`RunPlotter.py`): Visualize signal models

### 3.3 Background Workflow

1. **F-Test Parallel** (`mode=fTestParallel`): Test multiple background functions
2. **Output**: RooMultiPdf containing passing background functions
3. **Discrete Profiling**: Background function choice treated as nuisance parameter

## 4. Key Concepts and Terminology

### 4.1 Physics Concepts

- **Signal Processes**: Different Higgs production modes
  - `ggH`: Gluon-gluon fusion
  - `VBF/qqH`: Vector boson fusion
  - `WH/ZH`: Associated production with W/Z bosons
  - `ttH`: Associated production with top quarks
  - `tHq/tHW`: Single top associated production
  - `ggZH/bbH`: Rare production modes

- **Categories**: Analysis regions with different S/B ratios
- **STXS**: Simplified Template Cross Sections framework

### 4.2 Statistical Concepts

- **RooWorkspace**: ROOT framework for statistical modeling
- **RooMultiPdf**: Collection of alternative background models
- **Discrete Profiling**: Treating model choice as discrete nuisance
- **Asimov Dataset**: Expected (median) dataset for sensitivity studies

### 4.3 Systematic Uncertainties

- **Photon Energy Scale**: Affects signal mean
- **Photon Energy Resolution**: Affects signal width  
- **Theory Uncertainties**: Cross section, PDF, scale variations
- **Experimental**: Luminosity, trigger, selection efficiencies

## 5. Code Structure Analysis

### 5.1 Common Tools (`tools/`)

#### `commonObjects.py`
- Defines global constants and configuration
- Luminosity values for different data-taking periods
- Production mode and decay channel definitions
- Workspace naming conventions

```python
# Key objects
lumiMap = {'2016':36.33, '2017':41.48, '2018':59.83, ...}
productionModes = ['ggH','qqH','ttH','tHq','tHW','ggZH','WH','ZH','bbH']
inputWSName__ = "tagsDumper/cms_hgg_13TeV"
```

#### `commonTools.py`
- Utility functions for workspace manipulation
- Process name conversions
- File name parsing
- ROOT object iteration

### 5.2 Configuration System

Analyses are configured via Python config files:
```python
signalScriptCfg = {
    'inputWSDir': '/path/to/workspaces',
    'procs': 'auto',  # Auto-detect from files
    'cats': 'auto',   # Auto-detect from workspace
    'ext': 'tutorial_2022preEE',
    'year': '2022preEE',
    'massPoints': '120,125,130',
    'batch': 'condor',
    'queue': 'espresso'
}
```

### 5.3 Job Submission

Supports multiple batch systems:
- `condor`: HTCondor
- `SGE`: Sun Grid Engine
- `IC`: Imperial College specific
- `local`: Run locally

## 6. Technical Details

### 6.1 Dependencies

**Core Requirements**:
- CMSSW (CMS Software framework)
- ROOT 6+ with RooFit
- Python 3
- HiggsAnalysis-CombinedLimit (combine tool)
- CombineHarvester/CombineTools

**Python Libraries**:
- NumPy, SciPy (for fitting)
- Pandas (data manipulation)
- Matplotlib (plotting)

### 6.2 Data Formats

**Input Formats**:
- ROOT files with TTree structure from HiggsDNA
- RooWorkspace files for signal/background/data

**Output Formats**:
- ROOT files with RooWorkspace
- Text datacards for combine
- JSON files for configuration/results
- Pickle files for pandas DataFrames

### 6.3 Key Algorithms

#### Signal Fitting
- Uses `scipy.optimize.minimize` for parameter optimization
- Simultaneous fit across multiple mass points
- Polynomial interpolation for mass dependence
- Split into Right Vertex (RV) and Wrong Vertex (WV) scenarios

#### Background Modeling
- F-test to select appropriate function families
- Multiple function families tested:
  - Exponential
  - Bernstein polynomials
  - Laurent series
  - Power law
- Goodness-of-fit criteria for function selection

## 7. Usage Examples

### 7.1 Signal Modeling

```bash
# Run F-test
python3 RunSignalScripts.py --inputConfig config_tutorial_2022preEE.py --mode fTest

# Calculate photon systematics
python3 RunSignalScripts.py --inputConfig config_tutorial_2022preEE.py --mode calcPhotonSyst

# Get diagonal processes
python3 RunSignalScripts.py --inputConfig config_tutorial_2022preEE.py --mode getDiagProc

# Run signal fit
python3 RunSignalScripts.py --inputConfig config_tutorial_2022preEE.py --mode signalFit

# Package signal models
python3 RunPackager.py --cats auto --inputWSDir /path/to/ws --exts test_2016,test_2017,test_2018 --mergeYears
```

### 7.2 Background Modeling

```bash
# Build the package
cd Background
make

# Run background F-test
python3 RunBackgroundScripts.py --inputConfig config_test.py --mode fTestParallel
```

### 7.3 Datacard Creation

```bash
cd Datacard
python3 makeDatacard.py --config config.py
```

## 8. Important Features

### 8.1 Year Merging
- Models can be created per year or merged
- Allows proper treatment of year-dependent systematics
- Recommended: Create separate models per year, merge at datacard stage

### 8.2 Mass Point Handling
- Can fit single mass point or multiple points
- Polynomial interpolation for mass-dependent parameters
- Default: Linear interpolation (order 1)
- Single mass point: Automatically set to constant (order 0)

### 8.3 Replacement Datasets
- For processes with low statistics
- Uses diagonal process or similar category
- Threshold: Default 100 events
- Defined in `tools/replacementMap.py`

### 8.4 Normalization
- Signal: (σ × BR) × (ε × A) × L
- Cross sections from LHCHXSWG
- Efficiency × Acceptance from HiggsDNA sum of weights
- Defined in `tools/XSBRMap.py`

## 9. Known Issues and Limitations

### 9.1 Background Package
- Not fully pythonized (C++ with makefiles)
- Requires compilation
- Pseudo-data functionality not yet ported to HiggsDNA branch

### 9.2 High S/B Categories
- Background normalization includes signal pre-fit
- Can affect expected sensitivity in very high S/B cases
- Workaround: Use post-fit background for Asimov toys

### 9.3 Version Compatibility
- Requires specific CMSSW version (14_1_0_pre4 for EL9)
- Specific combine and CombineHarvester commits

## 10. Development Considerations

### 10.1 Extending the Package

**Adding New Signal Processes**:
1. Update `tools/commonTools.signalFromFileName()`
2. Update `tools/replacementMap.py`
3. Update `tools/XSBRMap.py`

**Adding New Systematics**:
1. Define in config file (`scales`, `smears`, etc.)
2. Ensure proper naming in workspace
3. Add to datacard systematics mapping

**Custom Analysis Categories**:
1. Define replacement map in `tools/replacementMap.py`
2. Define XS×BR map in `tools/XSBRMap.py`
3. Create analysis-specific config file

### 10.2 Code Quality

**Strengths**:
- Modular design with clear separation of concerns
- Configuration-driven approach
- Parallel job submission support
- Comprehensive documentation in README files

**Areas for Improvement**:
- Background package should be pythonized
- More unit tests needed
- Some hardcoded paths/assumptions
- Documentation could be centralized

## 11. Resources

### 11.1 Tutorials
- [Latest HiggsDNA Final Fits Tutorial](https://gitlab.cern.ch/jspah/higgsdna_finalfits_tutorial_24/-/tree/master)
- [Older Flashgg Tutorial Slides](https://indico.cern.ch/event/963619/contributions/4112177/attachments/2151275/3627204/finalfits_tutorial_201126.pdf)

### 11.2 Related Tools
- [Combine Documentation](https://cms-analysis.github.io/HiggsAnalysis-CombinedLimit/)
- [CombineHarvester](https://github.com/cms-analysis/CombineHarvester)
- [HiggsDNA](https://github.com/maxgalli/HiggsDNA)

### 11.3 Physics References
- LHC Higgs Cross Section Working Group recommendations
- CMS Higgs to two photons results

## 12. Quick Reference

### 12.1 Common Commands

```bash
# Setup environment
export SCRAM_ARCH=el9_amd64_gcc12
cd CMSSW_14_1_0_pre4/src
cmsenv
cd flashggFinalFit
source setup.sh

# Test mode (print jobs without submission)
python3 RunSignalScripts.py --inputConfig config.py --mode fTest --printOnly

# Check output directories
ls -la outdir_*/

# Common debugging
# - Check log files in outdir_*/mode/logs/
# - Check job scripts in outdir_*/mode/jobs/
# - Run individual jobs locally for testing
```

### 12.2 File Naming Conventions

**Signal Workspaces**:
- Input: `output_*.root` (from HiggsDNA)
- Contains process name: `pythia8_<process>.root`
- Output: `output_<proc>_<cat>.root` (per process-category)

**Background Workspaces**:
- Input: `allData.root`
- Output: `CMS-HGG_multipdf_<ext>.root` (per category)

**Datacards**:
- Format: `Datacard_<analysis>_<year>_<category>.txt`

## 13. Conclusion

The flashggFinalFit package is a comprehensive, production-grade analysis framework for CMS Hgg analyses. It implements sophisticated statistical methods for signal and background modeling, with careful attention to systematic uncertainties and data-driven techniques. While primarily designed for Higgs to two photons, its modular structure makes it adaptable to other similar analyses.

### Key Takeaways:
1. **Modular Design**: Clear separation between signal, background, datacard, and fitting stages
2. **Configuration-Driven**: Flexible configuration system for different analyses
3. **Production Ready**: Supports batch submission, parallel processing, year merging
4. **Well-Documented**: Comprehensive README files in each directory
5. **Active Development**: Ongoing improvements and bug fixes

### Recommended Next Steps:
1. Follow the tutorial to understand the full workflow
2. Familiarize yourself with ROOT and RooFit
3. Study the configuration files for your analysis
4. Run a test analysis with tutorial data
5. Understand the output structure and validation plots
