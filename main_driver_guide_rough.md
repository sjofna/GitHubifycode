# User Guide for Main_driver.r file

<!--In terms of describing what a function is doing, you should only have to do this for custom functions.-->

<!--Make/ finish a table of contents with nested lists of sections/ subsections for easy navigation through the document.-->

1. [User Guide for Main_driver.r file](#user-guide-for-main_driver.r-file)
   - 1.[Initial Set Up](#initial-set-up)
     - 1.1[Required Packages](#required-packages)

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
Ensure that all additional scripts are downloaded and file path names have been replaced with personal file paths. 

>[!NOTE]
> The file path provided will not match the user's personal file path. If this is not changed, the code will not run correctly.

### Set File Paths for Inputs and Outputs

This is important as it will ensure that all inputs are coming from the correct folder and outputs will all be collected into one location for easy future access.


---

## Call in Data and Compute Initial Variables
These next few sections of code calculate a few different things. 

- Slope is derived from the digital elevation model (DEM) that is called in.
- Well points are cleaned, and converted to a vector file. A 8000m buffer is also created around the points to **_(???)_**
- Plotting of well points is optional but can be helpful for viasualization and checking that the code worked.
- Parameters are extracted from well data such as mean, maximum and median values. _(only mean??)_
- 

  
Slope is derived from the digital elevation model (DEM) that is called in.
```
dem <- rast("E:/MSc/LiDAR/RobsonValley/DEM/RobsonValleyDEM.tif")
dem_filled <- rast(file.path(main_in, "LiDAR/Filled/DEM_filled_detrend_test_CV.tif")) # filled (optional)

target <- target_rast(dem = dem)

slope <- terra::terrain(dem, "slope")
```

 
Well points are cleaned, and converted to a vector file. A 8000m buffer is also created around the points to **_(???)_**
```
well_data <- well_data %>%
  select(well_tag_number, latitude_Decdeg, longitude_Decdeg, utm_zone_code,
         utm_northing, utm_easting, bedrock_depth_ft.bgl) %>%
  filter(!is.na(bedrock_depth_ft.bgl), bedrock_depth_ft.bgl >= 0) %>%
  mutate(bedrock_depth_m = bedrock_depth_ft.bgl * 0.3048) # convert from feet to meters

well_points <- vect(well_data, geom = c("longitude_Decdeg", "latitude_Decdeg"), crs = "+proj=longlat +datum=WGS84")
well_points <- project(well_points, target$crs)

well_points <- well_depth(well_points = well_points,
                          target = target, 
                          buff = 8000)
```


Plotting of well points is optional but can be helpful for viasualization and checking that the code worked.
```
well_data <- as.data.frame(well_points)
ggplot(well_data, aes(bedrock_depth_m)) +
  geom_density(fill = "skyblue", alpha = 0.6) + 
  labs(
    title = "Depth to Bedrock Distribution",
    x = "Depth to Bedrock (m bgl)", 
    y = "Density"
  ) + 
  theme_minimal()
```
<!-- insert plot created from code?-->


Parameters are extracted from well data such as mean, maximum and median values. _(only mean??)_
```
mean_well_depth <- round(mean(well_data$bedrock_depth_m), 3)
```

>[!TIP]
> Objects can be removed with the `rm` function to clean memory. This can be helpful for _( efficiency?)_


To conduct road filling, road file is called in, 15m buffer is created and file is converted to a raster layer. A new DEM is then created from this road raster where road cells are converted to NaN values which are then used to ??? `inpaint_nans` [(inpaint_nans documentation)](https://www.mathworks.com/matlabcentral/fileexchange/4551-inpaint_nans).
A mask is then added 

