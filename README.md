# neurofxn

Use-case specific python-based calcium imaging preprocessing and event detection pipeline for neurons aggregated into spheroids and placed in a hydrogel microcolumn.

Adapted corresponding methods section from forthcoming paper:

Developed in python 3.10.9.

Motion correction: First motion-corrects videos using a non-rigid flow algorithm due to movement of the microcolumn in the z-plane. Videos should be inspected by eye for artifactual warping due to sudden movements. 

Codebase includes custom correction for warping in select videos, whereby motion correction by translation was applied to some time windows prior to applying nonrigid flow. Videos from the same session were translationally aligned and concatenated. 

Global ROI mask: A mask of the microcolumn is created automatically using the following process: 
1)	The max projection over time is taken of the motion-corrected video
2)	The max projection over time is spatially smoothed (filtered by a gaussian with sigma = 0.8 = 1.95 μm)
3)	The top 30% brightest pixels are selected for inclusion in an initial mask
4)	(For one well only, the mask was subjected to morphological closing with a kernel of size 15.)
5)	The mask is flood-filled in the center to the right-hand side.
6)	The mask is subjected to morphological closing with a kernel of size 7
   
Masks should be visually inspected to ensure they adequately capture the microcolumn.

Delta F over F0 normalization: 

Pixel values are converted to delta f over f0, where f0 is the mean activity level of the background pixels (those excluded from the mask) at each frame. Pixels outside the global mask are set to 0.

Initial detrending: 

A global trend video is made by spatially filtering each frame of the delta f over f0 video with a gaussian blur (sigma = 20) and low pass filtering with a cutoff of 0.45 Hz (we initially targeted 0.5 Hz but one video had a slower sampling rate and we wanted to apply the same cut-off to all videos). Then each pixel value in the delta f over f0 video is predicted using the global trend video, and the residuals from the simple linear regression model are output into a video of residuals excluding global trends.

Event detection:

Event detection is carried out using the CaImAn package in python (version 1.11.0) on the motion-corrected and detrended videos. The CaImAn CNMF-E (Constrained Non-negative Matrix Factorization – Endoscopic) event detection algorithm works by applying additional background subtraction in a ring around candidate ROI pixels, and using constrained non-negative matrix factorization to identify blob-like spatial ROIs that have event-like activity above background. Input videos require the following additional preprocessing for use with this algorithm: 1) To save computationally, videos should be cropped more closely to a rectangle including the global ROI mask. 2) Since the algorithm cannot handle static pixels, gaussian noise with a standard deviation equal to the median standard deviation of all non-static pixels must be added to all static pixels. 3) Voxels must be rendered non-negative; here we do so by subtracting the minimum value in the video across both space and time. Then the CaImAn CNMF-E event detection algorithm is applied using the CNMF function. The following key parameters are used within the script: ring size of 1.2 (for ring-based additional background subtraction), with a low-pass filter reflecting the expected half-width of neurons with a kernel size of 3 pixels (7.32 microns) in both the x and y directions, and minimum signal correlation of 0.4 and minimum peak-to-noise ratio of 7 for lumping adjacent pixels into (putatively cellular level) ROIs.
