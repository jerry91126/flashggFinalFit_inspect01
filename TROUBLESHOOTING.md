# flashggFinalFit Troubleshooting Guide

## Table of Contents
1. [Installation Issues](#installation-issues)
2. [Signal Modeling Issues](#signal-modeling-issues)
3. [Background Modeling Issues](#background-modeling-issues)
4. [Datacard Issues](#datacard-issues)
5. [Combine Fit Issues](#combine-fit-issues)
6. [Job Submission Issues](#job-submission-issues)
7. [ROOT and RooFit Issues](#root-and-roofit-issues)
8. [Performance Issues](#performance-issues)
9. [Data Issues](#data-issues)
10. [FAQ](#faq)

## Installation Issues

### Issue: CMSSW Compilation Fails

**Symptoms:**
```
>> Compiling ...
fatal error: ...
```

**Solutions:**

1. Check CMSSW version:
```bash
echo $CMSSW_VERSION
# Should be: CMSSW_14_1_0_pre4 (or compatible version)
```

2. Check architecture:
```bash
echo $SCRAM_ARCH
# Should be: el9_amd64_gcc12
```

3. Clean and rebuild:
```bash
cd $CMSSW_BASE/src
scram b clean
scram b -j 8
```

4. Check for conflicting packages:
```bash
cd $CMSSW_BASE/src
ls -la
# Remove any conflicting packages
```

### Issue: Cannot Find flashggFinalFit

**Symptoms:**
```
ImportError: No module named 'commonTools'
```

**Solutions:**

1. Check PYTHONPATH:
```bash
echo $PYTHONPATH | tr ':' '\n' | grep flashggFinalFit
# Should see: .../flashggFinalFit/tools
```

2. Re-source setup:
```bash
cd $CMSSW_BASE/src/flashggFinalFit
source setup.sh
```

3. Check file exists:
```bash
ls -l $CMSSW_BASE/src/flashggFinalFit/tools/commonTools.py
```

### Issue: Combine Not Found

**Symptoms:**
```
bash: combine: command not found
```

**Solutions:**

1. Check combine installation:
```bash
ls -l $CMSSW_BASE/src/HiggsAnalysis/CombinedLimit
```

2. Rebuild combine:
```bash
cd $CMSSW_BASE/src/HiggsAnalysis/CombinedLimit
git status
cd $CMSSW_BASE/src
scram b -j 8
```

3. Source CMSSW environment:
```bash
cmsenv
which combine  # Should show path in CMSSW
```

## Signal Modeling Issues

### Issue: "No such directory" Error

**Symptoms:**
```
--> [ERROR] No such directory (/path/to/workspaces)
```

**Solutions:**

1. Check path exists:
```bash
ls -ld /path/to/workspaces
```

2. Use absolute path in config:
```python
'inputWSDir': '/eos/user/j/jlangfor/...',  # Not '../workspaces'
```

3. Check EOS mount (if using EOS):
```bash
eos ls /eos/user/j/jlangfor/workspaces
```

### Issue: "Cannot Extract Production Mode"

**Symptoms:**
```
--> [ERROR]: cannot extract production mode from input file name
```

**Solutions:**

1. Check file naming convention:
```bash
# Files should be named like:
# output_GluGluHToGG_M125_pythia8.root
# output_VBFHToGG_M125_pythia8.root
ls -l inputWSDir/*.root
```

2. Add custom process to `commonTools.py`:
```python
# In tools/commonTools.py, function signalFromFileName()
elif "MyProcess" in _fileName:
    p = "myprocess"
    d = None
```

### Issue: F-test Returns NaN or Inf

**Symptoms:**
```
chi2: nan
nGaussians: -1
```

**Solutions:**

1. Check input data quality:
```bash
root -l output_process_M125.root
# Check: wsig->Print()
# Look for datasets with reasonable statistics
```

2. Increase statistics threshold:
```bash
python3 RunSignalScripts.py \
    --modeOpts "--nProcsToFTest 3"  # Only test top 3 processes
```

3. Skip problematic categories:
```python
# In config
'cats': 'cat0,cat1,cat2'  # Explicitly list working categories
```

### Issue: Signal Fit Fails to Converge

**Symptoms:**
```
Minimization failed
Status: 1
```

**Solutions:**

1. Try different minimizer:
```bash
python3 RunSignalScripts.py --mode signalFit \
    --modeOpts "--minimizerMethod Nelder-Mead"
```

2. Increase tolerance:
```bash
python3 RunSignalScripts.py --mode signalFit \
    --modeOpts "--minimizerTolerance 1e-8"
```

3. Use DCB instead of multiple Gaussians:
```bash
python3 RunSignalScripts.py --mode signalFit \
    --modeOpts "--useDCB"
```

4. Skip vertex splitting:
```bash
python3 RunSignalScripts.py --mode signalFit \
    --modeOpts "--skipVertexScenarioSplit"
```

### Issue: Missing Photon Systematic Datasets

**Symptoms:**
```
KeyError: 'ggh_125_13TeV_cat0_scaleUp'
```

**Solutions:**

1. Check if systematics exist in input:
```bash
root -l output_ggh_M125.root
# In ROOT:
tagsDumper/cms_hgg_13TeV->allData()->Print()
# Look for datasets with "sigma" or "scale"/"smear"
```

2. Skip systematics if not available:
```bash
python3 RunSignalScripts.py --mode signalFit \
    --modeOpts "--skipSystematics"
```

3. Or don't run calcPhotonSyst:
```bash
# Skip the calcPhotonSyst step entirely
python3 RunSignalScripts.py --mode signalFit --modeOpts "--skipSystematics"
```

### Issue: Replacement Map Error

**Symptoms:**
```
KeyError: 'myanalysis' in replacementMap
```

**Solutions:**

1. Add your analysis to `tools/replacementMap.py`:
```python
replacementMap = {
    # ... existing ...
    'myanalysis': {
        ('ggh', 'cat0'): ('ggh', 'cat0'),
        # Add all (proc, cat) pairs
    }
}
```

2. Or use existing map:
```python
# In config
'analysis': 'STXS',  # Use existing STXS map
```

### Issue: XSBR Map Error

**Symptoms:**
```
KeyError: 'myanalysis' in XSBRMap
```

**Solutions:**

1. Add your analysis to `tools/XSBRMap.py`:
```python
XSBRMap = {
    # ... existing ...
    'myanalysis': {
        'ggh': {'factor': 'ggH', 'BR': 'gamgam'},
        'vbf': {'factor': 'qqH', 'BR': 'gamgam'},
        # Add all processes
    }
}
```

## Background Modeling Issues

### Issue: Compilation Fails

**Symptoms:**
```
make: *** [bin/fTest] Error 1
```

**Solutions:**

1. Clean and rebuild:
```bash
cd Background
make clean
make
```

2. Check ROOT libraries:
```bash
root-config --libs
# Should include: -lRooFit -lRooFitCore
```

3. Check compiler:
```bash
g++ --version
# Should match CMSSW's gcc version
```

### Issue: fTest Hangs or Runs Forever

**Symptoms:**
- Job runs for hours without completing
- No output in log file

**Solutions:**

1. Check if data file is accessible:
```bash
root -l /path/to/allData.root
# Try to open and browse
```

2. Reduce function orders to test:
```bash
# Edit Background/scripts/fTestParallel.py
# Reduce the orders being tested
```

3. Run locally to debug:
```bash
cd Background/outdir_ext/fTestParallel/jobs
bash sub_cat0.sh  # Run one category
```

### Issue: All Functions Fail F-test

**Symptoms:**
```
Error: No valid background functions found
```

**Solutions:**

1. Check data quality:
```bash
root -l allData.root
# Check: tagsDumper/cms_hgg_13TeV->Print()
# Look at: Data_13TeV_cat0
```

2. Lower goodness-of-fit threshold (expert use):
```bash
# Edit fTest.cc, modify GoF threshold
```

3. Check for outliers in data:
```bash
# Plot data first
root -l allData.root
Data_13TeV_cat0->plotOn(frame)
```

### Issue: RooMultiPdf Creation Fails

**Symptoms:**
```
Error creating RooMultiPdf
```

**Solutions:**

1. Check if at least one function passes:
```bash
# Check log file
cat Background/outdir_ext/fTestParallel/logs/log_cat0.log
grep "PASS" Background/outdir_ext/fTestParallel/logs/log_cat0.log
```

2. Manually inspect output:
```bash
root -l Background/outdir_ext/CMS-HGG_multipdf_cat0.root
multipdf->Print()
```

## Datacard Issues

### Issue: Cannot Find Signal/Background Workspaces

**Symptoms:**
```
FileNotFoundError: Signal workspace not found
```

**Solutions:**

1. Check paths:
```bash
ls -l Signal/outdir_packaged/CMS-HGG_sigfit_*.root
ls -l Combine/CMS-HGG_multipdf_*.root
```

2. Ensure packaging completed:
```bash
python3 Signal/RunPackager.py --cats auto --inputWSDir /path --exts ext1
```

3. Check file permissions:
```bash
ls -l Combine/*.root
# Should be readable
```

### Issue: Mismatched Categories

**Symptoms:**
```
Error: Category cat0 in signal but not in background
```

**Solutions:**

1. Check category names match:
```bash
# Signal categories
root -l Signal/outdir_packaged/CMS-HGG_sigfit_cat0.root

# Background categories
root -l Combine/CMS-HGG_multipdf_cat0.root

# Data categories
root -l /path/to/allData.root
```

2. Use consistent naming:
```python
# Ensure all use same category names: cat0, cat1, etc.
```

### Issue: Negative or Zero Rates

**Symptoms:**
```
Warning: Process X in category Y has rate 0.000
```

**Solutions:**

1. Check signal normalization:
```bash
# Verify XSBR map is correct
cat tools/XSBRMap.py | grep -A 5 "myanalysis"
```

2. Check if process exists in category:
```bash
root -l Signal/outdir_packaged/CMS-HGG_sigfit_cat0.root
# Check: proc_125_13TeV_cat0_norm
```

3. Use replacement for low-stats processes:
```python
# In tools/replacementMap.py
('low_stats_proc', 'cat0'): ('high_stats_proc', 'cat0')
```

### Issue: Systematic Not Found

**Symptoms:**
```
KeyError: 'CMS_hgg_scale'
```

**Solutions:**

1. Check if systematic is in workspace:
```bash
root -l Signal/outdir_packaged/CMS-HGG_sigfit_cat0.root
wsig_13TeV->allData()->Print()
# Look for: proc_125_13TeV_cat0_scaleUp/Down
```

2. Skip missing systematics:
```python
# In makeDatacard.py or systematics.py
# Comment out missing systematics
```

3. Rebuild signal models with systematics:
```bash
python3 Signal/RunSignalScripts.py --mode calcPhotonSyst
python3 Signal/RunSignalScripts.py --mode signalFit
```

## Combine Fit Issues

### Issue: "Unable to Open Datacard"

**Symptoms:**
```
Error: Unable to open datacard Datacard.txt
```

**Solutions:**

1. Check file exists:
```bash
ls -l Datacard/Datacard.txt
```

2. Check datacard syntax:
```bash
text2workspace.py Datacard.txt --dry-run
```

3. Validate datacard manually:
```bash
# Open in text editor, check for:
# - All bins defined
# - All processes listed
# - Rates present
# - Systematics formatted correctly
```

### Issue: Combine Crashes or Segfaults

**Symptoms:**
```
Segmentation fault (core dumped)
```

**Solutions:**

1. Check workspace integrity:
```bash
root -l workspace.root
w->Print()
# Look for any errors
```

2. Recreate workspace:
```bash
text2workspace.py Datacard.txt -o workspace.root --X-allow-no-signal
```

3. Try different fit strategy:
```bash
combine -M MaxLikelihoodFit Datacard.txt --robustFit=1
```

4. Increase verbosity:
```bash
combine -M MaxLikelihoodFit Datacard.txt -v 3
```

### Issue: Fit Does Not Converge

**Symptoms:**
```
WARNING: Fit failed with status 1
```

**Solutions:**

1. Check initial parameter values:
```bash
combine -M MaxLikelihoodFit Datacard.txt --cminDefaultMinimizerType Minuit2
```

2. Use robust fit:
```bash
combine -M MaxLikelihoodFit Datacard.txt --robustFit=1
```

3. Freeze problematic parameters:
```bash
combine -M MaxLikelihoodFit Datacard.txt --freezeParameters bkg_norm
```

4. Check for correlations:
```bash
combine -M FitDiagnostics Datacard.txt --saveShapes --saveWithUncertainties
```

### Issue: Limits Look Wrong

**Symptoms:**
- Limits are 0 or inf
- Expected and observed very different
- Limits don't change with signal strength

**Solutions:**

1. Check signal rates are reasonable:
```bash
grep "^rate" Datacard.txt
```

2. Run expected-only:
```bash
combine -M AsymptoticLimits Datacard.txt -t -1 --expectSignal=1
```

3. Check background model:
```bash
combine -M FitDiagnostics Datacard.txt --saveShapes
# Inspect shapes in fitDiagnostics.root
```

4. Validate with counting experiment:
```bash
# Simplify datacard to counting experiment to debug
```

## Job Submission Issues

### Issue: Jobs Not Submitting

**Symptoms:**
```
No jobs in queue
```

**Solutions:**

1. Check batch system:
```bash
condor_q  # For HTCondor
bjobs     # For LSF
qstat     # For SGE
```

2. Check submission script:
```bash
cat outdir_ext/mode/jobs/sub_all.sh
# Verify commands are correct
```

3. Check permissions:
```bash
ls -l outdir_ext/mode/jobs/*.sh
# Should be executable
chmod +x outdir_ext/mode/jobs/*.sh
```

4. Check queue exists:
```bash
condor_q -analyze  # HTCondor
bqueues            # LSF
```

### Issue: Jobs Failing Immediately

**Symptoms:**
```
Jobs go to held/error state immediately
```

**Solutions:**

1. Check log files:
```bash
cat outdir_ext/mode/logs/log_*.log
cat outdir_ext/mode/logs/log_*.err  # If exists
```

2. Check memory requirements:
```bash
# Edit job submission to increase memory
# In RunSignalScripts.py, modify job creation
```

3. Test job locally:
```bash
cd outdir_ext/mode/jobs
bash sub_cat0.sh  # Run one job locally
```

### Issue: Jobs Timing Out

**Symptoms:**
```
Job exceeded wall time limit
```

**Solutions:**

1. Use longer queue:
```python
# In config
'queue': 'longlunch',  # Instead of 'espresso'
```

2. Split jobs finer:
```bash
# Use default (one job per proc×cat) instead of grouping
python3 RunSignalScripts.py --mode signalFit
# Remove --groupSignalFitJobsByCat
```

3. Optimize code:
```bash
# Reduce mass points
'massPoints': '125',  # Only fit at 125 GeV
```

## ROOT and RooFit Issues

### Issue: RooFit Warnings/Errors

**Symptoms:**
```
[#1] INFO:Minization -- ...
[#2] WARNING:Eval -- ...
```

**Solutions:**

1. Most INFO messages can be ignored
2. Check WARNING messages:
   - "parameter at limit" → Check parameter ranges
   - "negative PDF" → Check model definition
   - "NaN in likelihood" → Check input data

3. Suppress non-critical messages:
```python
import ROOT
ROOT.RooMsgService.instance().setGlobalKillBelow(ROOT.RooFit.ERROR)
```

### Issue: "Negative PDF" Error

**Symptoms:**
```
ERROR: Negative PDF value
```

**Solutions:**

1. Check parameter ranges:
```bash
root -l workspace.root
w->var("param")->Print()
# Verify ranges are physical
```

2. Check input data:
```bash
# Plot data to look for outliers
root -l file.root
data->plotOn(frame)
```

### Issue: Memory Leaks

**Symptoms:**
- Jobs killed for exceeding memory
- Memory usage grows over time

**Solutions:**

1. Delete ROOT objects:
```python
ws.Delete()
f.Close()
```

2. Use memory monitoring:
```bash
/usr/bin/time -v python3 script.py
```

3. Process in batches:
```python
# Process categories in batches instead of all at once
```

## Performance Issues

### Issue: Jobs Run Very Slowly

**Solutions:**

1. Check I/O:
```bash
# Use local disk instead of network storage for temp files
export TMPDIR=/tmp
```

2. Reduce number of bins:
```bash
python3 RunSignalScripts.py --mode signalFit \
    --modeOpts "--nBins 40"  # Default is 80
```

3. Parallelize more:
```bash
# Use more jobs, don't group
python3 RunSignalScripts.py --mode signalFit
# Remove --groupSignalFitJobsByCat
```

### Issue: Disk Space Issues

**Symptoms:**
```
No space left on device
```

**Solutions:**

1. Clean up old outputs:
```bash
rm -rf outdir_old_*/
```

2. Don't save plots:
```bash
# Remove --doPlots from modeOpts
```

3. Use EOS for outputs:
```bash
# Set output directory to EOS path
```

## Data Issues

### Issue: Wrong Data Loaded

**Symptoms:**
- Unexpected number of events
- Wrong categories

**Solutions:**

1. Verify data file:
```bash
root -l allData.root
tagsDumper/cms_hgg_13TeV->allData()->Print()
```

2. Check year/era:
```python
# In config, verify year matches data
'year': '2022preEE',  # Must match data taking period
```

### Issue: Blinded Data

**Symptoms:**
- Cannot see signal region

**Solutions:**

1. Use Asimov dataset:
```bash
combine -M FitDiagnostics Datacard.txt -t -1 --expectSignal=1
```

2. Use toy data:
```bash
combine -M GenerateOnly Datacard.txt -t 1 --saveToys
```

## FAQ

### Q: How do I know if my fit converged?

A: Check for:
- Status = 0 in fit output
- All parameters within reasonable ranges
- Correlation matrix looks reasonable
- Postfit pulls look reasonable

### Q: My background model doesn't fit the data well. What should I do?

A: 
1. Check if this is in signal region (shouldn't fit perfectly, that's the signal!)
2. Run background-only fit to check
3. Consider if background model needs more complexity
4. Check for data quality issues

### Q: Can I use a single mass point instead of multiple?

A: Yes! Set `'massPoints': '125'` and the code will fit at 125 GeV only with no interpolation.

### Q: Should I split by vertex scenario?

A: 
- For ggH 0J: Yes (significant WV contribution)
- For other processes: Usually no (`--skipVertexScenarioSplit`)

### Q: How do I add a new systematic uncertainty?

A:
1. Add to workspace with proper naming convention
2. Add to `Datacard/systematics.py`
3. Ensure consistent across all categories/processes

### Q: My jobs are stuck in queue. How long should I wait?

A:
- espresso: ~20 minutes
- microcentury: ~1 hour
- longlunch: ~2 hours
- If longer, check job status and logs

### Q: Can I run everything locally without batch?

A: Yes! Set `'batch': 'local'` in config. But it will be much slower.

### Q: How do I debug a specific category?

A:
1. Go to `outdir_ext/mode/jobs/`
2. Find script for that category
3. Run it locally: `bash sub_cat0.sh`
4. Check logs and output

### Q: What if I don't have photon systematics?

A: Use `--skipSystematics` flag in signal fit mode.

### Q: How do I merge results from different data-taking periods?

A: Run signal/background separately for each period, then use `RunPackager.py --mergeYears`.

## Getting More Help

If you're still stuck:

1. **Check logs**: Always start with the log files
2. **Run locally**: Test problematic jobs locally
3. **Simplify**: Remove complexity (fewer categories, processes, systematics)
4. **Ask for help**:
   - CMS Hgg Mattermost channel
   - Hypernews: hn-cms-higgs@cern.ch
   - GitHub issues: flashggFinalFit repository

## Appendix: Useful Commands

### Debugging Commands

```bash
# Check workspace contents
root -l file.root
w->Print()
w->allVars()->Print("v")
w->allFunctions()->Print("v")
w->allPdfs()->Print("v")

# Check datacard
text2workspace.py Datacard.txt --dry-run

# Test fit
combine -M MaxLikelihoodFit Datacard.txt -v 3

# Check file integrity
root -l file.root
# If it opens without errors, file is OK

# Monitor job
tail -f outdir_ext/mode/logs/log_*.log

# Check batch system
condor_q -analyze  # HTCondor detailed status
bjobs -l JOB_ID    # LSF detailed status
```

### Recovery Commands

```bash
# Resubmit failed jobs
cd outdir_ext/mode/jobs
for job in $(grep -l "ERROR" ../logs/*.log); do
    jobname=$(basename $job .log | sed 's/log_/sub_/')
    bash ${jobname}.sh &
done

# Clean and restart
rm -rf outdir_ext/
python3 RunSignalScripts.py --inputConfig config.py --mode {mode}

# Reset CMSSW
cd $CMSSW_BASE/src
cmsenv
scram b clean
scram b -j 8
```

Good luck with your analysis!
