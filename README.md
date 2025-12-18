# Cryo-EM-ice-detection-Mic-hacakthon
This project presents a machine learning based approach for automatically detecting ice contamination in cryo-electron microscopy (cryo-EM) micrographs. Using expert-curated annotations from the CryoPPP dataset (EMPIAR-10532), image patches are labeled as 'BAD' and "GOOD' and used to train a convolutional Neural Network.

The pipeline used in this project is as follows:

Step1: Inputs from CryoPPP 
	Micrograph: a 2D image recorded by the detector (e.g., .mrc)
	True particle coordinates: points where experts say “this is a real particle”
	False-positive coordinates: points where experts say “this is not a particle” (often ice/contamination/carbon)
	
Output of Step 1
	A list of micrographs + two sets of points per micrograph (true vs false)

Step2: Build training examples (patch)

2A. BAD patches (artifact-enriched)
	For every false-positive coordinate (x, y), extract a square patch centered there (e.g., 256×256 px).
	Label = BAD
	Rationale: false positives are often caused by ice artifacts or contamination that look particle-like.

2B. GOOD patches (clean background)
	Randomly choose patch centers in “empty” regions, but require they are:
	far from any false-positive point
	far from any true particle point
	Label = GOOD
	Rationale: avoids including real protein or known artifact neighborhoods.

Output of Step 2
	A dataset of many labeled patches
	
Step3: Add FFT as a second view (optional)
	For each patch, compute:
	real-space patch (what it looks like)

FFT “power spectrum” patch (what repeating patterns exist)
An FFT (Fast Fourier Transform) converts an image patch from:
real space (pixels) to frequency space (patterns of repeating structure)

Why this helps:
Crystalline ice produces strong ring patterns in frequency space (power spectrum).
Thick ice reduces high-frequency content (signal “drops off”).

In cryo-EM, people often look at the power spectrum / FFT to judge ice quality—this just makes the model do something similar.

Output of Step 3
	Each training example becomes:
		Input =[real_patch, fft_patch]
		Label = GOOD or BAD
	
Step 4 — Train a CNN classifier
	Training task
	Input: a patch (and optionally its FFT)
	Output: probability the patch is BAD (artifact-like)

Output of Step 4
	A trained CNN that can score any patch:
	P(BAD | patch)

Step 5 — Scan a whole micrograph to create an ice mask
	Move a square window across the micrograph (sliding window)
	For each location:
		extract patch (+ FFT)
		CNN outputs P(BAD)
	Combine all probabilities into a 2D map over the micrograph

Output of Step 5
	An artifact probability map
	A binary mask after thresholding:
	1 = good region
	0 = bad region (ice/artifacts)
	
Step 6 — “Delete” ice artifacts safely (masking)
	Set BAD regions to 0/NaN for visualization, or
	Exclude BAD regions during particle picking and filtering

Output of Step 6
	A micrograph mask + a filtered particle set (remove particles that land in BAD regions)
