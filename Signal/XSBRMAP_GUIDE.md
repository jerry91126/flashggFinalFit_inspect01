# XSBRMap Guide: Application in Signal Modeling

## Table of Contents
1. [Overview](#overview)
2. [Purpose and Role](#purpose-and-role)
3. [Structure of XSBRMap](#structure-of-xsbrmap)
4. [How XSBRMap is Used](#how-xsbrmap-is-used)
5. [Configuration Examples](#configuration-examples)
6. [Technical Implementation](#technical-implementation)
7. [Best Practices](#best-practices)

## Overview

The **XSBRMap** (Cross Section × Branching Ratio Map) is a critical component of the signal modeling pipeline in the Final Fits package. It defines how signal processes are normalized during the fitting procedure, ensuring that each process has the correct theoretical cross section and branching ratio applied.

## Purpose and Role

### What is XSBRMap?

XSBRMap is a configuration dictionary defined in `Signal/tools/XSBRMap.py` that maps each signal process in your analysis to:
- A **production mode** (e.g., ggH, qqH, WH, ttH)
- A **cross section normalization factor**
- A **branching ratio mode** (typically H→γγ for Hgg analyses)

### Why is it Needed?

The final signal model normalization follows this equation:

```
N(i,j) = (σ × BR)(i) × (ε × A)(i,j) × L
```

Where:
- `N(i,j)` = number of signal events for process *i* in category *j*
- `(σ × BR)(i)` = cross section × branching ratio for process *i* → **provided by XSBRMap**
- `(ε × A)(i,j)` = efficiency × acceptance for process *i* in category *j* → calculated from HiggsDNA sum of weights
- `L` = integrated luminosity

XSBRMap provides the theoretically-predicted cross sections and branching ratios, ensuring your signal models are normalized to SM predictions or specific BSM scenarios.

## Structure of XSBRMap

### Basic Structure

The XSBRMap is an OrderedDict with the following hierarchy:

```python
globalXSBRMap = {
    'analysis_name': {
        'decay': {'mode': 'decay_channel', 'factor': optional_factor},
        'PROCESS_NAME': {'mode': 'production_mode', 'factor': optional_factor},
        # ... more processes ...
    }
}
```

### Entry Components

Each process entry has:
- **`mode`**: The production or decay mode name
  - Production modes: 'ggH', 'qqH', 'WH', 'qqZH', 'ggZH', 'ttH', 'bbH', 'tHq', 'tHW'
  - Decay mode: 'hgg' (H→γγ)
  - Special: 'constant' for fixed cross sections
- **`factor`** (optional): A multiplicative factor to apply
  - For STXS bins: the fractional composition of the bin
  - For associated production: branching ratios of W/Z decays
  - For custom processes: arbitrary normalization factors

## How XSBRMap is Used

### During Signal Fitting (`signalFit.py`)

1. **Configuration Loading**:
   ```python
   # The analysis name from config selects which XSBRMap to use
   xsbrMap = globalXSBRMap[opt.analysis]
   ```

2. **XS/BR Initialization**:
   - The `initialiseXSBR()` function loads theoretical cross sections and branching ratios from Combine's SM Higgs data files
   - These values are computed as a function of Higgs mass (MH) from 120-130 GeV

3. **Building Splines** (in `finalModel.py`):
   ```python
   def buildXSBRSplines(self):
       # Extract factor and mode for the process
       fp = self.xsbrMap[self.proc]['factor'] if 'factor' in self.xsbrMap[self.proc] else 1.
       mp = self.xsbrMap[self.proc]['mode']
       
       # Compute XS: factor × base_cross_section(MH)
       xs = fp * self.XSBR[mp]
       
       # Create spline for MH-dependent cross section
       self.Splines['xs'] = ROOT.RooSpline1D(...)
       
       # Similarly for BR (branching ratio)
       fd = self.xsbrMap['decay']['factor'] if 'factor' in self.xsbrMap['decay'] else 1.
       md = self.xsbrMap['decay']['mode']
       br = fd * self.XSBR[md]
       
       self.Splines['br'] = ROOT.RooSpline1D(...)
   ```

4. **Final Normalization**:
   ```python
   # The final normalization combines all components
   final_norm = xs_spline × br_spline × ea_spline × rate_systematics
   ```

### In Datacard Creation

The `Datacard/tools/XSBR.py` module uses the same XSBRMap structure to:
- Extract cross sections and branching ratios for each signal process
- Populate the datacard with correct signal normalizations
- Ensure consistency between signal modeling and statistical interpretation

## Configuration Examples

### Example 1: Tutorial Analysis (Constant Cross Sections)

For a simple tutorial or when Combine doesn't have the cross sections (e.g., 13.6 TeV):

```python
globalXSBRMap['tutorial'] = od()
globalXSBRMap['tutorial']['decay'] = {'mode': 'hgg'}
globalXSBRMap['tutorial']['GG2H'] = {'mode': 'constant', 'factor': 51.96}  # σ_ggH at 13.6 TeV
globalXSBRMap['tutorial']['VBF'] = {'mode': 'constant', 'factor': 4.067}   # σ_VBF at 13.6 TeV
```

**When to use**:
- Cross sections not available in Combine's data files
- Testing with fixed normalizations
- Simplified analyses

### Example 2: Inclusive Production Modes

For standard production modes without sub-binning:

```python
globalXSBRMap['example'] = od()
globalXSBRMap['example']['decay'] = {'mode': 'hgg'}
globalXSBRMap['example']['GG2H'] = {'mode': 'ggH'}
globalXSBRMap['example']['VBF'] = {'mode': 'qqH'}
globalXSBRMap['example']['WH2HQQ'] = {'mode': 'WH', 'factor': BR_W_qq}  # W→qq branching ratio
globalXSBRMap['example']['TTH'] = {'mode': 'ttH'}
```

**When to use**:
- Standard inclusive H→γγ analysis
- No STXS or differential binning
- Need full SM cross sections from Combine

### Example 3: STXS Analysis (Differential Bins)

For STXS (Simplified Template Cross Sections) with multiple bins:

```python
globalXSBRMap['STXS'] = od()
globalXSBRMap['STXS']['decay'] = {'mode': 'hgg'}

# ggH STXS bins: factor represents fraction of total ggH in that bin
globalXSBRMap['STXS']['GG2H_0J_PTH_0_10'] = {'mode': 'ggH', 'factor': 0.1387}
globalXSBRMap['STXS']['GG2H_0J_PTH_GT10'] = {'mode': 'ggH', 'factor': 0.3940}
globalXSBRMap['STXS']['GG2H_1J_PTH_0_60'] = {'mode': 'ggH', 'factor': 0.1477}
# ... more bins ...

# VBF STXS bins
globalXSBRMap['STXS']['VBF_GE2J_MJJ_350_700_PTH_0_200_PTHJJ_0_25'] = {'mode': 'qqH', 'factor': 0.1026}
# ... more bins ...

# Associated production with decay branching ratios
globalXSBRMap['STXS']['WH2HQQ_1J'] = {'mode': 'WH', 'factor': 0.3113 * BR_W_qq}
globalXSBRMap['STXS']['GG2HLL_PTV_75_150'] = {'mode': 'ggZH', 'factor': 0.4325 * BR_Z_ll}
```

**When to use**:
- Differential cross section measurements
- STXS stage 1.2 (or other) binning schemes
- Multiple production mechanism topologies

**Understanding the factors**:
- For STXS bins: The factor is the **fraction of the total production mode** cross section in that kinematic region
  - Example: `GG2H_0J_PTH_0_10` has factor 0.1387 → 13.87% of all ggH events fall in this bin
  - The sum of all ggH STXS bin factors ≈ 1.0
- For associated production: Additional branching ratio factor for V→ff decays
  - Example: `WH2HQQ` includes `BR_W_qq` (W→qq branching ratio ~67.4%)

### Example 4: Custom BSM Process

For Beyond Standard Model (BSM) signals with arbitrary normalizations:

```python
globalXSBRMap['myBSM'] = od()
globalXSBRMap['myBSM']['decay'] = {'mode': 'hgg'}
globalXSBRMap['myBSM']['newPhysicsProc'] = {'mode': 'constant', 'factor': 0.001}  # 1 fb cross section
```

**When to use**:
- BSM signal searches
- Resonance searches
- Any process not in SM production modes

## Technical Implementation

### Code Flow

1. **Signal Fit Script** (`Signal/scripts/signalFit.py`):
   ```python
   # Load the XSBRMap for your analysis
   xsbrMap = globalXSBRMap[opt.analysis]
   
   # Pass to FinalModel constructor
   model = FinalModel(..., _xsbrMap=xsbrMap, ...)
   ```

2. **FinalModel Class** (`Signal/tools/finalModel.py`):
   ```python
   class FinalModel:
       def __init__(self, ..., _xsbrMap, ...):
           self.xsbrMap = _xsbrMap
           self.XSBR = initialiseXSBR()  # Get base XS/BR from Combine
           self.buildXSBRSplines()  # Build MH-dependent splines
   ```

3. **Spline Construction**:
   - Creates RooSpline1D objects for smooth MH interpolation
   - Splines evaluated during fit to get XS and BR at any MH value
   - Used in final normalization: `N = xs(MH) × br(MH) × ea(MH) × L`

4. **Systematic Effects**:
   - Rate systematics can modify the overall normalization
   - Shape systematics affect the PDF parameters
   - Both are combined with the XS×BR normalization

### Data Sources

The base cross sections and branching ratios come from:
- **LHC Higgs Cross Section Working Group (LHCHXSWG)** recommendations
- Stored in Combine's data files: `HiggsAnalysis/CombinedLimit/data/lhc-hxswg/sm/`
- Accessed via `SMHiggsBuilder` class in Combine

### Available Constants

Common branching ratio constants defined in `tools/commonObjects.py`:
```python
BR_W_qq = 0.6741      # W → hadrons
BR_W_lnu = 0.3259     # W → leptons
BR_Z_qq = 0.6991      # Z → hadrons
BR_Z_ll = 0.06730     # Z → charged leptons
BR_Z_nunu = 0.2000    # Z → neutrinos
```

## Best Practices

### 1. Creating a New Analysis XSBRMap

**Step-by-step**:

a. **Identify your signal processes**:
   - List all signal processes in your analysis
   - Determine if they map to SM production modes or need custom normalization

b. **Choose appropriate modes**:
   - Use SM production modes ('ggH', 'qqH', etc.) when possible
   - Use 'constant' for fixed cross sections or BSM signals

c. **Determine factors**:
   - STXS bins: Use theoretical predictions for bin fractions
   - Associated production: Include V decay branching ratios
   - Custom processes: Set to desired cross section value

d. **Add to XSBRMap.py**:
   ```python
   globalXSBRMap['myAnalysis'] = od()
   globalXSBRMap['myAnalysis']['decay'] = {'mode': 'hgg'}
   # Add all processes...
   ```

e. **Test**:
   ```bash
   python3 RunSignalScripts.py --inputConfig config_myAnalysis.py --mode signalFit
   ```

### 2. Validation

**Check your XSBRMap**:

- **Completeness**: Ensure every signal process in your workspace has an entry
- **Factor sum**: For STXS bins of same production mode, factors should sum to ~1.0
- **Branching ratios**: Use standard values from PDG/commonObjects.py
- **Cross sections**: Verify against LHCHXSWG recommendations

**Common errors**:
```
[ERROR] XS * BR map does not exist for analysis (myAnalysis)
```
→ Add your analysis to `globalXSBRMap` in `XSBRMap.py`

```
KeyError: 'MY_PROCESS'
```
→ Add the process entry to your analysis XSBRMap

### 3. Debugging

Enable plotting to visualize normalization:
```bash
python3 RunSignalScripts.py --inputConfig config.py --mode signalFit --modeOpts "--doPlots"
```

This creates plots showing:
- XS and BR as functions of MH
- Efficiency × Acceptance splines
- Total normalization for each (process, category)

### 4. Documentation

When adding a new analysis XSBRMap:
- Comment the physical interpretation of factors
- Document the source of STXS fractions or branching ratios
- Note any approximations or assumptions

Example:
```python
# STXS ggH bins: fractions from STXS stage 1.2 theory predictions
# Reference: LHC Higgs Cross Section Working Group recommendations
globalXSBRMap['STXS']['GG2H_0J_PTH_0_10'] = {'mode': 'ggH', 'factor': 0.1387}  # 13.87% of ggH
```

### 5. Consistency with Replacement Map

The processes in XSBRMap must be consistent with `replacementMap.py`:
- All processes in your analysis should appear in both maps
- Process names must match exactly
- Replacement processes must also have XSBRMap entries

## Summary

The XSBRMap is essential for proper signal normalization in Final Fits:

- **Purpose**: Provides theoretical cross sections and branching ratios for signal processes
- **Usage**: Loaded during signal fitting and used to build MH-dependent normalization splines
- **Configuration**: Define in `XSBRMap.py` for each analysis
- **Flexibility**: Supports SM production modes, STXS bins, and custom normalizations

For any signal modeling task, ensure your XSBRMap is:
1. ✓ Complete (all processes covered)
2. ✓ Accurate (correct factors and modes)
3. ✓ Consistent (matches your analysis design)
4. ✓ Documented (clear comments explaining choices)

## Related Documentation

- **Signal Modeling**: `Signal/README.md` - Full signal modeling workflow
- **Replacement Map**: `Signal/tools/replacementMap.py` - Process replacement strategy
- **Configuration**: `QUICKSTART.md` - Setting up your analysis config
- **Package Overview**: `INVESTIGATION.md` - Comprehensive package overview including STXS framework

## Questions?

If you encounter issues:
1. Check that your analysis name in config matches an entry in `globalXSBRMap`
2. Verify all process names match between workspace, XSBRMap, and replacementMap
3. Review example configurations (tutorial, STXS) for patterns
4. Use `--doPlots` option to visualize normalizations
