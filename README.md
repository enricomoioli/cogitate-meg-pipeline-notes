# Getting the Cogitate Consortium MEG pipeline running on Windows

Some notes on getting the Cogitate Consortium's `cogitate-msp1` MEG pipeline to run on a normal Windows machine, as part of my master's thesis (extending their IIT vs GNWT work with GED and connectivity analyses).

So far I've mostly been fighting the environment and getting preprocessing to run. Below are six issues I actually hit (a couple were caught by reading the code before running it, not by trial and error). I'm keeping this as problem/fix instead of writing it up nicer, because honestly the debugging is the part worth showing.

The environment file I ended up with is in this repo: `requirements_cogitate_meg.yml`.

The consortium's original file was built for their HPC cluster and doesn't install as-is anywhere else, so most of this is about getting from their file to something that actually works on Windows.

## Fix 1: netCDF4 wouldn't build

First attempt at building the environment failed. `netCDF4` gets pulled in indirectly by `nipype`/`neuropythy` but was never listed explicitly in the original file, so pip tried to compile it from source on Windows, which needs Visual C++ Build Tools I didn't have.

Fix: added `netcdf4` explicitly as a conda dependency instead. conda-forge has precompiled binaries, pip building from source doesn't.

## Fix 2: autoreject and mne fighting each other

Second attempt looked like it worked, but `import mne` failed, it just wasn't there. Turns out `autoreject` wasn't pinned in the original file, so it grabbed the newest version (0.4.3), which needs `mne>=1.0` and quietly bumped out the `mne==0.24.0` the whole pipeline (GED and connectivity included) is actually written against. Pretty typical dependency drift issue, a file written a while ago stops working because unpinned packages resolve against today, not against whenever the file was written.

I found the fix by comparing against `requirements_cogitate_meg_exact.yml`, which is a real `conda env export` from the consortium's actual HPC environment (path was something like `/hpc/users/oscar.ferrante/.conda/envs/mne_meg01`). That file is Linux only so I couldn't just use it directly, but it told me the real answer: `autoreject==0.2.2`.

So I pinned that in my Windows-compatible file instead of copying the whole HPC file over. Same thing for `mne-connectivity==0.2`.

## Fix 3: frites wasn't there at all

This one I found just by reading through the code, not by running it. Went through all 14 pipeline scripts (7 preprocessing, 3 GED, 4 connectivity) and checked every import against what was actually installed. `connectivity/Co01c_connect_dfc.py` imports `frites.conn` (it's the mutual information part of Prediction 3), and `frites` isn't in the environment file anywhere. Not a version mismatch like the mne thing, just completely missing, would have blown up the first time anyone ran that script.

Added `frites==0.4.3`, same version as in the HPC export.

## Fix 4: fpdf vs fpdf2

The environment file just says `fpdf`, which pip installs as the old `fpdf` 1.7.2 package, hasn't been touched in years. But the script is actually written for `fpdf2`, the maintained fork. Same import line (`from fpdf import FPDF`) so it's not obvious at first, but fpdf2 has extra stuff like `.epw` that the old one doesn't. Got `AttributeError: 'FPDF' object has no attribute 'epw'`.

Fixed with `pip uninstall fpdf` then `pip install fpdf2`, didn't need to rebuild the whole environment. Updated the yml file too so it's fixed for next time.

## Fix 5: compute_psd doesn't exist in this mne version

Different kind of bug. `AttributeError: 'Raw' object has no attribute 'compute_psd'`, inside the function that plots the power spectrum, not in any actual analysis. Looked it up, `Raw.compute_psd()` only exists from MNE 1.2.0 onward (Oct 2022), a year after the 0.24.0 version everything is deliberately pinned to. So this script was never actually compatible with the mne version the rest of the pipeline needs.

What's funny is the old, correct line was still sitting right there in the file, commented out (`from mne.time_frequency import psd_multitaper`), with someone's initials next to it. Looks like someone updated the script for a newer mne at some point and just left the old code commented instead of deleting it, without updating the environment file to match.

Just uncommented the original line instead of writing anything new.

## Fix 6: this one's just not enough RAM

Not a package issue. Some preprocessing steps load multi-gigabyte arrays into memory in one go, which is fine on a cluster but nobody wrote down as a requirement anywhere. On my machine (8GB RAM) it started swapping to disk during preprocessing already. Got `numpy.core._exceptions.MemoryError: Unable to allocate 2.15 GiB for an array with shape (341, 846000)`.

Also noticed it's not totally consistent, the same step on the same data sometimes works and sometimes doesn't, depending on how fragmented the memory is at that moment. Nothing to fix here really, just leaving it documented in case someone else tries this on a normal laptop.

---

Fork with this environment file, PR to the original repo: link coming once it's open.
