# flashggFinalFit Quick Start Guide

## Prerequisites

Before starting, ensure you have:
- Access to LXPLUS or a machine with CMSSW installed
- Access to input workspaces from HiggsDNA
- Familiarity with ROOT and RooFit basics
- Understanding of CMS Higgs analysis workflow

## 5-Minute Quick Start

### Step 1: Environment Setup (First Time Only)

```bash
# On LXPLUS or similar
export SCRAM_ARCH=el9_amd64_gcc12
cmsrel CMSSW_14_1_0_pre4
cd CMSSW_14_1_0_pre4/src
cmsenv

# Install Combine
COMBINE_TAG=07b56c67ba6e4304b42c3a6cdba710d59c719192
git clone https://github.com/cms-analysis/HiggsAnalysis-CombinedLimit.git HiggsAnalysis/CombinedLimit
cd HiggsAnalysis/CombinedLimit && git fetch origin ${COMBINE_TAG} && git checkout ${COMBINE_TAG}

# Install CombineHarvester
cd ${CMSSW_BASE}/src
COMBINEHARVESTER_TAG=94017ba5a3a657f7b88669b1a525b19d34ea41a2
bash <(curl -s https://raw.githubusercontent.com/cms-analysis/CombineHarvester/${COMBINEHARVESTER_TAG}/CombineTools/scripts/sparse-checkout-https.sh)
cd CombineHarvester && git fetch origin ${COMBINEHARVESTER_TAG} && git checkout ${COMBINEHARVESTER_TAG}

# Compile
cd ${CMSSW_BASE}/src
cmsenv
scram b clean
scram b -j 8

# Install flashggFinalFit
git clone -b higgsdnafinalfit https://github.com/cms-analysis/flashggFinalFit.git
cd flashggFinalFit/
source setup.sh
```

### Step 2: Run Tutorial Example (2022 preEE)

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Signal

# Use existing tutorial config
# This will run with tutorial data from /eos
python3 RunSignalScripts.py \
    --inputConfig config_tutorial_2022preEE.py \
    --mode fTest \
    --printOnly  # Check the jobs first

# If everything looks good, submit
python3 RunSignalScripts.py \
    --inputConfig config_tutorial_2022preEE.py \
    --mode fTest
```

### Step 3: Check Output

```bash
# Wait for jobs to complete (check with: condor_q or bjobs)

# Check output
ls -l outdir_tutorial_2022preEE/fTest/json/
cat outdir_tutorial_2022preEE/fTest/json/fTest_results.json

# Check logs for errors
grep -i error outdir_tutorial_2022preEE/fTest/logs/*.log
```

### Step 4: Continue the Workflow

```bash
# Calculate photon systematics
python3 RunSignalScripts.py \
    --inputConfig config_tutorial_2022preEE.py \
    --mode calcPhotonSyst

# Get diagonal processes
python3 RunSignalScripts.py \
    --inputConfig config_tutorial_2022preEE.py \
    --mode getDiagProc

# Run signal fit
python3 RunSignalScripts.py \
    --inputConfig config_tutorial_2022preEE.py \
    --mode signalFit \
    --groupSignalFitJobsByCat
```

That's it! You've run your first signal modeling with flashggFinalFit.

## 30-Minute Complete Tutorial

### Your Own Analysis Setup

#### 1. Prepare Your Configuration

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Signal

# Copy tutorial config as template
cp config_tutorial_2022preEE.py config_myanalysis_2022preEE.py

# Edit with your settings
vim config_myanalysis_2022preEE.py
```

Edit the configuration:

```python
_year = '2022preEE'
_analysis = 'myanalysis'
_input_ws_dir = '/path/to/your/signal/workspaces'  # ← Change this

signalScriptCfg = {
    'inputWSDir': _input_ws_dir,
    'procs': 'auto',
    'cats': 'auto',
    'ext': 'myanalysis_2022preEE',
    'analysis': 'myanalysis',
    'year': _year,
    'massPoints': '120,125,130',
    'scales': 'Scale',
    'scalesCorr': '',
    'scalesGlobal': '',
    'smears': 'Smearing',
    'batch': 'condor',  # or 'local' for testing
    'queue': 'espresso',
}
```

#### 2. Setup Replacement Map and XSBR Map

```bash
cd $CMSSW_BASE/src/flashggFinalFit/tools
```

Edit `replacementMap.py`:

```python
replacementMap = {
    # ... existing maps ...
    'myanalysis': {
        # Define replacements for low-statistics (proc, cat) pairs
        # Format: (process, category) -> (replacement_process, replacement_category)
        ('ggh', 'cat0'): ('ggh', 'cat0'),  # No replacement needed
        ('vbf', 'cat1'): ('ggh', 'cat1'),  # Use ggh shape if vbf has low stats
        # Add more as needed
    }
}
```

Edit `XSBRMap.py`:

```python
XSBRMap = {
    # ... existing maps ...
    'myanalysis': {
        # Define how each process is normalized
        # Use LHCHXSWG cross sections
        'ggh': {
            'factor': 'ggH',  # Use standard ggH cross section
            'BR': 'gamgam',   # Use H→γγ branching ratio
        },
        'vbf': {
            'factor': 'qqH',
            'BR': 'gamgam',
        },
        # Add all your signal processes
    }
}
```

#### 3. Test Locally First

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Signal

# Run F-test locally for one category
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode fTest \
    --printOnly

# Check the job script
cat outdir_myanalysis_2022preEE/fTest/jobs/sub_cat0.sh

# Run locally
cd outdir_myanalysis_2022preEE/fTest/jobs
bash sub_cat0.sh

# Check output
ls -l ../json/
cat ../logs/log_cat0.log
```

#### 4. Run Full Signal Workflow

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Signal

# Submit all F-test jobs
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode fTest

# Wait for completion, then continue
python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode calcPhotonSyst

python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode getDiagProc

python3 RunSignalScripts.py \
    --inputConfig config_myanalysis_2022preEE.py \
    --mode signalFit \
    --groupSignalFitJobsByCat \
    --modeOpts "--doPlots --useDiagonalProcForSyst --skipVertexScenarioSplit"
```

#### 5. Background Modeling

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Background

# First time: compile
make clean && make

# Create config
cp config_tutorial.py config_myanalysis.py
# Edit: set inputWSDir to your allData.root location

# Run background F-test
python3 RunBackgroundScripts.py \
    --inputConfig config_myanalysis.py \
    --mode fTestParallel

# Wait for completion
ls -l outdir_myanalysis/CMS-HGG_multipdf_*.root

# Copy to Combine directory
cp outdir_myanalysis/CMS-HGG_multipdf_*.root ../Combine/
```

#### 6. Package Signal Models

```bash
cd $CMSSW_BASE/src/flashggFinalFit

# Package signal models
python3 Signal/RunPackager.py \
    --cats auto \
    --inputWSDir /path/to/your/signal/workspaces \
    --exts myanalysis_2022preEE \
    --outputExt myanalysis_2022preEE_packaged

# Output will be in Signal/outdir_packaged/
ls -l Signal/outdir_packaged/CMS-HGG_sigfit_*.root
```

#### 7. Create Datacard

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Datacard

# Run datacard creation
python3 makeDatacard.py \
    --years 2022preEE \
    --signalWSDir ../Signal/outdir_packaged \
    --backgroundWSDir ../Combine \
    --cats cat0,cat1,cat2 \
    --procs all \
    --ext myanalysis_2022preEE

# Check output
ls -l Datacard_myanalysis_2022preEE_*.txt
```

#### 8. Run Combine Fits

```bash
cd $CMSSW_BASE/src/flashggFinalFit/Combine

# Create inputs JSON
cat > inputs_myanalysis.json << EOF
{
    "inputWSDir": "../Datacard",
    "ext": "myanalysis_2022preEE",
    "fit_types": ["MaxLikelihoodFit", "AsymptoticLimits", "Significance"]
}
EOF

# Convert to workspace
python3 RunText2Workspace.py --inputConfig inputs_myanalysis.json

# Run fits
python3 RunFits.py --inputConfig inputs_myanalysis.json --mode all

# Check results
ls -l higgsCombine*.root
```

## Essential Commands Cheat Sheet

### Environment

```bash
# Setup environment (every session)
cd $CMSSW_BASE/src/flashggFinalFit
cmsenv
source setup.sh
```

### Signal Modeling

```bash
# Test mode (no submission)
python3 RunSignalScripts.py --inputConfig config.py --mode {mode} --printOnly

# Run modes
python3 RunSignalScripts.py --inputConfig config.py --mode fTest
python3 RunSignalScripts.py --inputConfig config.py --mode calcPhotonSyst
python3 RunSignalScripts.py --inputConfig config.py --mode getDiagProc
python3 RunSignalScripts.py --inputConfig config.py --mode signalFit

# With options
python3 RunSignalScripts.py --inputConfig config.py --mode signalFit \
    --modeOpts "--doPlots --useDCB"
```

### Background Modeling

```bash
# Compile
cd Background && make

# Run
python3 RunBackgroundScripts.py --inputConfig config.py --mode fTestParallel
```

### Packaging

```bash
# Single year
python3 RunPackager.py --cats auto --inputWSDir /path --exts ext1

# Multiple years
python3 RunPackager.py --cats cat0,cat1 --exts ext1,ext2,ext3 --mergeYears
```

### Monitoring

```bash
# Check batch jobs
condor_q              # HTCondor
bjobs                 # LSF
qstat                 # SGE

# Check output files
ls -l outdir_*/mode/output/

# Check logs
grep -i error outdir_*/mode/logs/*.log
tail -f outdir_*/mode/logs/log_cat0.log  # Follow log in real-time
```

## Common Workflows

### Workflow 1: Single Year Analysis

```bash
# 1. Signal
cd Signal
python3 RunSignalScripts.py --inputConfig config_2022preEE.py --mode fTest
python3 RunSignalScripts.py --inputConfig config_2022preEE.py --mode calcPhotonSyst
python3 RunSignalScripts.py --inputConfig config_2022preEE.py --mode signalFit

# 2. Background
cd ../Background
python3 RunBackgroundScripts.py --inputConfig config.py --mode fTestParallel

# 3. Datacard
cd ../Datacard
python3 makeDatacard.py --years 2022preEE

# 4. Fits
cd ../Combine
python3 RunFits.py --inputConfig inputs.json --mode all
```

### Workflow 2: Multi-Year Combined Analysis

```bash
# For each year: 2016, 2017, 2018
for year in 2016 2017 2018; do
    # 1. Signal
    cd Signal
    python3 RunSignalScripts.py --inputConfig config_${year}.py --mode fTest
    python3 RunSignalScripts.py --inputConfig config_${year}.py --mode calcPhotonSyst
    python3 RunSignalScripts.py --inputConfig config_${year}.py --mode signalFit
    
    # 2. Background
    cd ../Background
    python3 RunBackgroundScripts.py --inputConfig config_${year}.py --mode fTestParallel
    
    cd ..
done

# 3. Merge signal models
python3 Signal/RunPackager.py \
    --cats cat0,cat1,cat2 \
    --exts myanalysis_2016,myanalysis_2017,myanalysis_2018 \
    --mergeYears

# 4. Combined datacard
cd Datacard
python3 makeDatacard.py --years 2016,2017,2018

# 5. Combined fits
cd ../Combine
python3 RunFits.py --inputConfig inputs_combined.json --mode all
```

### Workflow 3: Quick Iteration (Testing Changes)

```bash
# Run locally without batch submission
cd Signal

# Test signal fit for one category
python3 RunSignalScripts.py \
    --inputConfig config.py \
    --mode signalFit \
    --printOnly

# Run the job locally
cd outdir_ext/signalFit/jobs
bash sub_cat0.sh

# Check result
root -l ../output/output_ggh_cat0.root
# In ROOT: wsig_13TeV->Print()
```

## Tips for Beginners

1. **Always use `--printOnly` first** to check what jobs will be created
2. **Start with local execution** before submitting to batch
3. **Check logs frequently** for errors
4. **Use tutorial configs as templates** - they are tested and working
5. **Keep configs in version control** to track your changes
6. **Document your analysis** - future you will thank present you

## Next Steps

After completing this quick start:

1. Read [INVESTIGATION.md](INVESTIGATION.md) for detailed package overview
2. Study [ARCHITECTURE.md](ARCHITECTURE.md) for system architecture
3. Review [CODE_EXAMPLES.md](CODE_EXAMPLES.md) for advanced usage
4. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) when issues arise
5. Follow the [official tutorial](https://gitlab.cern.ch/jspah/higgsdna_finalfits_tutorial_24)

## Getting Help

- **Documentation**: Check README files in each subdirectory
- **Tutorial**: https://gitlab.cern.ch/jspah/higgsdna_finalfits_tutorial_24
- **Mattermost**: CMS Hgg channel
- **GitHub Issues**: flashggFinalFit repository
- **Hypernews**: hn-cms-higgs@cern.ch

## Summary

You should now be able to:
- ✅ Set up the flashggFinalFit environment
- ✅ Run the tutorial example
- ✅ Configure your own analysis
- ✅ Run signal and background modeling
- ✅ Create datacards and run fits
- ✅ Monitor and debug jobs

Happy analyzing! 🎉
