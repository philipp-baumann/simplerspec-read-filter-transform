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

    ## ── Attaching packages ───────────────────────────────────────────── tidyverse 1.2.1 ──

    ## ✔ ggplot2 3.0.0     ✔ purrr   0.2.5
    ## ✔ tibble  1.4.2     ✔ dplyr   0.7.6
    ## ✔ tidyr   0.8.1     ✔ stringr 1.3.1
    ## ✔ readr   1.1.1     ✔ forcats 0.3.0

    ## ── Conflicts ──────────────────────────────────────────────── tidyverse_conflicts() ──
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

When you have spectra of various different spectral data sets or that cover separate experiments and/or different locations and times, you might prefer to organize your spectra as sub-folders within `data/spectra`. This hands-on will be based spectral data that have been used to build and evaluate the YAMSYS spectroscopy reference models. Besides these reference spectra measured with a Bruker ALPHA mid-IR spectrometer at the Sustainable Agroecosystems group at ETH Zürich, there are other spectra that have been acquired to test different questions such as spectrometer cross-comparisons. Therefore, YAMSYS reference spectra are contained within a separate path `data/spectra/soilspec_eth_bin`.

In Figure you can see file explorer screenshot showing *OPUS* files of three replicate scans for each of the first three reference soil samples. *OPUS* have the extension `.n` where `n` represents an integer of repeated sample measurements starting from 0.

<img src="figures/spectra_files_to_read.png" alt="Screenshot showing replicate scans of first three samples reading example. " width="400" />

We aim to read all the reference spectra contained within this folder. First, we get the full path names of the file names, which we subsequently assign to the object `files`:

``` r
# Extract data from OPUS binary files; list of file paths
files <- list.files("data/spectra/soilspec_eth_bin", full.names = TRUE)
```

Note that you need to set the `full.names` argument to `TRUE` (default is `FALSE` to get the path of all *OPUS* spectra files contained within the target directory, otherwise R will not be able to find the files when using the universal `simplerspec` *OPUS* reader.

You can compactly display the internal structure of the `files` object using \`str()̀.

``` r
str(files)
```

    ##  chr(0)

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

    ## [1] 0

Base R has subsetting operations that allow you to extract pieces of data structures you are interested in. One of the three base subsetting operators is `[`.

We subset the character vector `files` as follows:

``` r
# Use character subsetting to return the first element
# Subsetting can be seen as complement to str()
# (1) Subsetting with positive integers (position)
files[1:3]
```

    ## [1] NA NA NA

``` r
# (2) Subsetting with negative integers (remove values)
head(files[-c(1:3)], n = 5L) # show only first 5 values
```

    ## character(0)

``` r
# The first three elements of the character vector are removed
```

Bruker FTIR spectrometers produce binary files in the OPUS format that can contain different types of spectra and many parameters such as instrument type and settings that were used at the time of data acquisition and internal processing (e.g. Fourier transform operations). Basically, the entire set of *Setup Measurement Parameters*, selected spectra, supplementary metadata such as the time of measurement are written into *OPUS* binary files. In contrast to simple text files that contain only plain text with a defined character encoding, binary files can contain any type of data represented as sequences of bytes (a single byte is sequence of 8 bits and 1 bit either represents 0 or 1).

Figure shows graphical representation from the *OPUS* viewer software to get familiarize with types of parameters *OPUS* files may contain.

<img src="figures/opus_instrument_parameters_crop.png" alt="Instrument parameters during sample measurement shown for an example YAMSYS soil reference spectroscopy sample. Spectra and parameters can be shown by the dialogue Window &gt; New Report Window within the OPUS viewer software. " style="width:65.0%" />

You can download the *OPUS viewer* software from [**this Bruker webpage**](https://www.bruker.com/products/infrared-near-infrared-and-raman-spectroscopy/opus-spectroscopy-software/downloads/opus-downloads.html) for free. However, Bruker only provides a Windows version and the free version is limited to visualize only final spectra. The remaining spectral blocks can be checked choosing the menu *Window* &gt; *New Report Window* and opening *OPUS* by the menu *File* &gt; *Load File*.
