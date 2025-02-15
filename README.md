
<img src="figures/swm_logo.png" width=20% height=20% align=left>

    
# Superficial White Matter
[![GitHub issues](https://img.shields.io/github/issues/jordandekraker/superficial-white-matter)](https://github.com/jordandekraker/superficial-white-matter/issues)
[![GitHub stars](https://img.shields.io/github/stars/jordandekraker/superficial-white-matter.svg?style=flat&label=⭐%EF%B8%8F%20stars&color=brightgreen)](https://github.com/jordandekraker/superficial-white-matter/stargazers)

Generates surfaces at various white matter depths (default 1, 2, and 3 milimiters).
The depths are calculated based on the real world image resolution voxel size and transformed to milimiters.

## Method
This is done by first computing a Laplace field over white matter (cortex to subcortex+ventricles), and then shifting an exiting white matter surface along that gradient.
Stopping conditions are set by geodesic distance travelled.

![swm method](figures/swm_methods.png)

> White is the original wm surface, red, orange and yellow are depths 1mm, 2mm, and 3mm accordingly.

## Installation
```
git clone https://github.com/jordandekraker/superficial-white-matter.git
pip install superficial-white-matter/
```

## Usage with Freesurfer/Fastsurfer (example)

> The code expects *standard* nifti orientation, in which the resolution is the diagonal of the header affine matrix. Running the inputs through `fslreorient2std` will ensure that everything is calculated correctly.

```bash
# This is the freesurfer/fastSurfer Subject directory
SUBJECTS_DIR=<path to surface subjects directory FreeSurfer/FastSurfer>

# Subject to process
SUBJECT=sub-01

# Output directory
OUT=<path to output directory>

# Convert segmentation to NIFTI
aparc_aseg=${OUT}/${SUBJECT}_aparc+aseg.nii.gz
mri_convert ${SUBJECTS_DIR}/${SUBJECT}/mri/aparc+aseg.mgz ${aparc_aseg}

# Reorient to standard
fslreorient2std ${aparc_aseg} ${aparc_aseg}

# 1. Calculate the Laplace field
python sWM/laplace_solver.py \
  ${aparc_aseg} \
  ${OUT}/${SUBJECT}_laplace-wm.nii.gz

# 2. Generate the surfaces for each hemisphere
for hemi in lh rh; do
  # White matter surface
  WM=${SUBJECTS_DIR}/${SUBJECT}/surf/${hemi}.white
  WM_gii=${OUT}/${SUBJECT}_hemi-${hemi}_label-white.surf.gii

  # Convert white matter to GIFTI
  mris_convert ${WM} ${WM_gii}

  # Calculate the SWM surfaces
  python sWM/surface_generator.py \
    "${WM_gii}" \
    ${OUT}/${SUBJECT}_laplace-wm.nii.gz \
    ${OUT}/${SUBJECT}_hemi-${hemi}_label-sWF_depth-
done

```

> If you ran `micapipe v0.2.0` or higher check the example script:  [`example_usage.sh`](./example_usage.sh)
> `SWM` is implemented in [`micapipe v0.2.3`](https://github.com/MICA-MNI/micapipe/releases/tag/v0.2.3)

## `laplace_solver.py`
```python
Solves Laplace equation over the domain of white matter.

Using grey matter as the source and ventricles as the sink.
Inputs are expected to be Free/FastSurfer aparc+aseg.mgz in .nii.gz format

Parameters
----------
NIFTI  :    str
            Parcellation file generated by Freesurfer/fastsurfre in nii.gz format (from mri/aparc+aseg.mgz).
NIFTI  :    str
            Output laplacian file path (nii.gz)

Returns
-------
NIFTI
    Laplacian image (nii.gz)

Usage
-----
laplace_solver.py aparc+aseg.nii.gz laplace-wm.nii.gz
```

## `surface_generator.py`
```python
Shifts a white matter surface inward along a Laplace field

Parameters
----------
GIFTI  :    str
            White matter surface in GIFTI format (surf.gii)
NIFTI  :    str
            laplacian image generated by laplace_solver.py
OUTPUT :    str
            path and name to the output surfaces
DEPTHS :    list [int | float] (OPTIONAL)
            DEFAULT=[1,2,3] List of depths to sample (in voxels)

Returns
-------
NIFTI
    a list of strings representing the header columns

Usage
-----
surface_generator.py hemi-L_label-white.surf.gii laplace-wm.nii.gz hemi-L_label-sWF_depth- 1,2,3
```

Superficial White Matter © 2023 by Jordan DeKraker is licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/
