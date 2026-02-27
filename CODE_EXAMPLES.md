# flashggFinalFit Code Examples and Best Practices

## Table of Contents
1. [Basic Workflows](#basic-workflows)
2. [Configuration Examples](#configuration-examples)
3. [Common Tasks](#common-tasks)
4. [Advanced Usage](#advanced-usage)
5. [Debugging Techniques](#debugging-techniques)
6. [Best Practices](#best-practices)
7. [Common Pitfalls](#common-pitfalls)

## Basic Workflows

### Complete Signal Modeling Workflow

```bash
#!/bin/bash
# complete_signal_workflow.sh

# 1. Setup environment
export SCRAM_ARCH=el9_amd64_gcc12
cd $CMSSW_BASE/src/flashggFinalFit
source setup.sh
cd Signal

# 2. Create configuration file
# Edit config_myanalysis_2022preEE.py with your settings

# 3. Test mode (print jobs without submission)
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode fTest \
    --printOnly

# 4. Run F-test
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode fTest \
    --modeOpts "--doPlots"

# Wait for jobs to complete, then check output
ls -l outdir_myanalysis_2022preEE/fTest/json/
cat outdir_myanalysis_2022preEE/fTest/json/fTest_results.json

# 5. Calculate photon systematics
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode calcPhotonSyst

# Wait for completion
ls -l outdir_myanalysis_2022preEE/calcPhotonSyst/dat/

# 6. Get diagonal processes (optional but recommended)
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode getDiagProc

# 7. Run signal fit
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode signalFit \
    --groupSignalFitJobsByCat \
    --modeOpts "--doPlots --useDiagonalProcForSyst --skipVertexScenarioSplit"

# Wait for completion
ls -l outdir_myanalysis_2022preEE/signalFit/output/

# 8. Package the results
cd ..
python3 RunPackager.py \
    --cats auto \
    --inputWSDir /path/to/input/workspaces \
    --exts myanalysis_2022preEE \
    --outputExt myanalysis_2022preEE_packaged

# 9. Make plots
python3 RunPlotter.py \
    --procs all \
    --cats cat0,cat1,cat2 \
    --years 2022preEE \
    --ext myanalysis_2022preEE_packaged
```

### Complete Background Modeling Workflow

```bash
#!/bin/bash
# complete_background_workflow.sh

cd $CMSSW_BASE/src/flashggFinalFit/Background

# 1. Build the package (first time only)
make clean
make

# Check compilation succeeded
if [ $? -ne 0 ]; then
    echo "Compilation failed!"
    exit 1
fi

# 2. Run background F-test
python3 RunBackgroundScripts.py \
    --inputConfig config_myanalysis.py \
    --mode fTestParallel \
    --printOnly  # Remove this after checking

python3 RunBackgroundScripts.py \
    --inputConfig config_myanalysis.py \
    --mode fTestParallel

# Wait for jobs to complete
# Check output files
ls -l outdir_myanalysis/CMS-HGG_multipdf_*.root

# 3. Copy background models to Combine directory
cp outdir_myanalysis/CMS-HGG_multipdf_*.root ../Combine/
```

### Complete End-to-End Workflow

```bash
#!/bin/bash
# complete_analysis.sh

ANALYSIS_NAME="tutorial"
YEAR="2022preEE"
CONFIG_EXT="${ANALYSIS_NAME}_${YEAR}"

# Stage 1: Signal and Background modeling (parallel)
cd $CMSSW_BASE/src/flashggFinalFit

# Signal modeling
(cd Signal && \
    python3 RunSignalScripts.py --inputConfig config_${CONFIG_EXT}.py --mode fTest && \
    python3 RunSignalScripts.py --inputConfig config_${CONFIG_EXT}.py --mode calcPhotonSyst && \
    python3 RunSignalScripts.py --inputConfig config_${CONFIG_EXT}.py --mode getDiagProc && \
    python3 RunSignalScripts.py --inputConfig config_${CONFIG_EXT}.py --mode signalFit --groupSignalFitJobsByCat \
) &

# Background modeling
(cd Background && \
    python3 RunBackgroundScripts.py --inputConfig config_${ANALYSIS_NAME}.py --mode fTestParallel \
) &

# Wait for both to complete
wait

# Stage 2: Packaging
python3 Signal/RunPackager.py \
    --cats auto \
    --inputWSDir /path/to/input/ws \
    --exts ${CONFIG_EXT}

# Copy background models
cp Background/outdir_${ANALYSIS_NAME}/CMS-HGG_multipdf_*.root Combine/

# Stage 3: Datacard creation
cd Datacard
python3 makeDatacard.py \
    --years ${YEAR} \
    --signalWSDir ../Signal/outdir_packaged \
    --backgroundWSDir ../Combine \
    --dataWSDir /path/to/data/allData.root \
    --outputDir datacards_${CONFIG_EXT}

# Stage 4: Run fits
cd ../Combine
python3 RunText2Workspace.py \
    --inputConfig inputs_${CONFIG_EXT}.json

python3 RunFits.py \
    --inputConfig inputs_${CONFIG_EXT}.json \
    --mode all

# Stage 5: Make plots
cd ../Plots
python3 makeSplusBModelPlot.py \
    --datacard ../Datacard/datacards_${CONFIG_EXT}/Datacard.txt \
    --workspace ../Combine/workspace.root
```

## Configuration Examples

### Signal Configuration

```python
# config_tutorial_2022preEE.py

_year = '2022preEE'
_analysis = 'tutorial'
_input_ws_dir = '/eos/user/j/jlangfor/hgg/workspaces/signal_%s' % _year

signalScriptCfg = {
    # ==================== Setup ====================
    # Path to input workspace directory containing output_*.root files
    'inputWSDir': _input_ws_dir,
    
    # Process list: 'auto' or comma-separated list like 'ggh,vbf,wh,zh'
    'procs': 'auto',
    
    # Category list: 'auto' or comma-separated list like 'cat0,cat1,cat2'
    'cats': 'auto',
    
    # Extension for output directory name (outdir_{ext})
    'ext': '%s_%s' % (_analysis, _year),
    
    # Analysis name (used to select replacement map and XSBR map)
    'analysis': _analysis,
    
    # Year or 'combined' for merged years
    'year': _year,
    
    # Mass points to fit (comma-separated, no spaces)
    'massPoints': '120,125,130',
    
    # ==================== Photon Systematics ====================
    # Separate nuisance per year
    'scales': 'Scale',
    'scalesCorr': '',  # Correlated across years
    'scalesGlobal': '',  # Affect all processes equally
    'smears': 'Smearing',
    
    # ==================== Job Submission ====================
    'batch': 'condor',  # Options: 'condor', 'SGE', 'IC', 'local'
    'queue': 'espresso',  # For condor: espresso, microcentury, longlunch, workday
}

# Additional mode-specific options can be added via --modeOpts
# Example: --modeOpts "--doPlots --useDCB --skipVertexScenarioSplit"
```

### Background Configuration

```python
# config_tutorial.py

_analysis = 'tutorial'
_input_data_dir = '/eos/user/j/jlangfor/hgg/workspaces/data'

backgroundScriptCfg = {
    # ==================== Setup ====================
    # Path to allData.root file (or directory containing it)
    'inputWSDir': _input_data_dir,
    
    # Category list: 'auto' or comma-separated list
    'cats': 'auto',
    
    # Category number offset (useful when merging multiple allData.root files)
    'catOffset': 0,
    
    # Extension for output directory
    'ext': _analysis,
    
    # Year for luminosity in plots
    'year': 'combined',  # or specific year like '2022preEE'
    
    # ==================== Job Submission ====================
    'batch': 'condor',
    'queue': 'microcentury',
}
```

### Datacard Configuration Example

```python
# Example of systematics configuration
import systematics as systs

# Define categories
categories = ['cat0', 'cat1', 'cat2']

# Define signal processes
signal_procs = {
    'ggH': 1.0,
    'qqH': 1.0,
    'WH_had': 1.0,
    'ZH_had': 1.0,
    'WH_lep': 1.0,
    'ZH_lep': 1.0,
    'ttH': 1.0,
}

# Add theory uncertainties
theory_uncertainties = {
    'THU_ggH_Mu': {'type': 'lnN', 'ggH': 1.039},
    'THU_ggH_Res': {'type': 'lnN', 'ggH': 1.046},
    'THU_ggH_Mig01': {'type': 'lnN', 'ggH': 1.10},
    'THU_qqH': {'type': 'lnN', 'qqH': 1.005},
}

# Photon systematics (from signal workspace)
photon_systematics = {
    'CMS_hgg_scale': 'shape',
    'CMS_hgg_smearing': 'shape',
}

# Experimental systematics
experimental_systematics = {
    'lumi': {'type': 'lnN', 'all_signal': 1.016},  # 1.6% for 2022
    'trigger': {'type': 'lnN', 'all_signal': 1.01},
}
```

## Common Tasks

### Task 1: Adding a New Signal Process

```python
# Step 1: Update tools/commonTools.py
def signalFromFileName(_fileName):
    # ... existing code ...
    elif "MyNewProcess" in _fileName:
        p = "mynewproc"
        d = None  # Or specific decay mode
    # ... rest of code ...

# Step 2: Update tools/replacementMap.py
replacementMap = {
    'myanalysis': {
        # Define replacement for new process
        ('mynewproc', 'cat0'): ('ggh', 'cat0'),  # Use ggh as fallback
        # ... other mappings ...
    }
}

# Step 3: Update tools/XSBRMap.py
XSBRMap = {
    'myanalysis': {
        # Define normalization for new process
        'mynewproc': {
            'mode': 'constant',
            'sigma': 0.001,  # pb
            'BR': 1.0,
        },
        # ... other processes ...
    }
}

# Step 4: Ensure input files follow naming convention
# output_MyNewProcess_M125_pythia8.root
```

### Task 2: Merging Multiple Years

```bash
# After running signal fits for each year separately

# Package each year
for year in 2016 2017 2018; do
    python3 Signal/RunPackager.py \
        --cats auto \
        --inputWSDir /path/to/input \
        --exts myanalysis_${year} \
        --outputExt myanalysis_${year}_packaged
done

# Merge years
python3 Signal/RunPackager.py \
    --cats cat0,cat1,cat2 \
    --exts myanalysis_2016_packaged,myanalysis_2017_packaged,myanalysis_2018_packaged \
    --mergeYears \
    --outputExt myanalysis_combined

# Now use myanalysis_combined for datacard creation
```

### Task 3: Running Single Mass Point

```python
# In configuration file
signalScriptCfg = {
    # ... other settings ...
    'massPoints': '125',  # Only fit at 125 GeV
}

# The code will automatically set polynomial order to 0 (constant)
# No mass interpolation will be performed
```

### Task 4: Using DCB Instead of Gaussian

```bash
# Use Double Crystal Ball + Gaussian instead of N Gaussians
python3 RunSignalScripts.py \
    --inputConfig config.py \
    --mode signalFit \
    --modeOpts "--useDCB --doPlots"

# This skips the F-test step entirely
```

### Task 5: Debugging a Specific Category

```bash
# Run signal fit for single category locally
cd Signal/outdir_ext/signalFit/jobs

# Edit the submission script for your category
# Find: sub_cat0.sh (or similar)

# Run it locally
bash sub_cat0.sh

# Check the output
ls -l ../output/output_*_cat0.root
cat ../logs/log_cat0.log
```

### Task 6: Custom Systematic Uncertainty

```python
# In Datacard/systematics.py

# Add custom systematic
custom_systematics = {
    'my_custom_syst': {
        'type': 'lnN',  # or 'shape' for shape systematic
        'processes': {
            'ggH': 1.05,  # 5% effect on ggH
            'qqH': 1.03,  # 3% effect on qqH
        },
        'categories': ['cat0', 'cat1'],  # Only in these categories
    }
}

# For shape systematic, ensure it's in the signal workspace
# with naming: CMS_hgg_my_custom_syst[Up/Down]
```

## Advanced Usage

### Advanced 1: Using Diagonal Process for Shape

```bash
# Step 1: Get diagonal processes
python3 RunSignalScripts.py \
    --inputConfig config.py \
    --mode getDiagProc

# Step 2: Use diagonal shape in signal fit
python3 RunSignalScripts.py \
    --inputConfig config.py \
    --mode signalFit \
    --modeOpts "--useDiagonalProcForShape --useDiagonalProcForSyst"

# This uses the highest-statistics process for both shape and systematics
# Useful when some processes have very low statistics
```

### Advanced 2: Custom Minimizer Settings

```python
# In scripts/signalFit.py or via command line

# Change minimizer method
--modeOpts "--minimizerMethod BFGS --minimizerTolerance 1e-6"

# Available methods in scipy.optimize.minimize:
# - Nelder-Mead
# - Powell
# - CG (Conjugate Gradient)
# - BFGS (default)
# - L-BFGS-B
# - SLSQP
```

### Advanced 3: Beamspot Reweighting

```bash
# Adjust beamspot width for MC and data
python3 RunSignalScripts.py \
    --inputConfig config.py \
    --mode signalFit \
    --modeOpts "--beamspotWidthMC 5.0 --beamspotWidthData 4.5"

# Or skip reweighting entirely
python3 RunSignalScripts.py \
    --inputConfig config.py \
    --mode signalFit \
    --modeOpts "--skipBeamspotReweigh"
```

### Advanced 4: Partial Fits and Caching

```bash
# Run signal fit only for specific processes
# Edit the job submission script to comment out unwanted processes

cd Signal/outdir_ext/signalFit/jobs
# Edit sub_all.sh, comment out lines for processes you don't need

# Or create custom submission
for proc in ggh vbf; do
    for cat in cat0 cat1; do
        bash sub_${proc}_${cat}.sh &
    done
done
wait
```

### Advanced 5: Two-Dimensional Categories

For 2D categories (e.g., split by mass range and another variable):

```python
# Ensure category names reflect 2D structure
# Example: cat0_mass100to120, cat0_mass120to130

# In replacement map
replacementMap = {
    'myanalysis': {
        ('proc', 'cat0_mass100to120'): ('proc', 'cat_inclusive_mass100to120'),
        # ... define for all 2D bins ...
    }
}
```

## Debugging Techniques

### Debug 1: Inspecting ROOT Files

```python
# inspect_workspace.py
import ROOT

# Open workspace file
f = ROOT.TFile.Open("output_ggh_cat0.root")
ws = f.Get("wsig_13TeV")

# List all objects
ws.Print()

# Get specific objects
pdf = ws.pdf("ggh_125_13TeV_cat0")
norm = ws.function("ggh_125_13TeV_cat0_norm")

# Print PDF structure
pdf.Print("v")

# Plot PDF
mass = ws.var("CMS_hgg_mass")
frame = mass.frame()
pdf.plotOn(frame)

canvas = ROOT.TCanvas()
frame.Draw()
canvas.SaveAs("debug_pdf.pdf")

f.Close()
```

### Debug 2: Checking Fit Convergence

```python
# check_fit_convergence.py
import json
import glob

# Check F-test results
with open("outdir_ext/fTest/json/fTest_results.json") as f:
    ftest = json.load(f)

for cat in ftest:
    print(f"Category {cat}:")
    for proc in ftest[cat]:
        nGauss = ftest[cat][proc]['nGauss']
        chi2 = ftest[cat][proc]['chi2']
        print(f"  {proc}: nGauss={nGauss}, chi2/ndof={chi2:.2f}")
```

### Debug 3: Validating Systematics

```bash
# Check systematic variations in signal workspace
root -l output_ggh_cat0.root

# In ROOT prompt:
wsig_13TeV->Print()

# Look for datasets with "sigma" in name:
# ggh_125_13TeV_cat0_scaleUp
# ggh_125_13TeV_cat0_scaleDown
# ggh_125_13TeV_cat0_smearUp
# ggh_125_13TeV_cat0_smearDown

# Plot systematic variation
CMS_hgg_mass = wsig_13TeV->var("CMS_hgg_mass")
ggh_nominal = wsig_13TeV->data("ggh_125_13TeV_cat0")
ggh_scaleUp = wsig_13TeV->data("ggh_125_13TeV_cat0_scaleUp")

TCanvas c
CMS_hgg_mass->setRange(100, 180)
auto frame = CMS_hgg_mass->frame()
ggh_nominal->plotOn(frame, LineColor(kBlack))
ggh_scaleUp->plotOn(frame, LineColor(kRed))
frame->Draw()
c.SaveAs("debug_syst.pdf")
```

### Debug 4: Datacard Validation

```bash
# Check datacard syntax
text2workspace.py Datacard.txt --dry-run

# Print datacard rates
grep -A 10 "^rate" Datacard.txt

# Check for negative rates or zeros
awk '/^rate/{getline; for(i=1;i<=NF;i++) if($i<=0) print "Zero/negative rate in column",i}' Datacard.txt

# Validate systematic effects
combine -M MultiDimFit Datacard.txt -t -1 --expectSignal=1 --saveFitResult

# Check pulls and constraints
combineTool.py -M Impacts -d Datacard.txt -t -1 --doInitialFit --expectSignal=1
combineTool.py -M Impacts -d Datacard.txt -t -1 --doFits --expectSignal=1
combineTool.py -M Impacts -d Datacard.txt -t -1 -o impacts.json
plotImpacts.py -i impacts.json -o impacts
```

## Best Practices

### Best Practice 1: Workspace Organization

```bash
# Recommended directory structure
/eos/user/username/hgg/
├── inputs/
│   ├── signal_2016/
│   ├── signal_2017/
│   ├── signal_2018/
│   └── data/
│       └── allData.root
├── finalfit/
│   ├── signal_models/
│   │   ├── 2016/
│   │   ├── 2017/
│   │   └── 2018/
│   ├── background_models/
│   ├── datacards/
│   └── fits/
└── plots/
```

### Best Practice 2: Version Control for Configs

```bash
# Keep configs in git
cd $CMSSW_BASE/src/flashggFinalFit
git add Signal/config_myanalysis_*.py
git add Background/config_myanalysis.py
git commit -m "Add configs for myanalysis"

# Tag releases
git tag -a myanalysis_v1 -m "First complete analysis"
git push origin myanalysis_v1
```

### Best Practice 3: Logging and Monitoring

```bash
# Check job status regularly
watch -n 60 'condor_q'

# Monitor output files
watch -n 60 'ls -lh outdir_ext/signalFit/output/*.root | wc -l'

# Collect logs for failed jobs
cd outdir_ext/signalFit/logs
grep -l "ERROR\|FATAL\|Segmentation" *.log > failed_jobs.txt
```

### Best Practice 4: Reproducibility

```python
# Document exact versions in config
signalScriptCfg = {
    # ... other settings ...
    '_metadata': {
        'cmssw_version': 'CMSSW_14_1_0_pre4',
        'combine_commit': '07b56c67ba6e4304b42c3a6cdba710d59c719192',
        'flashggfinalfit_commit': 'abc123...',
        'creation_date': '2024-01-15',
        'author': 'username',
    }
}
```

### Best Practice 5: Resource Optimization

```python
# For large number of categories, group jobs
signalScriptCfg = {
    'batch': 'condor',
    'queue': 'espresso',  # Short jobs
}

# Use --groupSignalFitJobsByCat to reduce number of jobs
# One job per category instead of proc×cat

# For very long jobs, use longer queue
signalScriptCfg = {
    'batch': 'condor',
    'queue': 'longlunch',  # Longer jobs
}
```

## Common Pitfalls

### Pitfall 1: Incorrect Input Path

```bash
# ❌ Wrong: Relative path
'inputWSDir': '../workspaces/signal'

# ✅ Correct: Absolute path
'inputWSDir': '/eos/user/j/jlangfor/workspaces/signal'

# ❌ Wrong: Missing trailing slash sometimes matters
'inputWSDir': '/eos/user/j/jlangfor/workspaces/signal/'

# ✅ Correct: No trailing slash
'inputWSDir': '/eos/user/j/jlangfor/workspaces/signal'
```

### Pitfall 2: Forgetting to Compile Background

```bash
# ❌ Wrong: Running without compiling
cd Background
python3 RunBackgroundScripts.py --inputConfig config.py --mode fTestParallel
# Error: ./bin/fTest: No such file or directory

# ✅ Correct: Compile first
cd Background
make clean && make
python3 RunBackgroundScripts.py --inputConfig config.py --mode fTestParallel
```

### Pitfall 3: Mismatched Category Names

```python
# ❌ Wrong: Inconsistent naming
# Signal uses: cat0, cat1, cat2
# Background uses: category0, category1, category2
# Result: Cannot match signal and background in datacard

# ✅ Correct: Consistent naming everywhere
# Ensure category names match in:
# - HiggsDNA output
# - Signal modeling
# - Background modeling
# - Datacard creation
```

### Pitfall 4: Missing Systematic Datasets

```bash
# ❌ Wrong: Enabling systematics without running calcPhotonSyst
python3 RunSignalScripts.py --mode signalFit
# Error: Cannot find systematic datasets

# ✅ Correct: Either run calcPhotonSyst first, or skip systematics
python3 RunSignalScripts.py --mode calcPhotonSyst
python3 RunSignalScripts.py --mode signalFit

# Or skip systematics explicitly
python3 RunSignalScripts.py --mode signalFit --modeOpts "--skipSystematics"
```

### Pitfall 5: Insufficient Batch Resources

```bash
# ❌ Wrong: Default resources might be too small
signalScriptCfg = {
    'batch': 'condor',
    'queue': 'espresso',  # Only 20 minutes!
}
# Job gets killed for exceeding time

# ✅ Correct: Choose appropriate queue
signalScriptCfg = {
    'batch': 'condor',
    'queue': 'microcentury',  # 1 hour
    # or 'longlunch' (2 hours), 'workday' (8 hours)
}
```

### Pitfall 6: Not Checking Job Completion

```bash
# ❌ Wrong: Proceeding without verifying
python3 RunSignalScripts.py --mode signalFit
# Immediately run packaging without waiting
python3 RunPackager.py ...
# Error: Missing input files

# ✅ Correct: Check job completion
python3 RunSignalScripts.py --mode signalFit

# Wait and check
condor_q  # Check if jobs are done
ls -l outdir_ext/signalFit/output/*.root  # Check output files
grep -i error outdir_ext/signalFit/logs/*.log  # Check for errors

# Then proceed
python3 RunPackager.py ...
```

### Pitfall 7: Wrong Year in Merged Analysis

```python
# ❌ Wrong: Using 'combined' for signal fits
signalScriptCfg = {
    'year': 'combined',  # Don't do this for individual years!
}

# ✅ Correct: Use specific year for fits
signalScriptCfg = {
    'year': '2022preEE',  # Or '2016', '2017', '2018', etc.
}
# Merge at packaging stage if needed
```

## Performance Tips

1. **Parallelize Independent Steps**: Run signal and background modeling simultaneously
2. **Use Fast Queue for Testing**: Use 'espresso' or 'local' for quick tests
3. **Group Jobs Smartly**: Use `--groupSignalFitJobsByCat` to reduce overhead
4. **Cache F-test Results**: Don't rerun F-test unless input changes
5. **Use `--printOnly` First**: Always check job scripts before submission

## Quick Reference Card

```bash
# Signal Modeling Quickstart
cd Signal
python3 RunSignalScripts.py --inputConfig config.py --mode fTest
python3 RunSignalScripts.py --inputConfig config.py --mode calcPhotonSyst
python3 RunSignalScripts.py --inputConfig config.py --mode signalFit

# Background Modeling Quickstart
cd Background
make
python3 RunBackgroundScripts.py --inputConfig config.py --mode fTestParallel

# Packaging
python3 RunPackager.py --cats auto --inputWSDir /path --exts ext1,ext2

# Datacard
cd Datacard
python3 makeDatacard.py --years 2022preEE

# Combine
cd Combine
python3 RunFits.py --inputConfig inputs.json --mode all
```

This guide should help you avoid common mistakes and follow best practices when using flashggFinalFit!
