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
     - [2.5 Extract Soil Properties](#2.5-extract-soil-properties)
     - [2.6 Calculate Shear Strength](#2.6-calculate-shear-trength)
   - [References](#references)


<!-- Describe basic package functions? -->


## 1. Initial Set Up

### 1.1 Required Packages
[GGplot2](https://ggplot2.tidyverse.org/), [dplyr](https://dplyr.tidyverse.org/), [terra](https://cran.r-project.org/web/packages/terra/index.html), [whitebox](https://whiteboxr.gishub.org/), [sf](https://r-spatial.github.io/sf/), [geojsonio](https://github.com/ropensci/geojsonio/), and [rgee](https://github.com/r-spatial/rgee) packages are needed to run the model.
Read up on documention by clicking the hyperlinks above.


**_Setting Up RGEE_**

1. Sign up for Earth Engine if not already.
2. Set up Earth Engine user with user's email (replace "user_email@email.com" with user's email).
3. Connect to Earth Engine drive via initializer function above.
   
>[!TIP]
>If credentials are stale run the required scripts code and then re run `ee_Initialize()`. 


### 1.2 Other Required Scripts to Run the Package

Ensure that all additional scripts are downloaded and file path names have been replaced with personal file paths. These additional scripts define custom functions that will be used in this package.

>[!NOTE]
> The file path provided will not match the user's personal file path. If this is not changed, the code will not run correctly.


### 1.3 Set File Paths

This step is important as it will ensure that all inputs are coming from the correct folder and outputs will all be collected into one location for easy future access. Be sure to modify the input `main_in` and output `main_out` paths with personal paths. Temporary paths `tmp_path` can also be used _(reason??)_


---

## 2. Initial Variable Computation

This next few sections of code lays the groundwork for later functions. Calculated objects can be removed with `rm` and `gc()` can be used to clean memory. This can be helpful for minimizing computing power while working through the code. 


### 2.1 Slope Calculation

Slope is derived from the called in digital elevation model (DEM) using the `terra::terrain(dem, "slope")` function.


### 2.2 Well Data

Well data is needed for later calulation of soil depths. As well data includes distance to bedrock, this can be used as the maximum possible soil depth that can be calulated.
Well data is cleaned to remove unwanted collumns before being converted to a vector file. A 8000m buffer is also created around the points to **_(???)_**. 

Additional plotting of well points is optional but can be helpful for viasualization and checking that the code worked.


### 2.3 Road Filling

To conduct road filling, the vector road file is called in, a 15m buffer is created and the buffer file is converted to a raster layer. 15 meters is used to adequitely remove the road surface as well as the cuttslope (upslope embankment) and fill slope (downslope embankment). A new DEM is  created from the road raster and road cells are converted to NaN values using `ifel`. The `inpaint_nans` function is then applied reconstruct hillslope topography and detrending is done to create a more natural surface for computation. 

The `inpaint_nans` function was developed by [John D'Errico](https://www.mathworks.com/matlabcentral/fileexchange/4551-inpaint_nans).  This method was simialrly used by [Crema et al. (2020)](https://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/epdf/10.1002/esp.4739?domain=p2p_domain&token=GFE7JAFNEAQCUJK53MKK) and has been shown to reliably re-construct tropography from high resolution DTMs for eomorphometric analysis. This function solves a PDE (partial differential equation) across voids (in this case representing former roads) using edge information. The function for method ‘0’, a simple plate metaphor approach to solving the PDE, was translated into python for computational efficiency using the packages numpy (Harris et al., 2020) and scipy (Virtanen et al., 2020) and then applied as a function in R using the package reticulate (Ushey et al., 2025).

Digital image inpainting approaches can be grouped into three categories: 

   (i) patch-based

   (ii) sparse

   (iii) partial differential equations/variational methods 
   
The inpainting algorithm we apply in this paper uses a PDE model to simulate a heat diffusion process. The accuracy of this inpainting technique is compared to the results of commonly used void-filling interpolation algorithms for a complex alpine test area. Void-filling performance, in particular, is evaluated quantitatively by means of statistical metrics (Reuter et al.2007) but also from a geomorphometric point of view.

The detrending function `detrend_surface` and `infel` add noise back to the inpainted surface. Where inpainting is applied the surface follows the general topographic trend, however, this surface is unnaturally smooth. To detrend the surface in these areas first the entire surface is smoothed using a 3 by 3 mean filter, the residual topography is derived by subtracting the smoothed surface from the initial DTM. Next, for each inpainted cell a large moving window (41 by 41) is used to randomly sample a surrounding residual value, this residual is then added to the smooth, inpainted, cell value.

These inpainting and detrending functions aim at developing a surface reconstruction framework that is capable of preserving topographic variability and morphometric properties (e.g., roughness, flow directions) for carrying out solid hydrogeomorphological elaborations. 


## 2.4 Soil Depth

### 2.4.1 Calculate Soil Depth from Parameters
Soil depth is calculated with the LRSC soil depth model translated from a [regolith](https://github.com/rogerlew/usgs-regolith), a fortran program, to r. This model allows for the estimation of soil mantle thickess in a digital landscape and assessment of debris flow based on parameters. 

The LCSC model uses the equation 

$$\mathrm d_r = C_0 + C_1 \bigtriangledown z + C_2 (S_c - |\bigtriangledown z|),       where      (S_c - |\bigtriangledown z|) > 0$$

<!-- ![equation](https://latex.codecogs.com/svg.latex?d_%7Br%7D%20%3D%20C_%7B0%7D&plus;C_%7B1%7D%5Cbigtriangledown%20z&plus;C_%7B2%7D%5Cleft%20%28%20S_%7Bc%7D-%5Cleft%20%7C%20%5Cbigtriangledown%20z%20%5Cright%20%7C%20%5Cright%20%29%2C%20where%20%5Cleft%20%28%20S_%7Bc%7D-%5Cleft%20%7C%20%5Cbigtriangledown%20z%20%5Cright%20%7C%20%5Cright%20%29%3E0) -->
to calculate linear regression of slope and curvature. This method combines [Patton et al. (2018)](https://doi.org/10.1038/s41467-018-05743-y) with the linear slope model. The output d<sub>r</sub> is the regolith depth in meters. In the main code this function looks like:

```
soil_depth <- lrsc_depth_model(
  dem = dem,
  ca = ca,
  c0 = 1.09,
  c1 = 3.522,
  c2 = 0.5,
  theta_c = 55,
  depth_max = 10,
  depth_min = 0,
  chan_thresh = 2000000,
  smooth_topo = FALSE, # no smoothing for road adjustment
  smooth_soil = FALSE
)
```

The parameters in this case were based the [model developer's](#https://code.usgs.gov/ghsc/lhp/soil_depth) recommened values for the most similar climate, geology and vegetation to the area. This values will need to be altered to fit the location being calculated. 

C<sub>0</sub> represents the background thickness soil thickness, or average soil depth within an area. This value is equal to the y-intercept in the linear regression. C<sub>0</sub> was calculated using the formula presented in the Patton et al. paper where average soil thickness \bar{h} can be derived from the general equation 

$${h = (\frac{\mathrm \bigtriangleup h}{\mathrm \bigtriangleup C }) C + \bar{h}}$$

>$(\frac{\mathrm \bigtriangleup h}{\mathrm \bigtriangleup C })$ is the slope of the relationship between mobile regolith thickness and curvature of the terrain. $C$ in this equation is terrain roughness, or rate of slope change in any given direction. When this value is equal to 0, $\bar{h}$ becomes $h$.

C<sub>1</sub> represents the soil's sensitivity to slope curvature. Patton et al. described this as sensitiviity as $(\frac{\mathrm \bigtriangleup h}{\mathrm \bigtriangleup C })$. <!--Using the values from the most similar site tested (Coos Bay) $C_1 = (\frac{\mathrm \bigtriangleup h}{\mathrm \bigtriangleup C }) = 3.522$.-->

C<sub>2</sub> represents the control of slope angle on soil thickness. Larger C<sub>2</sub> indicates more thinning due to increasing slope angle. This this is not found in the Patton et al. equation. C<sub>2</sub> can be derived from the equation $h_1 = C_2 \times ( sc - tan(slope\theta)$. $(sc)$ is calculated from the tangent of `theta_c` in radians.

For soil depth compuation `sca` (Specific Catchment Area) is additionally converted to `ca` (Catchment Area) to account for the potential upslope soil erosion and depsoition due to flow dynamics. 
<!--SCA and CA are Specific Catchment Area and Specific Catchment, respectively. They are both variables used to represent drainage. CA is, for any given cell in a DEM, the total area of all upslope cells that would in theory be draining into it. SCA, for any given cell in a dem, provides an idea of the discharge, or potential concentration of flow, per unit flow width, due to upslope cells. Basically, SCA accounts for the fact that in a DEM each pixel covers some amount of area and is not just an infinitely small point. This allows us to consider the fact that overland flow would be diffused across the width of a cell (water spread out) and not just stacked in some infinitely thin line of flow that spans the length of the cell. This gives a better idea of the erosive capability of water flowing over any given point within a DEM cell. This is all to say that SCA is CA divided by the width of a pixel, so therefore CA is SCA times the width of a pixel. The reason I calculate SCA first then convert to CA is because SCA is more commonly used and there are better algorithms out there for calculating it. Anyway, SCA in particular can be a confusing concept but it is widely used so you don't need to go into too much detail about it-->


### 2.4.2 Adjusting Soil Depth Around Roads

Using the filled DEM created with the mentioned inpainting in section 2.3, a layer is created to capture the difference in surface height of the original DEM and the Filled DEM. This allows for a normalization of soil depth around the roads when `dem_difference` is subtracted from `soil_depth`. Negative values are set to zero using the `infel` function to remove unrealistic values from the layer. Maximum soil depth is extracted from the original `soil_depth` layer. Soil depth around roads is then clipped to the maximum soil depth using the code `soil_depth_adj <- ifel(soil_depth_adj > soil_depth_max, soil_depth_max, soil_depth_adj)`. This creates a layer where if at any point the `soil_depth_adj` is less than `soil_depth_max`, then the value from the `soil_depth_adj` will be (preferred/ chosen) in the new layer created. This new layer then represents the realistic soil depth in areas both around roads and areas without roads combined. Gaussian smoothing is applied to new soil depth layer to remove noise and road artifacts. The function `soil_depth <- soil_depth_smth` resets the object  `soil_depth` as the corrected soil depth instead of the previous result calculated solely from inpainted and detrended DEM.


## 2.5 Extract Soil Properties

The function `get_soilgrids` is adapted from [Giulio Genova's code](#https://git.wur.nl/isric/soilgrids/soilgrids.notebooks/-/commit/23fe857b81fea0149526fbdee2115d1480b1568c) to access and download the desired [SoilGrids](#https://soilgrids.org) layer. 

This function is used to extract sand and clay percentages from layer depths of 0-100m. These values are then fixes for soil classifcation using `fix-soil`.  This function converts the original units from $g/100g$ to a fraction. This conversion also allows silt to be calculated from with the equation $( 1 - (\mathrm slope + clay )). This ensures that all three values add up to one (100% of the soil) and is an easier method than extracting the silt value with the `get_soilgrids` function.

Soil texture class can optionally be computed with the `soil_texture` function. This function classifies the computed amount of sand and clay from the previous function `get_soilgrids` and compares it with the USDA classification. The function is based on [Hoffmann (2026)](#https://www.mathworks.com/matlabcentral/fileexchange/45468-soil_classification-sand-clay-t-varargin) and [Mathews (2014)](#https://code.usgs.gov/ghsc/lhp/regiongrow3d/-/blob/main/lib/functions/soil_classification_NM.m?ref_type=heads) soil classification functions. The function defines different silt and clay tresholds within twelve different soil types as seen in the image below. 

![Soil Classifcation Triangle (Hoffmann, 2026)](https://www.mathworks.com/matlabcentral/mlc-downloads/downloads/submissions/45468/versions/2/screenshot.png)


<img width="1067" height="983" alt="image" src="https://github.com/user-attachments/assets/9309a05b-dc2a-470f-b86a-d607cff53a60" />

_Soil Classifcation Triangle (Corral-Pazos-de-Provenset al., 2018)_


## 2.6 Calculate Shear Strength

### 2.6.1 Calculating Sand Subfractions

To calculate shear strength parameters, sand sub-fractractions must be calculated first. Very fine sand, fine sand and coarse sand sub fractions are extracted based on methods proposed by [Corral-Pazos-de-Provenset al. (2018)](#http://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/10.1002/ldr.3121). The function `get_vfs` utilizes the RUSLE2 formula, ESDAC method and the Shirazi–Boersma theory to calculate very fine sand based on the soil classification computed above. 

<img width="1486" height="982" alt="image" src="https://github.com/user-attachments/assets/ff238c5b-c5ba-4716-859c-f5800fc57819" />

Models used to estimate very fine sand fraction (Corral-Pazos-de-Provenset al., 2018)

<img width="1073" height="981" alt="image" src="https://github.com/user-attachments/assets/5495af72-588e-4d6d-9038-0aab09f9f94a" />

Model preference based on soil classification (Corral-Pazos-de-Provenset al., 2018) 

Based on which classification is best represented by each model, the corresponding model is chosen to calculate the fraction of very fine sand. 

> The RUSLE2 model formula uses the formula $[(0.74 -0.62 \times \mathrm sand) \times \mathrm sand ]$
>
> The ESDAC model uses the formula $( \frac{1}{5} \times \mathrm sand )$
>
> The Shirazi–Boersma model uses the function $[\Phi (0.698810 + 0.812098 \times \Phi^{-1}( 1 - \mathrm sand)) - 1 + \mathrm sand]$


Fine sand and coarse sand are calculated based on the research done by [Panagos et al. (20140](#https://www.sciencedirect.com/science/article/pii/S0048969714001727?via%3Dihub). This function `get_fs` and `get_cs` are the same as the ESDAC model from the function above which indicates that fine sand and coarse sand can be represented with the equation $( \frac{1}{5} \times \mathrm sand )$ as they and very fine sand represent one of five equal subcategories of sand. 


### 2.6.2 Calculating Shear Strength Parameters

Based on soil parameters calculated in sections indicated in 2.5 and 2.6.1, internal fraction and cohesion can be calulated with the functions `int_friction` and `unsat_cohesion`, respectively. 

There are two options of methods that can be used in the function `int_friction`. 
When `method = "subfraction"`, the model is based on a study by [Khaboushan et al. (2018)](#https://doi.org/10.1016/j.still.2018.07.006) and utilizes fine sand and very fine sand subfractions to determine the angle of internal friction. When using this method the friction angle (FA) $= 1.40 + 0.0001 \times (\mathrm fineSand^2) + 0.0001 * (\mathrm very Fine Sand^2)$. 
<!-- when use which method?? -->
<!-- also where is the GMD equation??? wanna make sure its the right source ig... idk im just confused @ teh equation-->
The GMD is based off of the methods proposed by [Luvai et al., (2022)](#https://doi-org.ezproxy.library.uvic.ca/10.1155/2022/2122554) for calulating the angle of internal friction. With this method FA becomes $( 1.43 + 1.23 \times \mathrm GMD )$ with GMD being $(\mathrm sand * log10(D_{sand}) + silt * log10(D_{silt}) + clay * log10(D_{clay}) )$. D represents the dominant grain size **_(d<sub>50</sub>??)_** of the sand, silt and clay. <!-- ?? -->

Cohesion is calculated as a function of the subfractions of clay, coarse sand and very fine sand based on the study done by [Khaboushan et al. (2018)](#https://doi.org/10.1016/j.still.2018.07.006). The to calculate unsaturated cohesion of the soil, the function `unsat_cohesion` $= ( -0.75 + 2.07 \times \mathrm clay^{0.5} - 5.87 \times \mathrm log10 (\mathrm coarse sand) - 0.035 \times \mathrm very fine sand^2)$. If this equation spits out a negative value the function will normalize that to zero.


## 2.7 Hydrology

Cation Exchange Capacity (CEC) and pH are extracted from soil grid data accessed eariler with the `get_soilgrids` function.

### 2.7.1 Calculating K<sub>sat</sub>
To calculate K<sub>sat</sub> (saturated hydraulic conductivity) the function `transmissivity` is utilized. This function allows two methods; rosetta which assign K<sub>sat</sub> based on [USDA texture codes *(?)*](#https://www.ars.usda.gov/pacific-west-area/riverside-ca/agricultural-water-efficiency-and-salinity-research-unit/docs/model/rosetta-class-average-hydraulic-parameters/) and EU method based off of Future Water report by [Simons et al. (2020)](#https://www.futurewater.nl/wp-content/uploads/2020/10/HiHydroSoil-v2.0-High-Resolution-Soil-Maps-of-Global-Hydraulic-Properties.pdf) and a study by [Tóth et al. (2014)](#https://doi.org/10.1111/ejss.12192).
The EU method utilizes the equation $( 0.40220 + 0.26122 \times \mathrm pH + 0.44565 \times \mathrm TS_{value} - 0.02329 \times \mathrm clay - 0.01265 \times \mathrm silt - 0.01038 \times \mathrm cec )$ to calulate $\mathrm log_{10}(k_{sat})$ where $TS_{value}$ is the distinction between subsoil and topsoil ([Simons et al., 2020)](#https://www.futurewater.nl/wp-content/uploads/2020/10/HiHydroSoil-v2.0-High-Resolution-Soil-Maps-of-Global-Hydraulic-Properties.pdf)). As K<sub>sat</sub> is logged and in the wrong units it is further derived using the equation $(10^{log_{10}(k_{sat})}) \div 100 \div 24)$.

Transimissivity is then calulated by multiplying K<sub>sat</sub> by soil depth extracted as `soil_depth` in section 2.4. <!-- function needs to be fixed so skip expanation for now? -->


> [!TIP]
> The code will prompt the user to clean menory using `rm` and `gc` functions. Do this to keep memory clear and computing power minimal.


### 2.7.2 Calibrating K<sub>sat</sub>
 <!-- work in progress code -->

This function aims to calibrate the calculated K<sub>sat</sub> for structural influence using the Modis NPP based on findings by [Fan et al. (2022)](#https://doi-org.ezproxy.library.uvic.ca/10.1029/2022GL100389) and [Bonetti et al., 2021](#https://doi.org/10.1038/s43247-021-00180-0). This method utilizes RGEE and requires Earth engine to be set up properly to function.    
