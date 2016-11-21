# Getting Data from the Human Connectome Project (HCP)
John Muschelli  
`r Sys.Date()`  



# Human Connectome Project (HCP)
The [Human Connectome Project](https://www.humanconnectome.org/) (HCP) is a consortium of sites whose goal is to map "human brain circuitry in a target number of 1200 healthy adults using cutting-edge methods of noninvasive neuroimaging" ([https://www.humanconnectome.org/](https://www.humanconnectome.org/)).  It includes a large cohort of individuals with a vast amount of neuroimaging data ranging from structural magnetic resonance imaging (MRI), functional MRI -- both during tasks and resting-state-- and diffusion tensor imaging (DTI), from multiple sites. 

# Getting Access to the Data

The data is available to those that agree to the license.  Users can either pay to get hard drives of the data sent to them, named "Connectome In A Box", or access the data online.  The data can be obtained through the database at [http://db.humanconnectome.org](http://db.humanconnectome.org).  Data can be downloaded from the website directly in a browser or through an Amazon Simple Storage Solution (S3) bucket.  We will focus on accessing the data from S3.
 
## Getting an Access/API Key

Once logged into [http://db.humanconnectome.org](http://db.humanconnectome.org) and the terms are accepted, the user must enable Amazon S3 access for their Amazon account.  The user will then be provided an access key identifier (ID), which is required to authenticate a user to Amazon as well as a secret key.  These access and secret keys are necessary for the [hcp package](https://github.com/muschellij2/hcp), and will be referred to as access keys or API (application program interface) keys.

# Installing the hcp package

We will install the hcp package using the Neuroconductor installer:

```r
source("http://neuroconductor.org/neurocLite.R")
neuro_install("hcp", release = "stable")
```

## Setting the API key

In the hcp package, `set_aws_api_key` will set the AWS access keys:


```r
set_aws_api_key(access_key = "ACCESS_KEY", secret_key = "SECRET_KEY")
```
or these can be stored in `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` [environment variables](https://stat.ethz.ch/R-manual/R-devel/library/base/html/EnvVar.html), respectively.

Once these are set, the functions of hcp are ready to use.  To test that the API keys are set correctly, one can run `bucketlist`:


```r
hcp::bucketlist()
```


```
                  Name             CreationDate
1       hcp-openaccess 2014-05-15T18:56:50.000Z
2  hcp-openaccess-logs 2014-05-15T18:57:07.000Z
3 hcp-openaccess-trail 2016-06-02T20:22:11.000Z
```

We see that `hcp-openaccess` is a bucket that we have access to, and therefore have access to the data.


## Getting Data: Downloading a Directory of Data

In the hcp package, there is a data set indicating the scans read for each subject, named `hcp_900_scanning_info`.  We can subset those subjects that have diffusion tensor imaging:


```r
ids_with_dwi = hcp_900_scanning_info %>% 
  filter(scan_type %in% "dMRI") %>% 
  select(id) %>% 
  unique
head(ids_with_dwi)
```

```
# A tibble: 6 × 1
      id
   <chr>
1 100307
2 100408
3 101006
4 101107
5 101309
6 101410
```

Let us download the complete directory of diffusion data using `download_hcp_dir`:

```r
r = download_hcp_dir("HCP/100307/T1w/Diffusion")
print(basename(r$output_files))
```

```
[1] "bvals"                   "bvecs"                  
[3] "data.nii.gz"             "grad_dev.nii.gz"        
[5] "nodif_brain_mask.nii.gz"
```
This diffusion data is the data that can be used to create summaries such as fractional anisotropy and mean diffusivity.  

If we create a new column with all the directories, we can iterate over these to download all the diffusion data for these subjects from the HCP database.

```r
ids_with_dwi = ids_with_dwi %>% 
  mutate(id_dir = paste0("HCP/", id, "/T1w/Diffusion"))
```

## Getting Data: Downloading a Single File
We can also download a single file using `download_hcp_file`.  Here we will simply download the `bvals` file:


```r
ret = download_hcp_file("HCP/100307/T1w/Diffusion/bvals")
```

```

  |                                                                       
  |                                                                 |   0%
  |                                                                       
  |=================================================================| 100%
```
