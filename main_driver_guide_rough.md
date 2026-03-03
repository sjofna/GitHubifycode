# User Guide for Main_driver.r file

<!--In terms of describing what a function is doing, you should only have to do this for custom functions.-->

<!--Make/ finish a table of contents with nested lists of sections/ subsections for easy navigation through the document.-->

1. [User Guide for Main_driver.r file](#user-guide-for-main_driver.r-file)
   - 1.[Initial Set Up](#initial-set-up)
     - 1.1[Required Packages](#required-packages)
   - 2. References

## Initial Set Up

### Required Packages
[GGplot2](https://ggplot2.tidyverse.org/), [dplyr](https://dplyr.tidyverse.org/), [terra](https://cran.r-project.org/web/packages/terra/index.html), [whitebox](https://whiteboxr.gishub.org/), [sf](https://r-spatial.github.io/sf/), [geojsonio](https://github.com/ropensci/geojsonio/), and [rgee](https://github.com/r-spatial/rgee) packages are needed to run the model.
Read up on documention by clicking the hyperlinks above.

To install & load:
```
packages_to_install <- c("ggplot2", "dplyr", "terra", "whitebox", "sf", "geojsonio", "rgee")

for (pkg in packages_to_install) {
  if (!require(pkg, character.only = TRUE)) {
    install.packages(pkg, dependencies = TRUE)
  }
}

library(whitebox)
library(ggplot2)
library(dplyr)
library(terra)
library(sf)
library(dplyr)
library(geojsonio)
library(rgee)
```
<!-- Describe basic package functions? -->

**_To Set Up Rgee_**
```
ee_username <- "user_email@email.com"

ee_Initialize(user = ee_username, drive = TRUE)
```
1. Sign up for Earth Engine if not already.
2. Set up Earth Engine user with user's email (replace "user_email@email.com" with user's email).
3. Connect to Earth Engine drive via initializer function above.
   
>[!TIP]
>If credentials are stale run the required scripts code and then re run `ee_Initialize()`. 
<!-- > [!NOTE]
> Useful text
 etc doesnt work w collapsable lists.... -->


### Other Required Scripts to Run the Package

Ensure that all additional scripts are downloaded and file path names have been replaced with personal file paths. These additional scripts define custom functions that will be used in this package.

>[!NOTE]
> The file path provided will not match the user's personal file path. If this is not changed, the code will not run correctly.


### Set File Paths for Inputs and Outputs

This is important as it will ensure that all inputs are coming from the correct folder and outputs will all be collected into one location for easy future access.


---

## Call in Data and Compute Initial Variables

This next few sections of code lays the groundwork for later functions. Calculated objects can be removed with `rm` and `gc()` can be used to clean memory. This can be helpful for _( efficiency? and computing power?)_ while working through the code. 


### Slope Calculation

Slope is derived from the called in digital elevation model (DEM) using the `terra::terrain(dem, "slope")` function.


### Well Data

Well data is needed for later calulation of soil depths. <!-- explain more here -->
Well data is cleaned to remove unwanted collumns before being converted to a vector file. A 8000m buffer is also created around the points to **_(???)_**. 

Additional plotting of well points is optional but can be helpful for viasualization and checking that the code worked.

Mean well depth is then extracted for **_(??)_**.


### Road Filling

To conduct road filling, the vector road file is called in, a 15m buffer is created and the buffer file is converted to a raster layer. A new DEM is then created from this road raster where road cells are converted to NaN values using `ifel`. The `inpaint_nans` function then .
A mask is then added 

