---
title: "Constrained Spherical Deconvolution (CSD)"
teaching: 30
exercises: 5
questions:
- "What is Constrained Spherical Deconvolution (CSD)?"
- "What does CSD offer compared to DTI?"
objectives:
- "Understand Spherical Deconvolution"
- "Visualizing the fiber Orientation Distribution Function"
keypoints:
- "CSD uses the information along more gradient encoding directions"
- "It allows to resolve complex fiber configurations, such as crossings"
---

## Constrained Spherical Deconvolution (CSD)

Spherical Deconvolution (SD) is a set of methods to reconstruct the local fiber
Orientation Distribution Functions (fODF) from diffusion MRI data. They have
become a popular choice for recovering the fiber orientation due to their
ability to resolve fiber crossings with small inter-fiber angles in datasets
acquired within a clinically feasible scan time. SD methods are based on the
assumption that the acquired diffusion signal in each voxel can be modeled as a
spherical convolution between the fODF and the fiber response function (FRF)
that describes the common signal profile from the white matter (WM) bundles
contained in the voxel. Thus, if the FRF can be estimated, the fODF can be
recovered as a deconvolution problem by solving a system of linear equations.
These methods can work on both single-shell and multi-shell data.

The basic equations of an SD method can be summarized as
![spherical_deconvolution_equation](../fig/constrained_spherical_deconvolution/spherical_deconvolution_equation.png){:class="img-responsive"} \
Spherical deconvolution

There are a number of variants to the general SD framework that differ, among
others, in the minimization objective and the regularization penalty imposed to
obtain some desirable properties in the linear equation framework.

In order to perform the deconvolution over the sphere, the spherical
representation of the diffusion data has to be obtained. This is done using the
so-called Spherical Harmonics (SH) which are a basis that allow to represent
any function on the sphere (much like the Fourier analysis allows to represent
a function in terms of in terms of trigonometric functions).

In this episode we will be using the Constrained Spherical Deconvolution (CSD)
method proposed by Tournier *et al*. in 2007. In essence, CSD imposes a
non-negativity constraint in the reconstructed fODF. For the sake of
simplicity, single-shell data will be used in this episode.

Let's start by loading the necessary data. For simplicity, we will assume that
the gradient table is the same across all voxels after the pre-processing.

~~~
import os

import nibabel as nib
import numpy as np

from bids.layout import BIDSLayout

from dipy.core.gradients import gradient_table
from dipy.data import default_sphere
from dipy.io.gradients import read_bvals_bvecs
from dipy.io.image import load_nifti


dwi_layout = BIDSLayout('../../data/ds000221/derivatives/uncorrected_topup_eddy/', validate=False)
t1_layout = BIDSLayout('../../data/ds000221/derivatives/uncorrected_topup_eddy_regT1/', validate=False)
gradient_layout = BIDSLayout('../../data/ds000221/sub-010006/ses-01/dwi/', validate=False)

subj = '010006'

# Get the diffusion files
dwi_fname = dwi_layout.get(subject=subj, suffix='dwi', extension='nii.gz', return_type='file')[0]
bval_fname = gradient_layout.get(subject=subj, suffix='dwi', extension='bval', return_type='file')[0]
bvec_fname = gradient_layout.get(subject=subj, suffix='dwi', extension='bvec', return_type='file')[0]

# Get the anatomical file
t1w_fname = t1_layout.get(subject=subj, extension='nii.gz', return_type='file')[0]

data, affine = load_nifti(dwi_fname)

bvals, bvecs = read_bvals_bvecs(bval_fname, bvec_fname)
gtab = gradient_table(bvals, bvecs)
~~~
{: .language-python}

You can verify the b-values of the dataset by looking at the attribute
`gtab.bvals`. Now that a datasets with multiple gradient directions is loaded,
we can proceed with the two steps of CSD.

## Step 1. Estimation of the fiber response function.

In this episode the response function will be estimated from a local brain
region known to belong to the white matter and where it is known that there are
single coherent fiber populations. This is determined by checking the
Fractional Anisotropy (FA) derived from the DTI model.

For example, if we use an ROI at the center of the brain, we will
find single fibers from the corpus callosum. `DIPY`'s `auto_response` function
will calculate FA for an ROI of radius equal to `roi_radius` in the center
of the volume and return the response function estimated in that region for
the voxels with FA higher than a given threshold.

~~~
from dipy.reconst.csdeconv import auto_response

response, ratio = auto_response(gtab, data, roi_radius=10, fa_thr=0.7)

# Create the directory to save the results
out_dir = '../../data/ds000221/derivatives/dwi/reconstruction/sub-%s/ses-01/dwi/' % subj

if not os.path.exists(out_dir):
    os.makedirs(out_dir)

# Save the FRF
np.savetxt(os.path.join(out_dir, 'frf.txt'), np.hstack([response[0], response[1]]))
~~~
{: .language-python}

The `response` tuple contains two elements. The first is an array with
the eigenvalues of the response function and the second is the average `S0`
signal value for this response.

Validating the numerical value of the response function is recommended to
ensure that the FA-based strategy provides a good result. To this end, the
elements of the `response` tuple can be printed and their values be studied.

~~~
print(response)
~~~
{: .language-python}

~~~
(array([0.00159258, 0.00033926, 0.00033926]), 201.48318)
~~~
{: .output}

The tensor generated belonging to the response function must be prolate (two
smaller eigenvalues should be equal), and look anisotropic with a ratio of
second to first eigenvalue of about 0.2. Or in other words, the axial
diffusivity of this tensor should be around 5 times larger than the radial
diffusivity. It is generally accepted that a response function with the
mentioned features is representative of a coherently oriented fiber population.

~~~
print(ratio)
~~~
{: .language-python}

~~~
0.21302577723299593
~~~
{: .output}

It is good practice to visualize the response function's ODF, which also gives
an insightful idea around the SD framework. The response function's ODF should
have sharp lobes, as the anisotropy of its diffusivity indicates:

~~~
from dipy.viz import window, actor
from dipy.sims.voxel import single_tensor_odf

scene = window.Scene()
evals = response[0]
evecs = np.array([[0, 1, 0], [0, 0, 1], [1, 0, 0]]).T

response_odf = single_tensor_odf(default_sphere.vertices, evals, evecs)
# transform our data from 1D to 4D
response_odf = response_odf[None, None, None, :]
response_actor = actor.odf_slicer(response_odf, sphere=default_sphere,
                                  colormap='plasma')
scene.add(response_actor)
response_scene_arr = window.snapshot(
    scene, fname=os.path.join(out_dir, 'frf.png'), size=(200, 200),
    offscreen=True)

import matplotlib.pyplot as plt
%matplotlib inline

fig, axes = plt.subplots()
axes.imshow(response_scene_arr, cmap="plasma", origin="lower")
axes.axis("off")
plt.show()
~~~
{: .language-python}

![frf](../fig/constrained_spherical_deconvolution/frf.png){:class="img-responsive"} \
Estimated response function


~~~
scene.rm(response_actor)
~~~
{: .language-python}


Note that, although fast, the FA threshold might not always be the best way to
find the response function, since it depends on the diffusion tensor, which has
a number of limitations. Similarly, different bundles are known to have
different response functions. More importantly, it also varies across subjects,
and hence it must be computed on a case basis.


## Step 2. fODF reconstruction

After estimating a response function, the fODF is reconstructed through the
deconvolution operation. In order to obtain the spherical representation of the
diffusion signal, the order of the Spherical Harmonics expansion must be
specified. The series is infinite, but must be truncated to a maximum order in
practice to be able to represent the diffusion signal. The maximum order will
determine the number of SH coefficients used. The number of diffusion encoding
gradient directions must be at least as large as the number of coefficients.
Hence, the maximum order $l_{max}$ is determined by the equation $R = (l_{max}+1)(l_{max}+2)/2$,
where $R$ is the number of coefficients. For example, an order $l_{max} = {4, 6, 8}$ SH
series has $R = {15, 28, 45}$ coefficients, respectively. Note the use of even
orders: even order SH functions allow to reconstruct symmetric spherical
functions. Traditionally, even orders have been used motivated by the fact that
the diffusion process is symmetric around the origin.

The CSD is performed in `DIPY` by calling the `fit` method of the CSD model on
the diffusion data:

~~~
from dipy.reconst.csdeconv import ConstrainedSphericalDeconvModel

sh_order = 8
csd_model = ConstrainedSphericalDeconvModel(gtab, response, sh_order=sh_order)
~~~
{: .language-python}

For illustration purposes we will fit only a small portion of the data.

~~~
csd_fit = csd_model.fit(data)
~~~
{: .language-python}

Getting the fODFs from the model fit is straightforward in `DIPY`. As a side
note, it is worthwhile mentioning that the orientation distribution recovered
by SD methods is also named fODFs to distinguish from the diffusion ODFs
(dODFs) that other reconstruction methods recover. The former are considered to
be a sharper version of the latter. At times, they are also called Fiber
Orientation Distribution (FOD).

~~~
csd_odf = csd_fit.odf(default_sphere)
~~~
{: .language-python}

Here we visualize only a 30x30 region (i.e. the slice corresponding to the
`[20:50, 55:85, 38:39]` volume data region that was used in the tutorial).

~~~
fodf_spheres = actor.odf_slicer(csd_odf, sphere=default_sphere, scale=0.9,
                                norm=False, colormap='plasma')

scene.add(fodf_spheres)
fodf_scene_arr = window.snapshot(
    scene, fname=os.path.join(out_dir, 'csd_odfs.png'), size=(600, 600),
    offscreen=True)

fig, axes = plt.subplots()
axes.imshow(fodf_scene_arr, cmap="plasma", origin="lower")
axes.axis("off")
plt.show()
~~~
{: .language-python}


![csd_odfs](../fig/constrained_spherical_deconvolution/csd_odfs.png){:class="img-responsive"} \
CSD ODFs.

The peak directions (maxima) of the fODFs can be found from the fODFs. For this
purpose, `DIPY` offers the `peaks_from_model` method.

~~~
from dipy.direction import peaks_from_model

csd_peaks = peaks_from_model(model=csd_model,
                             data=data,
                             sphere=default_sphere,
                             relative_peak_threshold=.5,
                             min_separation_angle=25,
                             parallel=True)

window.clear(scene)
fodf_peaks = actor.peak_slicer(csd_peaks.peak_dirs, csd_peaks.peak_values)
scene.add(fodf_peaks)
peaks_scene_arr = window.snapshot(
    scene, fname=os.path.join(out_dir, 'csd_peaks.png'), size=(600, 600),
    offscreen=True)

fig, axes = plt.subplots()
axes.imshow(peaks_scene_arr, origin="lower")
axes.axis("off")
plt.show()
~~~
{: .language-python}

![csd_peaks](../fig/constrained_spherical_deconvolution/csd_peaks.png){:class="img-responsive"} \
CSD Peaks.

We can finally visualize both the fODFs and peaks in the same space.

~~~
fodf_spheres.GetProperty().SetOpacity(0.4)

scene.add(fodf_spheres)
csd_peaks_fodfs_arr = window.snapshot(
    scene, fname=os.path.join(out_dir, 'csd_peaks_fodfs.png'), size=(600, 600),
    offscreen=True)

fig, axes = plt.subplots()
axes.imshow(csd_peaks_fodfs_arr, cmap="plasma", origin="lower")
axes.axis("off")
plt.show()
~~~
{: .language-python}

![csd_peaks_fodfs](../fig/constrained_spherical_deconvolution/csd_peaks_fodfs.png){:class="img-responsive"} \
CSD Peaks and ODFs.



References
----------

.. [Tournier2007] J-D. Tournier, F. Calamante and A. Connelly, "Robust
   determination of the fibre orientation distribution in diffusion MRI:
   Non-negativity constrained super-resolved spherical deconvolution",
   Neuroimage, vol. 35, no. 4, pp. 1459-1472, 2007.


{% include links.md %}
