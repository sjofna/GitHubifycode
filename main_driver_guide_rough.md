# User Guide for Main_driver.r file

<!--In terms of describing what a function is doing, you should only have to do this for custom functions.-->

<!--Make/ finish a table of contents with nested lists of sections/ subsections for easy navigation through the document.-->

- [User Guide for Main_driver.r file](#user-guide-for-main_driver.r-file)
   - [1. Initial Set Up](#1.-initial-set-up)
     - [1.1 Required Packages](#1.1-required-packages)
     - [1.2 Other Required Scripts to Run the Package](#1.2other-required-scripts-to-run-the-package)
     - [1.3 Set File Paths](#1.3-set-file-paths)
   - [2. Initial Variable Computation](#2.initial-variable-computation)
     - [2.1 Slope Calculation](#2.1-slope-calculation)
     - [2.2 Well Data](#2.2-well-data)
     - [2.3 Road Filling](#2.3-road-filling)
     - [2.4 Soil Depth](#2.4-soil-depth)
   - 3. [References](#references)

## 1. Initial Set Up

### 1.1 Required Packages
[GGplot2](https://ggplot2.tidyverse.org/), [dplyr](https://dplyr.tidyverse.org/), [terra](https://cran.r-project.org/web/packages/terra/index.html), [whitebox](https://whiteboxr.gishub.org/), [sf](https://r-spatial.github.io/sf/), [geojsonio](https://github.com/ropensci/geojsonio/), and [rgee](https://github.com/r-spatial/rgee) packages are needed to run the model.
Read up on documention by clicking the hyperlinks above.

<!-- Describe basic package functions? -->


**_Setting Up Rgee_**

1. Sign up for Earth Engine if not already.
2. Set up Earth Engine user with user's email (replace "user_email@email.com" with user's email).
3. Connect to Earth Engine drive via initializer function above.
   
>[!TIP]
>If credentials are stale run the required scripts code and then re run `ee_Initialize()`. 
<!-- > [!NOTE]
> Useful text
 etc doesnt work w collapsable lists.... -->


### 1.2 Other Required Scripts to Run the Package

Ensure that all additional scripts are downloaded and file path names have been replaced with personal file paths. These additional scripts define custom functions that will be used in this package.

>[!NOTE]
> The file path provided will not match the user's personal file path. If this is not changed, the code will not run correctly.


### 1.3 Set File Paths

This step is important as it will ensure that all inputs are coming from the correct folder and outputs will all be collected into one location for easy future access. Be sure to modify the input `main_in` and output `main_out` paths with personal paths. Temporary paths `tmp_path` can also be used _(reason??)_


---

## 2. Initial Variable Computation

This next few sections of code lays the groundwork for later functions. Calculated objects can be removed with `rm` and `gc()` can be used to clean memory. This can be helpful for _( efficiency? and computing power?)_ while working through the code. 


### 2.1 Slope Calculation

Slope is derived from the called in digital elevation model (DEM) using the `terra::terrain(dem, "slope")` function.


### 2.2 Well Data

Well data is needed for later calulation of soil depths. <!-- explain more here -->
Well data is cleaned to remove unwanted collumns before being converted to a vector file. A 8000m buffer is also created around the points to **_(???)_**. 

Additional plotting of well points is optional but can be helpful for viasualization and checking that the code worked.

Mean well depth is then extracted for **_(??)_**.


### 2.3 Road Filling

To conduct road filling, the vector road file is called in, a 15m buffer is created and the buffer file is converted to a raster layer. 15 meters is used to adequitely remove the road surface as well as the cuttslope (upslope embankment) and fill slope (downslope embankment). A new DEM is  created from the road raster and road cells are converted to NaN values using `ifel`. The `inpaint_nans` function is then applied reconstruct hillslope topography and detrending is done to create a more natural surface for computation. 

The `inpaint_nans` function was developed by John D'Errico ([Matlab documentation](https://www.mathworks.com/matlabcentral/fileexchange/4551-inpaint_nans).  This method was simialrly used by [Crema et al. (2020)](https://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/epdf/10.1002/esp.4739?domain=p2p_domain&token=GFE7JAFNEAQCUJK53MKK) and has been shown to reliably re-construct tropography from high resolution DTMs for eomorphometric analysis. This function solves a PDE (partial differential equation) across voids (in this case representing former roads) using edge information. The function for method ‘0’, a simple plate metaphor approach to solving the PDE, was translated into python for computational efficiency using the packages numpy (Harris et al., 2020) and scipy (Virtanen et al., 2020) and then applied as a function in R using the package reticulate (Ushey et al., 2025).

Digital image inpainting approaches can be grouped into three categories: (i) patch-based, (ii) sparse, and (iii) partial differential equations/variational methods. The inpainting algorithm we apply in this paper uses a PDE model to simulate a heat diffusion process. The accuracy of this inpainting technique is compared to the results of commonly used void-filling interpolation algorithms for a complex alpine test area. Void-filling performance, in particular, is evaluated quantitatively by means of statistical metrics (Reuter et al.2007) but also from a geomorphometric point of view.

The detrending function `detrend_surface` and `infel` add noise back to the inpainted surface. Where inpainting is applied the surface follows the general topographic trend, however, this surface is unnaturally smooth. To detrend the surface in these areas first the entire surface is smoothed using a 3 by 3 mean filter, the residual topography is derived by subtracting the smoothed surface from the initial DTM. Next, for each inpainted cell a large moving window (41 by 41) is used to randomly sample a surrounding residual value, this residual is then added to the smooth, inpainted, cell value.

These inpainting and detrending functions aim at developing a surface reconstruction framework that is capable of preserving topographic variability and morphometric properties (e.g., roughness, flow directions) for carrying out solid hydrogeomorphological elaborations. 


## 2.4 Soil Depth

Soil depth is calculated with the LRSC soil depth model translated from a [regolith](https://github.com/rogerlew/usgs-regolith), a fortran program, to r. This model allows for the estimation of soil mantle thickess in a digital landscape and assessment of debris flow based on parameters. 

The LCSC model uses the equation 
![equation](https://latex.codecogs.com/svg.latex?d_%7Br%7D%20%3D%20C_%7B0%7D&plus;C_%7B1%7D%5Cbigtriangledown%20z&plus;C_%7B2%7D%5Cleft%20%28%20S_%7Bc%7D-%5Cleft%20%7C%20%5Cbigtriangledown%20z%20%5Cright%20%7C%20%5Cright%20%29%2C%20where%20%5Cleft%20%28%20S_%7Bc%7D-%5Cleft%20%7C%20%5Cbigtriangledown%20z%20%5Cright%20%7C%20%5Cright%20%29%3E0) 
to calculate linear regression of slope and curvature. This method combines [Patton et al. (2018)](https://doi.org/10.1038/s41467-018-05743-y) with the linear slope model. The output d<sub>r</sub> is the regolith depth in meters. C<sub>0</sub> represents the background thickness soil thickness, or average soil depth within an area. C<sub>0</sub> is equal to the y-intercept in the linear regression. `C0=1.09` was calculated using the formula presented in the Patton et al. paper where average soil thickness \bar{h} can be derived from the general equation 

$${h = (\frac{\mathrm \bigtriangleup h}{\mathrm \bigtriangleup C }) C + \bar{h}}$$

>$(\frac{\mathrm \bigtriangleup h}{\mathrm \bigtriangleup C })$ is the slope of the relationship between mobile regolith thickness and curvature of the terrain. $C$ in this equation is terrain roughness, or rate of slope change in any given direction. When this value is equal to 0, $\bar{h}$ becomes $h$. _(still unsure where exactly 1.09 came from though??)_

C<sub>1</sub> represents the soil's sensitivity to slope curvature. Patton et al. described this as sensitiviity as $(\frac{\mathrm \bigtriangleup h})$. Using the values from the most similar site tested (Coos Bay) $C1 = (\frac{\mathrm \bigtriangleup h}) = 3.522 $.

C<sub>2</sub> represents the control of slope angle on soil thickness. Larger C<sub>2</sub> indicates more thinning due to increasing slope angle. This this is not found in the Patton et al. equation.

<!--
dr, regolith depth, m
C0, empirical constant (C0)
C1, empirical constant (C1)
C2, empirical constant (C2)
θ, slope angle, degrees
p, exponent of PSD polynomial or of upslope area (power)
κ, plan-view curvature of ground surface, ArcGIS convention (-100 * curvature)
A, upslope contributing area, m2
-->

<!--LRSC: Linear regression slope and curvature (combines Patton and others 2018 with linear slope)

# - C0 represents background thickness soil thickness, the y-intercept in the linear regression
## - in the Patton et al., 2018 paper this is the same as h bar in the generalized equation
## RG3D mean soil depth = 1.09
## from previous soil modeling paper in southern BC cited mean depth for Tulameen BC: 5m
###http://dx.doi.org/10.1016/j.geoderma.2016.07.012

# - C1 represents the sensitivity of soil thickness to slope curvature
## - in Patton et al., 2018 this is the same as delta h over delta c
### - based on the relationship between curvature sd and delta h over delta c as suggested in Patton et al., 2018  (y = -446.3x + 30.3)
### and using the values from the most similar site tested (Coos Bay) delta h over delta c (C1) = 3.522

# - C2 represents the control of slope angle on soil thickness, larger C2 = more thinning due to increasing slope angle
## - this is an add on and not found in the Patton et al., 2018 equation.-->

<!-- from the soild depth code 
    mag_del_z <- tan(slope_angle)
    sc <- tan(theta_c_rad)
    
     if (mag_del_z > sc) {
      depth <- 0
    } else {
      h1 <- C2 * (sc - mag_del_z)
      h2 <- C0 + C1 * lap
      depth <- h1 + h2
      depth <- max(min(depth, depth_max), depth_min)
