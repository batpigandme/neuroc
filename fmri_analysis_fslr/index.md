# An example of an fMRI analysis in FSL
John Muschelli  
`r Sys.Date()`  

All code for this document is located at [here](https://raw.githubusercontent.com/muschellij2/neuroc/master/fmri_analysis_fslr/index.R).





```r
library(methods)
library(fslr)
```

```
Loading required package: oro.nifti
```

```
oro.nifti 0.7.2
```

```
Loading required package: neurobase
```

```r
library(neurobase)
library(extrantsr)
```

In this tutorial we will discuss performing some preprocessing of a single subject functional MRI in FSL.  

# Data Packages

For this analysis, I will use one subject from the Kirby 21 data set.  The `kirby21.base` and `kirby21.fmri` packages are necessary for this analysis and have the data we will be working on.  You need devtools to install these.  Please refer to [installing devtools](../installing_devtools/index.html) for additional instructions or troubleshooting.



```r
packages = installed.packages()
packages = packages[, "Package"]
if (!"kirby21.base" %in% packages) {
  devtools::install_github("muschellij2/kirby21.base")
}
if (!"kirby21.fmri" %in% packages) {
  devtools::install_github("muschellij2/kirby21.fmri")
}
```

# Loading Data

We will use the `get_image_filenames_df` function to extract the filenames on our hard disk for the T1 image and the fMRI images (4D).  


```r
library(kirby21.fmri)
library(kirby21.base)
fnames = get_image_filenames_df(ids = 113, 
                    modalities = c("T1", "fMRI"), 
                    visits = c(1),
                    long = FALSE)
t1_fname = fnames$T1[1]
fmri_fname = fnames$fMRI[1]
base_fname = nii.stub(fmri_fname, bn = TRUE)
```

## Parameter file

If you'd like to see the header information from the fMRI data, it is located by the following commands:


```r
library(R.utils)
par_file = system.file("visit_1/113/113-01-fMRI.par.gz", 
                       package = "kirby21.fmri")
# unzip it
con = gunzip(par_file, temporary = TRUE, 
             remove = FALSE, overwrite = TRUE)
info = readLines(con = con)
info[11:23]
```

```
 [1] ".    Protocol name                      :   WIP Bold_Rest SENSE"
 [2] ".    Series Type                        :   Image   MRSERIES"   
 [3] ".    Acquisition nr                     :   11"                 
 [4] ".    Reconstruction nr                  :   1"                  
 [5] ".    Scan Duration [sec]                :   434"                
 [6] ".    Max. number of cardiac phases      :   1"                  
 [7] ".    Max. number of echoes              :   1"                  
 [8] ".    Max. number of slices/locations    :   37"                 
 [9] ".    Max. number of dynamics            :   210"                
[10] ".    Max. number of mixes               :   1"                  
[11] ".    Patient position                   :   Head First Supine"  
[12] ".    Preparation direction              :   Anterior-Posterior" 
[13] ".    Technique                          :   FEEPI"              
```

From the paper ["Multi-parametric neuroimaging reproducibility: A 3-T resource study"](http://dx.doi.org/10.1016/j.neuroimage.2010.11.047), which this data is based on, it describes the fMRI sequence:

> The sequence used for resting state functional connectivity MRI is typically identical to that used for BOLD functional MRI studies of task activation. Here, we used a 2D EPI sequence with SENSE partial-parallel imaging acceleration to obtain 3 × 3 mm (80 by 80 voxels) in-plane resolution in thirty-seven 3 mm transverse slices with 1 mm slice gap. An ascending slice order with TR/TE = 2000/30 ms, flip angle of 75°, and SENSE acceleration factor of 2 were used. SPIR was used for fat suppression. This study used an ascending slice acquisition order because a pilot studies revealed smaller motion induced artifacts with ascending slice order than with interleaved slice order. While using an ascending slice order, it was necessary to use a small slice gap to prevent cross talk between the slices. One 7-min run was recorded which provided 210 time points (discarding the first four volumes to achieve steady state).


# Outline 
The steps I will perform in this analysis:

1. Calculation of Motion Parameters (`fslr::mcflirt`)
1. Slice timing correction (`fslr::fsl_slicetimer`), but we need to know how the scan was taken/slice order and repetition time (TR)
2. Motion Correction on Corrected Data (`fslr::mcflirt`)
3. Coregistration of fMRI and a T1-weighted image (`fslr::flirt`)
4. Registration to the Template space (`fslr::fnirt_with_affine` )
5. De-meaning the data (fslr::fslmean(ts = TRUE))
6. Skull stripping (fslr::fslbet)
7. Registration to a template using the T1 and then transforming the fMRI with it
8. Spatially smoothing the data (fslr:fslsmooth)
9. Tissue-class segmentation (fslr::fast, ANTsR::atropos or extrantsr::otropos)?
10. Bandpass/butterworth filtering (signal::butter, signal::buttord)
11. Get a connectivity matrix of certain regions, you need to specify an atlas.



Now we know that the head is first in (as usual) and the data was acquired in ascending order (i.e. bottom -> up) and the repetition time (TR) was 2 seconds   The 


```r
fmri = readnii(fmri_fname)
ortho2(fmri, w = 1, add.orient = FALSE)
```

![](index_files/figure-html/fmri-1.png)<!-- -->

```r
rm(list = "fmri") # just used for cleanup 
```

# Stabilization of Signal

Volumes corresponding to the first 10 seconds of the rs-fMRI scan were dropped to allow for magnetization stabilization.


```r
fmri = readnii(fmri_fname)
tr = 2 # 2 seconds
first_scan = floor(10.0 / tr) + 1 # 10 seconds "stabilization of signal"
sub_fmri = subset_4d(fmri, first_scan:ntim(fmri))
```
# Slice Timing Correction


```r
library(fslr)
scorr_fname = paste0(base_fname, "_scorr.nii.gz")
if (!file.exists(scorr_fname)) {
  scorr = fsl_slicetimer(file = sub_fmri, 
                 outfile = scorr_fname, 
                 tr = 2, direction = "z", acq_order = "contiguous",
                 indexing = "up")
} else {
  scorr = readnii(scorr_fname)
}
```

# Slice Timing Correction

Similarly to the ANTsR processing, we can set `opts = "-meanvol"` so that the motion correction registers the images to the mean of the time series.  The default otherwise is to register to the scan in the middle (closest to number of time points / 2).  You can supercede this with either specifying `-refvol` for which time point to register to, or `-reffile` to specify a volume to register to.  You can think of `-meanvol` as a wrapper for making the mean time series (using `fslmaths(opt = "-Tmean")`) and then passing that in as a `reffile`.  We will do this explicitly in the code below.  The `-plots` arguments outputs a `.par` file so that we can read the motion parameters after `mcflirt` is run.


```r
moco_fname = paste0(base_fname, 
                    "_motion_corr.nii.gz")
par_file = paste0(nii.stub(moco_fname), ".par")
avg_fname = paste0(base_fname, 
                   "_avg.nii.gz")
if (!file.exists(avg_fname)) {
  fsl_maths(file = sub_fmri, 
            outfile = avg_fname,
            opts = "-Tmean")
}

if (!all(file.exists(c(moco_fname, par_file)))) {
  moco_img = mcflirt(
    file = sub_fmri, 
    outfile = moco_fname,
    verbose = 2,
    opts = paste0("-reffile ", avg_fname, " -plots")
  )
} else {
  moco_img = readnii(moco_fname)
}
moco_params = readLines(par_file)
moco_params = strsplit(moco_params, split = " ")
moco_params = sapply(moco_params, function(x) {
  as.numeric(x[ !(x %in% "")])
})
moco_params = t(moco_params)
colnames(moco_params) = paste0("MOCOparam", 1:ncol(moco_params))
head(moco_params)
```

```
       MOCOparam1  MOCOparam2   MOCOparam3 MOCOparam4 MOCOparam5
[1,] -0.000309931 0.001248200 -4.26752e-04 0.03832110  -0.558039
[2,] -0.000650414 0.000826515 -5.37836e-04 0.03833610  -0.607328
[3,] -0.000581280 0.001141430 -4.26752e-04 0.03831010  -0.566965
[4,] -0.000906044 0.000826516 -4.26752e-04 0.02266840  -0.577867
[5,] -0.000188335 0.001106950 -1.23971e-04 0.01542600  -0.556718
[6,] -0.000605871 0.000849419 -6.14711e-06 0.00108561  -0.577868
     MOCOparam6
[1,]  0.0768606
[2,]  0.0589568
[3,]  0.0704959
[4,]  0.0465875
[5,]  0.0412824
[6,]  0.0439374
```


## Let's Make a Matrix!

`img_ts_to_matrix` creates $V\times T$ matrix, $V$ voxels in mask, unlike `ANTsR::timeseries2matrix`.  Therefore, we transpose the matrix so that it is consistent with the [ANTsR tutorial for fMRI](../fmri_analysis_ANTsR/index.html) and so `boldMatrix` is $T\times V$.  We will get the average of the co-registered image using `fslmaths`.  We wil use this average image to get a mask using the `oMask` function, which calls `getMask`, but for `nifti` objects.  We will then zero out the average image using the mask image.



```r
moco_avg_img = fslmaths(moco_fname, opts = "-Tmean")
```

```
fslmaths "/Users/johnmuschelli/Dropbox/Projects/neuroc/fmri_analysis_fslr/113-01-fMRI_motion_corr.nii.gz"  -Tmean "/var/folders/1s/wrtqcpxn685_zk570bnx9_rr0000gr/T//RtmpZOrA0i/file16f323480d370";
```

```r
maskImage = oMask(moco_avg_img, 
    mean(moco_avg_img), 
    Inf, cleanup = 2)
mask_fname = paste0(base_fname, "_mask.nii.gz")
writenii(maskImage, filename = mask_fname)

bet_mask = fslbet(moco_avg_img) > 0
```

```
bet2 "/private/var/folders/1s/wrtqcpxn685_zk570bnx9_rr0000gr/T/RtmpZOrA0i/file16f321d509357.nii.gz" "/var/folders/1s/wrtqcpxn685_zk570bnx9_rr0000gr/T//RtmpZOrA0i/file16f325bfe6f98" 
```

```r
bet_mask_fname = paste0(base_fname, "_bet_mask.nii.gz")
writenii(bet_mask, filename = bet_mask_fname)
```

We can also look at the differences.  We will assume the `maskImage` as the "gold standard" and `bet_mask` to be the "prediction".


```r
double_ortho(moco_avg_img, maskImage, 
  col.y = "white")
```

![](index_files/figure-html/plot_masks-1.png)<!-- -->

```r
double_ortho(moco_avg_img, bet_mask, 
  col.y = "white")
```

![](index_files/figure-html/plot_masks-2.png)<!-- -->

```r
ortho_diff( moco_avg_img, roi = maskImage, pred = bet_mask )
```

```
Warning in max(img, na.rm = TRUE): no non-missing arguments to max;
returning -Inf
```

```
Warning in min(img, na.rm = TRUE): no non-missing arguments to min;
returning Inf
```

```
Warning in max(img, na.rm = TRUE): no non-missing arguments to max;
returning -Inf
```

```
Warning in min(img, na.rm = TRUE): no non-missing arguments to min;
returning Inf
```

![](index_files/figure-html/plot_masks-3.png)<!-- -->

Here we will create a matrix of time by voxels.


```r
moco_avg_img[maskImage == 0] = 0
boldMatrix = img_ts_to_matrix(
    moco_img)
boldMatrix = t(boldMatrix)
boldMatrix = boldMatrix[ , maskImage == 1]
```


### Calculation of DVARS

With this `boldMatrix`, we can calculate a series of information.  For example, we can calculate DVARS based on the motion corrected data.  We can also compare the DVARS to the DVARS calculated from the non-realigned data.  

Here we show how you can still use `ANTsR::computeDVARS` to calculate DVARS.  The first element of `dvars` is the mean of the `my_dvars`.  By the definition of @power2012spurious, the first element of DVARS should be zero.

Here we will multiply the 3 first motion parameters (roll, pitch, yaw) by 50 to convert radians to millimeters by assuming a brain radius of 50 mm, as similar to @power2012spurious.  The next 3 parameters are in terms of millimeters (x, y, z). We will plot each of the parameters on the same scale to look at the motion for each scan.



```r
dvars = ANTsR::computeDVARS(boldMatrix)
dMatrix = apply(boldMatrix, 2, diff)
dMatrix = rbind(rep(0, ncol(dMatrix)), dMatrix)
my_dvars = sqrt(rowMeans(dMatrix^2))
head(cbind(dvars = dvars, my_dvars = my_dvars))
```

```
        dvars my_dvars
[1,] 18861.75     0.00
[2,] 18335.35 18335.35
[3,] 18855.69 18855.69
[4,] 17292.62 17292.62
[5,] 18590.46 18590.46
[6,] 18375.67 18375.67
```

```r
print(mean(my_dvars))
```

```
[1] 18861.75
```

Similarly, we can calculate the marginal framewise displacement (FD).  The rotation parameters are again in radians so we can translate these to millimeters based on a 50 mm radius of the head.


```r
mp = moco_params
mp[, 1:3] = mp[, 1:3] * 50
mp = apply(mp, 2, diff)
mp = rbind(rep(0, 6), mp)
mp = abs(mp)
fd = rowSums(mp)
```


```r
mp = moco_params
mp[, 1:3] = mp[, 1:3] * 50
r = range(mp)
plot(mp[,1], type = "l", xlab = "Scan Number", main = "Motion Parameters",
     ylab = "Displacement (mm)",
     ylim = r * 1.25, 
     lwd = 2,
     cex.main = 2,
     cex.lab = 1.5,
     cex.axis = 1.25)
for (i in 2:ncol(mp)) {
  lines(mp[, i], col = i)
}
```

![](index_files/figure-html/moco_run_plot-1.png)<!-- -->

```r
rm(list = "mp")
```

### Heatmap of the values

We can look at the full trajectory of each voxel over each scan.  We scaled the data (by column, which is voxel), which is somewhat equivalent to doing whole-brain z-score normalization of the fMRI.

We can find the index which has the highest mean value, which may indicate some motion artifact.


```r
library(RColorBrewer)
library(matrixStats)
rf <- colorRampPalette(rev(brewer.pal(11,'Spectral')))
r <- rf(32)
mat = scale(boldMatrix)
image(x = 1:nrow(mat), 
      y = 1:ncol(mat), 
      mat, useRaster = TRUE, 
      col = r,
      xlab = "Scan Number", ylab = "Voxel",
      main = paste0("Dimensions: ", 
                    dim(mat)[1], "×", dim(mat)[2]),
     cex.main = 2,
     cex.lab = 1.5,
     cex.axis = 1.25)
rmeans = rowMeans(mat)
bad_ind = which.max(rmeans)
print(bad_ind)
```

```
[1] 174
```

```r
abline(v = bad_ind)
```

![](index_files/figure-html/ts_heatmap-1.png)<!-- -->

```r
sds = rowSds(mat)
print(which.max(sds))
```

```
[1] 174
```

```r
rm(list = "mat")
```



```r
library(animation)
ani.options(autobrowse = FALSE)
gif_name = "bad_dimension.gif"
if (!file.exists(gif_name)) {
  arr = as.array(moco_img)
  pdim = pixdim(moco_img)
  saveGIF({
    for (i in seq(bad_ind - 1, bad_ind + 1)) {
      ortho2(arr[,,,i], pdim = pdim, text = i)
    }
  }, movie.name = gif_name)
}
```

![](bad_dimension.gif)

Much of the rest of the commands are again within R or already implemented in `ANTSR`.  Analyses in FSL commonly use independent components analysis (ICA) with the `melodic` command.  We will not cover that here and will cover it in a subsequent tutorial.



# Session Info


```r
devtools::session_info()
```

```
Session info -------------------------------------------------------------
```

```
 setting  value                       
 version  R version 3.3.2 (2016-10-31)
 system   x86_64, darwin13.4.0        
 ui       X11                         
 language (EN)                        
 collate  en_US.UTF-8                 
 tz       America/New_York            
 date     2017-02-06                  
```

```
Packages -----------------------------------------------------------------
```

```
 package     * version     date      
 abind         1.4-5       2016-07-21
 ANTsR         0.3.3       2017-01-23
 assertthat    0.1         2013-12-06
 backports     1.0.4       2016-10-24
 bitops        1.0-6       2013-08-17
 colorout    * 1.1-0       2015-04-20
 devtools      1.12.0.9000 2017-01-23
 digest        0.6.11      2017-01-03
 evaluate      0.10        2016-10-11
 extrantsr   * 2.11        2017-01-31
 fslr        * 2.7         2017-02-03
 hash          2.2.6       2013-02-21
 htmltools     0.3.6       2016-12-08
 igraph        1.0.1       2015-06-26
 iterators     1.0.8       2015-10-13
 knitr         1.15.1      2016-11-22
 lattice       0.20-34     2016-09-06
 magrittr      1.5         2014-11-22
 Matrix        1.2-7.1     2016-09-01
 matrixStats   0.51.0      2016-10-09
 memoise       1.0.0       2016-01-29
 mgcv          1.8-16      2016-11-07
 mmap          0.6-12      2013-08-28
 neurobase   * 1.11.1      2017-02-01
 neuroim       0.1.0       2016-09-27
 nlme          3.1-128     2016-05-10
 oro.nifti   * 0.7.2       2017-01-23
 pkgbuild      0.0.0.9000  2016-12-08
 pkgload       0.0.0.9000  2016-12-08
 plyr          1.8.4       2016-06-08
 R.matlab      3.6.1       2016-10-20
 R.methodsS3   1.7.1       2016-02-16
 R.oo          1.21.0      2016-11-01
 R.utils       2.5.0       2016-11-07
 Rcpp          0.12.8.2    2016-12-08
 rmarkdown     1.3         2017-01-03
 RNifti        0.3.0       2016-12-08
 rprojroot     1.1         2016-10-29
 stringi       1.1.2       2016-10-01
 stringr       1.1.0       2016-08-19
 WhiteStripe   2.0.4       2017-01-26
 withr         1.0.2       2016-06-20
 yaImpute      1.0-26      2015-07-20
 yaml          2.1.14      2016-11-12
 source                                   
 cran (@1.4-5)                            
 Github (stnava/ANTsR@460c09a)            
 CRAN (R 3.2.0)                           
 CRAN (R 3.3.0)                           
 CRAN (R 3.2.0)                           
 Github (jalvesaq/colorout@1539f1f)       
 Github (hadley/devtools@1ce84b0)         
 CRAN (R 3.3.2)                           
 CRAN (R 3.3.0)                           
 local                                    
 local                                    
 CRAN (R 3.2.0)                           
 Github (rstudio/htmltools@4fbf990)       
 CRAN (R 3.2.0)                           
 CRAN (R 3.2.0)                           
 cran (@1.15.1)                           
 CRAN (R 3.3.2)                           
 CRAN (R 3.2.0)                           
 CRAN (R 3.3.2)                           
 cran (@0.51.0)                           
 CRAN (R 3.2.3)                           
 CRAN (R 3.3.0)                           
 CRAN (R 3.3.0)                           
 local                                    
 local                                    
 CRAN (R 3.3.2)                           
 Github (neuroconductor/oro.nifti@c287559)
 Github (r-pkgs/pkgbuild@65eace0)         
 Github (r-pkgs/pkgload@def2b10)          
 CRAN (R 3.3.0)                           
 CRAN (R 3.3.0)                           
 CRAN (R 3.2.3)                           
 cran (@1.21.0)                           
 cran (@2.5.0)                            
 Github (RcppCore/Rcpp@8c7246e)           
 Github (rstudio/rmarkdown@3276760)       
 cran (@0.3.0)                            
 cran (@1.1)                              
 CRAN (R 3.3.0)                           
 cran (@1.1.0)                            
 local                                    
 CRAN (R 3.3.0)                           
 CRAN (R 3.2.0)                           
 CRAN (R 3.3.2)                           
```

# References
