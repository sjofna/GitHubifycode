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
       - [2.6.1 Calculating Sand Subfractions](#2.6.1-calculating-sand-subfractions)
       - [2.6.2 Calculating Shear Strength Parameters](#2.6.2-calculating-shear-strength-parameters)
     - [2.7 Hydrology](#2.7-hydrology)
       - [2.7.1 Calculating K<sub>sat</sub>](#2.7.1-calculating-ksat)
       - [2.7.2 Calibrating K<sub>sat</sub>](#2.7.2-calibrating-ksat)
     - [2.8 Calculating Normal Recharge with PCIC Data](#2.8-calculating-normal-recharge-with-pcic-data)
     - [2.9 Soil Density](#2.9-soil-density)
     - [2.10 Root Cohesion](#2.10-root-cohesion)
     - [2.11 Wildfire Effects](#2.11-wildfire-effects)
       - [2.11.1 Modifier Layers](#2.11.1-modifier-layers)
       - [2.11.2 Uncertainty](#2.11.2-uncertainty)
   - [3. Landslide Probability](#3.-landslide-probability)
     - [3.1 Factor of Safety](#3.1-factor-of-safety)
     - [3.2 Computing Probability](#3.2-computing-probability)
     - [3.3 Optional Manual Options](#3.3-optional-manual-options)
   - [4. References](#references)


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

This step is important as it will ensure that all inputs are coming from the correct folder and outputs will all be collected into one location for easy future access. Be sure to modify the input `main_in` and output `main_out` paths with personal paths. Temporary paths `tmp_path` are set up as to not pollute the main output path with temporary files and allows for these files to be deleted easily as needed.


---

## 2. Initial Variable Computation

This next few sections of code lays the groundwork for later functions. Calculated objects can be removed with `rm` and `gc()` can be used to clean memory. This can be helpful for minimizing computing power while working through the code. 


### 2.1 Slope Calculation

Slope is the main topgraphic variable used in this package. It is derived from the digital elevation model (DEM) using the `terra::terrain(dem, "slope")` [function](#https://www.rdocumentation.org/packages/terra/versions/1.0-7/topics/slope). The output here is in degrees.


### 2.2 Well Data

Well data is needed for later calulation of soil depths. As well data includes distance to bedrock, this can be used as the maximum possible soil depth that can be calulated.
Well data is cleaned to remove unwanted collumns before being converted to a vector file. Well points within abuffer distance of the DEM create a signifcant subset of points for bedrock depth calculations. 

Additional plotting of well points is optional but can be helpful for viasualization and checking that well points were captured by the code.


### 2.3 Road Filling

To conduct road filling, the vector road file is called in, a 15m buffer is created and the buffer file is converted to a raster layer. 15 meters is used to adequitely remove the road surface as well as the cuttslope (upslope embankment) and fill slope (downslope embankment). A new DEM is  created from the road raster and road cells are converted to NaN values using `ifel`. The `inpaint_nans` function is then applied reconstruct hillslope topography and detrending is done to create a more natural surface for computation. 

The `inpaint_nans` function was developed by [John D'Errico](https://www.mathworks.com/matlabcentral/fileexchange/4551-inpaint_nans).  This method was simialrly used by [Crema et al. (2020)](https://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/epdf/10.1002/esp.4739?domain=p2p_domain&token=GFE7JAFNEAQCUJK53MKK) and has been shown to reliably re-construct tropography from high resolution DTMs for eomorphometric analysis. This function solves a PDE (partial differential equation) across voids (in this case representing former roads) using edge information. The function for method ‘0’, a simple plate metaphor approach to solving the PDE, was translated into python for computational efficiency using the packages numpy (Harris et al., 2020) and scipy (Virtanen et al., 2020) and then applied as a function in R using the package reticulate (Ushey et al., 2025).

<!--"Digital image inpainting approaches can be grouped into three categories: (i) patch-based, (ii) sparse and (iii) partial differential equations/variational methods 
The inpainting algorithm applied uses a PDE model to simulate a heat diffusion process. The accuracy of this inpainting technique is compared to the results of commonly used void-filling interpolation algorithms for a complex alpine test area. Void-filling performance, in particular, is evaluated quantitatively by means of statistical metrics (Reuter et al.2007) but also from a geomorphometric point of view.[(Crema et al., 2020)](https://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/epdf/10.1002/esp.4739?domain=p2p_domain&token=GFE7JAFNEAQCUJK53MKK)" -->

The detrending function `detrend_surface` and `infel` add noise back to the inpainted surface. Where inpainting is applied the surface follows the general topographic trend, however, this surface is unnaturally smooth. To detrend the surface in these areas first the entire surface is smoothed using a 3 by 3 mean filter, the residual topography is derived by subtracting the smoothed surface from the initial DTM. Next, for each inpainted cell a large moving window (41 by 41) is used to randomly sample a surrounding residual value, this residual is then added to the smooth, inpainted, cell value.

These inpainting and detrending functions aim at developing a surface reconstruction framework that is capable of preserving topographic variability and morphometric properties (e.g., roughness, flow directions) for carrying out solid hydrogeomorphological elaborations. 


### 2.4 Soil Depth

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


### 2.5 Extract Soil Properties

The function `get_soilgrids` is adapted from [Giulio Genova's code](#https://git.wur.nl/isric/soilgrids/soilgrids.notebooks/-/commit/23fe857b81fea0149526fbdee2115d1480b1568c) to access and download the desired [SoilGrids](#https://soilgrids.org) layer. 

This function is used to extract sand and clay percentages from layer depths of 0-100m. These values are then fixes for soil classifcation using `fix-soil`.  This function converts the original units from $g/100g$ to a fraction. This conversion also allows silt to be calculated from with the equation $( 1 - (\mathrm slope + clay )). This ensures that all three values add up to one (100% of the soil) and is an easier method than extracting the silt value with the `get_soilgrids` function.

Soil texture class can optionally be computed with the `soil_texture` function. This function classifies the computed amount of sand and clay from the previous function `get_soilgrids` and compares it with the USDA classification. The function is based on [Hoffmann (2026)](#https://www.mathworks.com/matlabcentral/fileexchange/45468-soil_classification-sand-clay-t-varargin) and [Mathews (2014)](#https://code.usgs.gov/ghsc/lhp/regiongrow3d/-/blob/main/lib/functions/soil_classification_NM.m?ref_type=heads) soil classification functions. The function defines different silt and clay tresholds within twelve different soil types as seen in the image below. 

![Soil Classifcation Triangle (Hoffmann, 2026)](https://www.mathworks.com/matlabcentral/mlc-downloads/downloads/submissions/45468/versions/2/screenshot.png)

_Soil Classifcation Triangle (Hoffmann, 2026)_


<img width="800" height="737" alt="image" src="https://github.com/user-attachments/assets/9309a05b-dc2a-470f-b86a-d607cff53a60" />

_Soil Classifcation Triangle (Corral-Pazos-de-Provenset al., 2018)_


### 2.6 Calculate Shear Strength

### 2.6.1 Calculating Sand Subfractions

To calculate shear strength parameters, sand sub-fractractions must be calculated first. Very fine sand, fine sand and coarse sand sub fractions are extracted based on methods proposed by [Corral-Pazos-de-Provenset al. (2018)](#http://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/10.1002/ldr.3121). The function `get_vfs` utilizes the RUSLE2 formula, ESDAC method and the Shirazi–Boersma theory to calculate very fine sand based on the soil classification computed above. 

<img width="700" height="463" alt="image" src="https://github.com/user-attachments/assets/ff238c5b-c5ba-4716-859c-f5800fc57819" />

_Models used to estimate very fine sand fraction (Corral-Pazos-de-Provenset al., 2018)_

<img width="650" height="594" alt="image" src="https://github.com/user-attachments/assets/5495af72-588e-4d6d-9038-0aab09f9f94a" />

_Model preference based on soil classification (Corral-Pazos-de-Provenset al., 2018)_

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
When `method = "subfraction"`, the model is based on a study by [Khaboushan et al. (2018)](#https://www.sciencedirect.com/science/article/abs/pii/S0167198718308031?via%3Dihub) and utilizes fine sand and very fine sand subfractions to determine the angle of internal friction. When using this method the friction angle (FA) $= 1.40 + 0.0001 \times (\mathrm fineSand^2) + 0.0001 * (\mathrm very Fine Sand^2)$. 
<!-- when use which method?? -->
<!-- also where is the GMD equation??? wanna make sure its the right source ig... idk im just confused @ teh equation-->
The GMD is based off of the methods proposed by [Luvai et al., (2022)](#https://onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/10.1155/2022/2122554) for calulating the angle of internal friction. With this method FA becomes $( 1.43 + 1.23 \times \mathrm GMD )$ with GMD being $(\mathrm sand * log10(D_{sand}) + silt * log10(D_{silt}) + clay * log10(D_{clay}) )$. D represents the dominant grain size **_(d<sub>50</sub>??)_** of the sand, silt and clay. <!-- ?? -->

Cohesion is calculated as a function of the subfractions of clay, coarse sand and very fine sand based on the study done by [Khaboushan et al. (2018)](#https://www.sciencedirect.com/science/article/abs/pii/S0167198718308031?via%3Dihub). The to calculate unsaturated cohesion of the soil, the function `unsat_cohesion` $= ( -0.75 + 2.07 \times \mathrm clay^{0.5} - 5.87 \times \mathrm log10 (\mathrm coarse sand) - 0.035 \times \mathrm very fine sand^2)$. If this equation spits out a negative value the function will normalize that to zero.


### 2.7 Hydrology

Cation Exchange Capacity (CEC) and pH are extracted from soil grid data accessed eariler with the `get_soilgrids` function.

### 2.7.1 Calculating K<sub>sat</sub>
To calculate K<sub>sat</sub> (saturated hydraulic conductivity) the function `transmissivity` is utilized. This function allows two methods; rosetta which assign K<sub>sat</sub> based on [USDA texture codes *(?)*](#https://www.ars.usda.gov/pacific-west-area/riverside-ca/agricultural-water-efficiency-and-salinity-research-unit/docs/model/rosetta-class-average-hydraulic-parameters/) and EU method based off of Future Water report by [Simons et al. (2020)](#https://www.futurewater.nl/wp-content/uploads/2020/10/HiHydroSoil-v2.0-High-Resolution-Soil-Maps-of-Global-Hydraulic-Properties.pdf) and a study by [Tóth et al. (2014)](#https://bsssjournals.onlinelibrary.wiley.com/doi/10.1111/ejss.12192).
The EU method utilizes the equation $( 0.40220 + 0.26122 \times \mathrm pH + 0.44565 \times \mathrm TS_{value} - 0.02329 \times \mathrm clay - 0.01265 \times \mathrm silt - 0.01038 \times \mathrm cec )$ to calulate $\mathrm log_{10}(k_{sat})$ where $TS_{value}$ is the distinction between subsoil and topsoil ([Simons et al., 2020)](#https://www.futurewater.nl/wp-content/uploads/2020/10/HiHydroSoil-v2.0-High-Resolution-Soil-Maps-of-Global-Hydraulic-Properties.pdf)). As K<sub>sat</sub> is logged and in the wrong units it is further derived using the equation $(10^{log_{10}(k_{sat})}) \div 100 \div 24)$.

Transimissivity is then calulated by multiplying K<sub>sat</sub> by soil depth extracted as `soil_depth` in section 2.4. <!-- function needs to be fixed so skip expanation for now? -->


> [!TIP]
> The code will prompt the user to clean menory using `rm` and `gc` functions. Do this to keep memory clear and computing power minimal.


### 2.7.2 Calibrating K<sub>sat</sub>
 <!-- work in progress code -->

This function aims to calibrate the calculated K<sub>sat</sub> for structural influence using the [Modis net primary production (NPP)](#https://modis.gsfc.nasa.gov/data/dataprod/mod17.php) based on findings by [Fan et al. (2022)](#https://agupubs-onlinelibrary-wiley-com.ezproxy.library.uvic.ca/doi/10.1029/2022GL100389) and [Bonetti et al., 2021](#https://www.nature.com/articles/s43247-021-00180-0). 
<!-- NPP represents the potential of the soil to support biological activity depending on the biome it is located. -->
This method utilizes RGEE and requires Earth engine to be set up properly to function. Using Earth Engine, the NPP for the target area is extracted and clipped to the target area.

To calibrate K<sub>sat</sub>, NPP must also be rescaled by a factor or 0.1 from $(kg C m^{-2} yr^{-1}$ to $g C m^-1 yr-1 (*1000))$. 
With this the K<sub>sat</sub> factor becomes 

$$ratio_{max} - ((ratio_{max} - 1) \div (1 + (\frac{NPP}{p})^q))$$
where $ratio_{max} = 10^{3.5 - (1.5 \times sand%^{0.13})}$ <!-- markdown doesnt like this last equation-->

K<sub>sat</sub> is then multiplied by this factor to end with the new calibrated K<sub>sat</sub> that takes into account conductivity with biomass present in the soil.

### 2.8 Calculating Normal Recharge with PCIC Data

PCIC (Pacific Climate Impacts Consortium) data contains meteorlogical data that will be used to extract climate normals with a particular focus on normal precipation in the target study area.

PCIC data is regionally specific data. If the target region is outside of this region, alternative data sources would be necessary to complete this step.

Average precipation data is extracted from the accessed CSV file over the last 30 years and the data's resolution <!--(do i need to know what 0.08333 or 0.1 deg means in this?--> is extracted to create a raster template for the data. Datapoints from the PCIC data are transfered onto this raster, converted to meters per hour, and rasterized _(across the whole are?)_. The 90th and 10th percentiles of this climate data are also extracted from the original datset.


### 2.9 Soil Density

Soil bulk density is calculated with the `bulk_density` function. This function uses ROSETTA method based on [???](#https://github.com/usda-ars-ussl/rosetta-soil) to assign volumetric water content in the soil based on its texture class. This model uses the density of quartz which is $2650kg/m^3$. Assumed density differs regionally and may change depending on the target area <!-- fact check?-->.

With a density of $2650kg/m^3$ as the assumed density, `bulk_density` becomes a function of $(density_{assumed} \times (1 - v)$. '$v$' represents theta_s ($\theta _s$). This value represents saturated volumetric water content which is dependant on the soil texture class.


### 2.10 Root Cohesion

Using Rgee, the satellite based forest inventory (SBFI) data is collected and bound by the area of interest (AOI). The SBFI is converted into a vector and theoretical maximum of basal area stand is set based off of regional data (cite?). A theoretical maximum root cohesion value is also set based on region. 

Regionally determined root cohesion with numbers being sources from studies such as from [Schmidt et al, (2021)](#https://cdnsciencepub.com/doi/10.1139/t01-031), [Sakals & Sidle (2004)](#https://cdnsciencepub.com/doi/10.1139/x03-268), and [Burroughs & Thomas (1977).

With the basal area maximum ($BA_{max}$), maximum root cohesion ($RCoh_{max}$) and average basal area (BA) extracted from SBFI data, root cohesion can be normalized with equation $((BA \div BA_{max}) \times RCoh_{max})$. In the code, the function that does this calulation looks like

```
sbfi$COHESION <- as.numeric((sbfi$STRUCTURE_BASAL_AREA_AVG / BA_max) * root_cohesion)
```

The cohesion is then rasterized and clamped between zero and the set maximum root cohesion for later computations.


### 2.11 Wildfire Effects

The main effect of wildfires calculated in this section is burn severity. 

To calculate burn severity, the difference normalized burn ration (dNBR) data is extracted through Google Earth Engine using the RGEE package. 

It should be noted that [this dataset](#https://gee-community-catalog.org/projects/ca_forest_fire/#dataset-citation) is a Canadian dataset. If the target region is not in Canada, alternative data will need to be accessed.

The data is clamped to the area of interest and then rescaled to the BARC256 for classifcation. To rescale the dNBR the equation $(dNBR_{(raw)} \times 2 + 55)$ is utilized. These rescaled values are additionally clamped from 0 to 255 and classified based on the BAR256 system. 

What is BARC256 -> "The normalized_burn_ratio (NBR) is used to assess a fire’s severity"/"BAER 8-bit datasets contain unclassified values of for use by BAER teams in the field. These datasets are referred to as BARC-256 by federal agencies."[(source)](#https://landscapetoolbox.org/remote-sensing-methods/burned-area-reflectance-classification-barc/) <!-- leave out or put in?-->

Burn year data is also an excellent tool for assessing burning of the target area at a specific time. By extracting burn year from the same projects as dNBR, a fire perimeter can be extracted based on the year of inetrest (2018 here) and burn severity of that season can be assessed using the function `dnbr_classified$clip(fire_perim)`. 

> [!NOTE]
> The task initiated prompts the user to export a burn severity raster to their Google Drive. After the file is properly uploaded it will need to be redownloaded and called in. Processing time for exporting the file this way are significantly shorter than attemting to save directly to the harddrive __*(assuming thats why it is done this way?)*__


### 2.11.1 Modifier Layers

The burn severity output raster is uutilized in this section of code to modify hydraulic condiuctivity (K<sub>sat</sub>) and root cohesion. 

The modification of K<sub>sat</sub> based on findings by [Abdollahi et al. (2024)](#https://www.sciencedirect.com/science/article/pii/S0013795224001388?via%3Dihub), [Abdollahi et al. (2023)](#https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2022EF003213), and [Ebel & Moody (2020)](#https://onlinelibrary.wiley.com/doi/10.1002/hyp.13865) indicated that hydraulic conductivity decreases about 66% in areas of moderate to high burn severity. These classifications are based on the BARC256 classifcation.

Root reinforcement based on burn severity is modified based on findings discussed by [Abdollahi et al. (2024)](#https://www.sciencedirect.com/science/article/pii/S0013795224001388?via%3Dihub) and [Abdollahi et al. (2023)](#https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2022EF003213)). Abdollahi et al. (2024) indicated that root cohesion decreased 50%  in moderately burned areas and decreased 80% in high severity burn areas. Abdollahi et al. (2023) found that this reduction wasassumed to be 25% in both moderate and high severity burned areas. As the 25% assumption is based on a single time-step during a fire, the 25% and 80% decrease will be utlized by the code to better encompass the loss of root cohesion over time. Depending on the desired application, this modifier can be altered.


### 2.11.2 Uncertainty

This section is optional however will allow for assessment of uncertainty in later calculated probality values. Minimum, maximum and standard devation values are caluclated for layers including friction angle, transmissivity, soil cohesion, root cohesion, soil depth, (bulk density) and (recharge). 


## 3. Landslide Probability

The function `landslide_probability` utilizes the LHS (latin hypercube sampling) to randomly select probable variations of input parameters.  The function then runs a determined number of loops based on these parameter samples to determine probability of the pixel failing under the determined conditions.


### 3.1 Inputs

Calculated parameters from the sections prior are utilized to create a landslide probability raster map for the target area. Friction angle (FA), transmissivity (transmiss), SCA (sca), soil cohesion (coh), root cohesion (coh_r), soil depth, bulk density and recharge are all used as paramenters. 

All parameters can be used as indidual rasters created through computation but can also be input as constants. For example, recharge values can be set as one constant across a region to simulate seasonal change in hydrology. Bulk density can also be assigned for the same reason.

> [!IMPORTANT]
> For the fuction to operate without error all inputs must be true, spatial rasters and have the same resolution, CRS and extents. If these requirements are not met a error message with pop up.


### Factor of Safety

The factor of safety (Fs) defines the balance between resisting and driving forces on a slope.

$$Fs = \frac {F_{resisting}}{F_{driving}}$$

When this fraction is below 1 the theoretical risk of the slope failing is high. If the fraction is above 1, the slope is theoretically stable and at a low risk of failing. This equation used to determine the factor of safety in the package is:

$$FS = \frac{(cohesion^* + cos(slope) \times (1 - wetness \times (\frac{density_w}{bulk.density})) \times tan(FA))}{sin(slope)}$$

> $cohesion^*$ is defined by $(\frac{coh \times 1000 + coh_r \times 1000)}{(bulk.density \times g \times depth_{soil})}$
> with $g$ representing the acceleration of gravity at 9.81m/s<sup>2</sup>.
>
> $wetness$ is defined by a precomputed value or a function of $(\frac{R}{transmissivity} \times \frac{sca}{sin(slope)})$ where $R$ is the computed recharge rate at m/hr. Thes values clamped between 0 and 1.
>
> $density_w$ is the density of water in kg/m<sup>3</sup> which is 1000.

The main code of the package does not include calculations for the factor of safety but it is included within the landslide probability function described in the next section.


### 3.2 Computing Probability

### 3.2.1 Hydrological Variables

Two types of hydrological scenarios can be utilized to calculate landslide probability. 
Steady state wetness can be utilzed to compute landslide probability based on long term scales depending on persistant recharge rates in the target area. This scenario can also be utilized to determine future climate scenarios. Steady state wetness is computed by the function below.

```
compute_wetness <- function(R, transmissivity, sca, sin_slope) {
  terra::clamp((R / transmissivity) * (sca / sin_slope), lower = 0, upper = 1)
}
```

Transient wetness scenarios/ transient hydrology model can be utilized to compute landslide probability over the period of a precipiation event. In this scenerario, rain intensity and duration of the event can be used as parameters to calculate the probability of landslides during the given event.
To model transient wetness the Optimized Green-Ampt Solver is used (<source>). The equation used in the model (<add in>) uses the numerical method to iterately (?) solve the model. After the model is solved it can be used to determine the total saturation of a soil package during the course of the precipiation event. This saturation amount informs the function of the true weight of the soil.

<!-- parts from the code that idk if i need to cover or not?

- helper functions for converting degrees to radians & computing sin_slope fr rad
- computing wetness function
- FS function
   - wetness/ cohesion*/ tan fa/ cos_slope can be overrided?
- omptimized green-ampt solver using pre-allocated Rainfall Template
   - transient wetness computation fr poding possibles
   - landslide prob w rain intensity, duration etc - 
- preclamp scalars - suggestion 3
   - rain intensity/ duration/ recharge steady/ n_bins = clamped between 0-      1? others = rasters?
   - precomps like normalized soil depth/ porosity base/ f_total 
   - LHS loop - prevents Na/Nan values
- progress bar shows up?
- simulation loop
- updated transient wetness based on stuff above
      wetness_total <- clamp(m1 + m2, lower = 0, upper = 1.0)
       m2 <- compute_transient_wetness(params$ksat, psi, d_theta_dynamic, 
                                    rain_intensity, duration,params$soil_depth, 
                                    F_total_template)
      m1 <- compute_wetness(params$R_steady, current_trans, sca, sin_slope)
- calculate saturated density using porosity and density of water (1000 kg/m3)
    - porosity_base is already 1 - (bulk_density / 2650) in your code
- fs is redefined (fr FS) including dynamic bulk density (includes          saturated density)
- prob_landslide <- prob_landslide + (inv_n_bins / (1 + exp(alpha * (fs - 1))))
- prob of failure -> 5 bins
- perturb_settings =?? Preturbation `perturb_settings` (...)
-->






### 3.3 Optional Manual Options

Within the andslide probability function `n_bins` is the number of equally likely bins and defines how many times the model is run. This can be changed based on the user's goals. 
Using this value, the cumulative probabilities of landslides using `seg(0, 1, by = 1/n_bins)`.
The results of this function are then plotted to extract their best fit normal function which can be used to determine the distribution of landslide risk within the target region. 

The output should show something similar to this: _(insert image)_



## 4. References
<!-- hyperlink references from where theyre referenced? or could have sections of refences... want references links to be open source otherwise idk if tehy work-->
