Read, filter and transform spectra and metadata
================
Philipp Baumann // <philipp.baumann@usys.ethz.ch>
July 25, 2018

------------------------------------------------------------------------

Topics and goals of this section
================================

-   You will learn how to use different R base data structures and basic operations such as subsetting to explore and transform spectral data.

------------------------------------------------------------------------

Reading spectra from OPUS spectrometer files: prerequisites
===========================================================

Spectroscopy modeling requires that we first organize our spectra well. In particular, a proper and reproducible data management of spectral data, metadata, and data from reference chemical analyses is key for all the subsequent data processing and modeling workflow.

The Sustainable Agroecosystems group at ETH relies on Diffuse Reflectance Fourier Transform (DRIFT) infrared spectrometers manufactured by the company *Bruker* (see Figure ). The manufacturer relies on a proprietary binary format called *OPUS* to store an extensive amount of data that includes different types of intermediary spectra. For each sample that was measured a single *OPUS* file is produced.

<img src="figures/alpha_eth.jpg" alt="Bruker ALPHA mid-IR spectrometer (diffuse reflectance Fourier transform infrared) of the Sustainable Agroecosystems group at ETH Zürich with a sample cup filled with soil. " width="200" />

Reading spectrometer data into the R environment
================================================

First, we load the set of packages of `tidyverse` (see [**here**](http://tidyverse.org/) for details) and the `simplerspec` package (see [**here**](https://github.com/philipp-baumann/simplerspec/)). Simplerspec contains a universal file reader that allows to read selected parameters (e.g. instrument, optic and acquisition parameters) and all types of spectra from a single *OPUS* binary file or a list of files.

``` r
# Load collection of packages that work together seamlessly for efficient
# analysis workflows
library("tidyverse")
```

    ## ── Attaching packages ──────────────────────────────────────────── tidyverse 1.2.1 ──

    ## ✔ ggplot2 3.0.0     ✔ purrr   0.2.5
    ## ✔ tibble  1.4.2     ✔ dplyr   0.7.6
    ## ✔ tidyr   0.8.1     ✔ stringr 1.3.1
    ## ✔ readr   1.1.1     ✔ forcats 0.3.0

    ## ── Conflicts ─────────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

``` r
# Package that facilitates spectral data handling, processing and modeling
library("simplerspec")
```

    ## Loading required package: foreach

    ## 
    ## Attaching package: 'foreach'

    ## The following objects are masked from 'package:purrr':
    ## 
    ##     accumulate, when

I recommended that you set up a self-contained directory where all R scripts, data (spectra and chemical reference data), models, outputs (figures and text files of data and model summaries), and predictions live. Further, you can use the folder structure depicted in figure to organize your spectroscopy-related research projects.

<img src="figures/project_folder_structure.png" alt="Recommended directory structure for spectroscopy modeling projects " style="width:70.0%" />

When you have spectra of various different spectral data sets or that cover separate experiments and/or different locations and times, you might prefer to organize your spectra as sub-folders within `data/spectra`. This hands-on will be based spectral data that have been used to build and evaluate the YAMSYS spectroscopy reference models. Besides these reference spectra measured witha Bruker ALPHA mid-IR spectrometer at the Sustainable Agroecosystems group at ETH Zürich, there are other spectra that have been acquired to test different questions such as spectrometer cross-comparisons. Therefore, other comparison spectra are in separate paths, e.g. `data/spectra/soilspec_eth_bin`.

In Figure you can see file explorer screenshot showing *OPUS* files of three replicate scans for each of the first three reference soil samples. *OPUS* have the extension `.n` where `n` represents an integer of repeated sample measurements starting from 0.

<img src="figures/spectra_files_to_read.png" alt="Screenshot showing replicate scans of first three samples reading example. " width="400" />

We aim to read all the reference spectra contained within this folder. First, we get the full path names of the file names, which we subsequently assign to the object `files`:

``` r
# Extract data from OPUS binary files; list of file paths
files <- list.files("data/spectra", full.names = TRUE)
```

Note that you need to set the `full.names` argument to `TRUE` (default is `FALSE` to get the path of all *OPUS* spectra files contained within the target directory, otherwise R will not be able to find the files when using the universal `simplerspec` *OPUS* reader.

You can compactly display the internal structure of the `files` object:

``` r
str(files)
```

    ##  chr [1:284] "data/spectra/BF_lo_01_soil_cal.0" ...

The object `files` has the data structure *atomic vector*. An *atomic vectors* have six possible basic (*atomic*) vector types. These are *logical*, *integer*, *real*, *complex*, *string* (or *character*) and *raw*. Vector types can be returned by the R base function `typeof(x)`, which returns the type or internal storage mode an object `x`. For the `files` object it is

``` r
# Check type of files object
typeof(files)
```

    ## [1] "character"

We get the length of the vector or the number of elements by

``` r
# How many files are listed to read? length of vector
length(files)
```

    ## [1] 284

Base R has subsetting operations that allow you to extract pieces of data structures you are interested in. One of the three base subsetting operators is `[`.

We subset the character vector `files` as follows:

``` r
# Use character subsetting to return the first element
# Subsetting can be seen as complement to str()
# (1) Subsetting with positive integers (position)
files[1:3]
```

    ## [1] "data/spectra/BF_lo_01_soil_cal.0" "data/spectra/BF_lo_01_soil_cal.1"
    ## [3] "data/spectra/BF_lo_01_soil_cal.2"

``` r
# (2) Subsetting with negative integers (remove values)
head(files[-c(1:3)], n = 5L) # show only first 5 values
```

    ## [1] "data/spectra/BF_lo_02_soil_cal.0" "data/spectra/BF_lo_02_soil_cal.1"
    ## [3] "data/spectra/BF_lo_02_soil_cal.2" "data/spectra/BF_lo_03_soil_cal.0"
    ## [5] "data/spectra/BF_lo_03_soil_cal.1"

``` r
# The first three elements of the character vector are removed
```

Spectral measurement data
-------------------------

Bruker FTIR spectrometers produce binary files in the OPUS format that can contain different types of spectra and many parameters such as instrument type and settings that were used at the time of data acquisition and internal processing (e.g. Fourier transform operations). Basically, the entire set of *Setup Measurement Parameters*, selected spectra, supplementary metadata such as the time of measurement are written into *OPUS* binary files. In contrast to simple text files that contain only plain text with a defined character encoding, binary files can contain any type of data represented as sequences of bytes (a single byte is sequence of 8 bits and 1 bit either represents 0 or 1).

Figure shows graphical representation from the *OPUS* viewer software to get familiarize with types of parameters *OPUS* files may contain.

<img src="figures/opus_instrument_parameters_crop.png" alt="Instrument parameters during sample measurement shown for an example YAMSYS soil reference spectroscopy sample. Spectra and parameters can be shown by the dialogue Window &gt; New Report Window within the OPUS viewer software. " style="width:65.0%" />

You can download the *OPUS viewer* software from [**this Bruker webpage**](https://www.bruker.com/products/infrared-near-infrared-and-raman-spectroscopy/opus-spectroscopy-software/downloads/opus-downloads.html) for free. However, Bruker only provides a Windows version and the free version is limited to visualize only final spectra. The remaining spectral blocks can be checked choosing the menu *Window* &gt; *New Report Window* and opening *OPUS* by the menu *File* &gt; *Load File*.

Session info
============

``` r
sessionInfo()
```

    ## R version 3.4.4 (2018-03-15)
    ## Platform: x86_64-pc-linux-gnu (64-bit)
    ## Running under: KDE neon User Edition 5.13
    ## 
    ## Matrix products: default
    ## BLAS: /usr/lib/libblas/libblas.so.3.6.0
    ## LAPACK: /usr/lib/lapack/liblapack.so.3.6.0
    ## 
    ## locale:
    ##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
    ##  [3] LC_TIME=de_CH.UTF-8        LC_COLLATE=en_US.UTF-8    
    ##  [5] LC_MONETARY=de_CH.UTF-8    LC_MESSAGES=en_US.UTF-8   
    ##  [7] LC_PAPER=de_CH.UTF-8       LC_NAME=C                 
    ##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
    ## [11] LC_MEASUREMENT=de_CH.UTF-8 LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ##  [1] simplerspec_0.1.0 foreach_1.4.4     forcats_0.3.0    
    ##  [4] stringr_1.3.1     dplyr_0.7.6       purrr_0.2.5      
    ##  [7] readr_1.1.1       tidyr_0.8.1       tibble_1.4.2     
    ## [10] ggplot2_3.0.0     tidyverse_1.2.1  
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] tidyselect_0.2.4  reshape2_1.4.3    haven_1.1.1      
    ##  [4] lattice_0.20-35   colorspace_1.3-2  htmltools_0.3.6  
    ##  [7] yaml_2.1.19       rlang_0.2.1       pillar_1.2.2     
    ## [10] foreign_0.8-70    glue_1.2.0        withr_2.1.2      
    ## [13] modelr_0.1.2      readxl_1.1.0      bindrcpp_0.2.2   
    ## [16] bindr_0.1.1       plyr_1.8.4        munsell_0.4.3    
    ## [19] gtable_0.2.0      cellranger_1.1.0  rvest_0.3.2      
    ## [22] codetools_0.2-15  psych_1.8.3.3     evaluate_0.10.1  
    ## [25] knitr_1.20        parallel_3.4.4    broom_0.4.4      
    ## [28] Rcpp_0.12.17      scales_0.5.0      backports_1.1.2  
    ## [31] jsonlite_1.5      mnormt_1.5-5      hms_0.4.2        
    ## [34] digest_0.6.15     stringi_1.2.2     grid_3.4.4       
    ## [37] rprojroot_1.3-2   cli_1.0.0         tools_3.4.4      
    ## [40] magrittr_1.5      lazyeval_0.2.1    crayon_1.3.4     
    ## [43] pkgconfig_2.0.1   data.table_1.11.4 xml2_1.2.0       
    ## [46] lubridate_1.7.4   assertthat_0.2.0  rmarkdown_1.9    
    ## [49] httr_1.3.1        rstudioapi_0.7    iterators_1.0.9  
    ## [52] R6_2.2.2          nlme_3.1-137      compiler_3.4.4
