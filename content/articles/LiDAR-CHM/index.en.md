+++
title = "Canopy Height Model (CHM) from LiDAR"
summary = "Extracting vegetation height from LiDAR is straight forward. Such an approach can be  of use when distinguishing between vegetation types such as forest and scrub"
date = "2020-05-07T22:12:12+03:00"
tags = ["LiDAR"]
author = ""
images = ["articles/lidar-chm/banner.png"]
+++




## Intro
To explore extracting vegetation height from LiDAR data, a swath from the Land-use and Carbon Analysis System (LUCAS) LiDAR data was acquired from the Ministry for the Environment(MFE). This data is idea for such calculations as its collected in heavily forested areas and is a resolution high enough to analyse individual trees.

The computation of vegetation height using LiDAR is fairly straightforward.
1. A Digital Surface Model (DSM) must be extracted from the LiDAR point data. This is a model representing features (i.e vegetation) elevated above the “Bare Earth”.
2. A Digital Terrain Model (DTM) must be extracted from the LiDAR point data. This is a bare Earth model with surface features not included.
3. What is commonly known as a Canopy Height Model (CHM) is then produced by subtracting the DTM from the DSM.

## Extracting DSM and DTM
[PDAL](https://pdal.io/) (Point Data Abstraction Library) is a very powerful tool for processing LiDAR point cloud data and very adept extracting surface models models based on point classification and filtering algorithms.

PDAL allows the composition of operations on point clouds into 'pipelines'. These pipelines are written in the JSON format and define each step for the processing the data in a sequence. This leads itself to highly customiable point cloud processing, that is repeatable and runs many steps in one execution of the program.



#### DSM PDAL Pipeline
Below is the pipeline I composed for extracting the DSM from the LiDAR point cloud.

```

{

    "pipeline":[
        "LEVEL2_24304A_20150414_S004/LEVEL2_24304A_20150414_S004.las",
        {
            "type":"filters.reprojection",
            "in_srs":"EPSG:2193",
            "out_srs":"EPSG:2193"
        },
        {
            "type":"filters.range",
            "limits":"returnnumber[1:1]"
        },

        {
            "type": "writers.gdal",
            "filename":"dsm.tif",
            "output_type":"idw",
            "gdaldriver":"GTiff",
            "resolution": 0.5,
            "radius": 1

        }
    ]
}


```

#### Parameters
The PDAL parameters most important to extracting the Surface Model are outlined below

##### Filter
A return filter is applied that filters on the "returnnumber". To generate a DSM we only want first returns as these are the returns from the heights points measured by each light pulse. These first returns are represents by the returnnumber 1

##### Writers
The writer defines the pipelines output type and parameters for calculating the output.

Of use in this case is the:
* resolution: Length of raster cell edges in X/Y units.
* Radius: The value where each point that falls within a given radius of each raster cell center contributes to the raster cells value.

#### DTM PDAL Pipelne

```

{
    "pipeline":[
         "LEVEL2_24304A_20150414_S004/LEVEL2_24304A_20150414_S004.las",
        {
            "type":"filters.reprojection",
            "in_srs":"EPSG:2193",
            "out_srs":"EPSG:2193"
        },
        {
          "type":"filters.assign",
          "assignment":"Classification[:]=0"
        },
        {
          "type":"filters.elm"
        },
        {
          "type":"filters.outlier"
        },
        {

          "type":"filters.smrf",
          "ignore":"Classification[7:7]",
          "slope":0.2,
          "window":16,
          "threshold":0.45,
          "scalar":1.2
        },
        {
          "type": "writers.gdal",
          "last":true,
          "filename":"dtm.tif",
          "output_type":"idw",
          "gdaldriver":"GTiff",
          "resolution": 0.5,
          "radius": 1
        },
        {
          "type":"filters.range",
          "limits":"Classification[2:2]"
        }
    ]
}
```

#### Parameters
The PDAL parameters important to extracting the LiDAR ground returns that represented a DTM are discussed in the following sections.

##### Classification Values
In this case we want to ignore any LiDAR classification values that may have already been calculated so that we can derive our own.
In this example we apply a value of 0 (not classified) to the Classification dimension for every point.

##### Extended Local Minimum (ELM)
The Extended Local Minimum (ELM) algorithm identifies low noise points that can adversely affect ground segmentation algorithms. Noise points are classified with a value of 7.

##### Outliers
Used to identify any outlier points that may affect ground segmentation. Outliers can be a result of signal noise but also phenomena such as birds captured in a survey. Outliers are also classified with a value of 7.

##### Ground Classification
The `filters.smrf` classifies ground points based on the well accepted Simple Morphological Filter (SMRF) approach. The points classified as noise (value 7) are filter out of the final result. Following the LAS format, ground points are given the classification value of 2.

##### Ground Return Extraction
Having classified ground points by giving these points the classification value of 2, ground points can now be filtered out to create a model solely representing bare ground.

## Running the pipelines

Once the pipelines are defined in json it is simple as running the below commands to build the models

`pdal pipeline dxm_pipeline.json`


## Calculating the CHM
Once we have the two models, GDAL can be used to run a raster calculator to output the CHM.

```gdal_calc.py -A dtm.tif -B dsm.tif --calc="B-A" --outfile chm.tif –overwrite```


## Visualising the CHM model via Python and matplotlib
As a means to visualise the CHM, Python along with Matplotlib was used. This method of visualisation was selected, as with one script a well annotated map can be generated in a format that is ready to include in documents such as reports and in this case, a website.

{{< imgScale "chm.png" "A visualization of the Canopy Height Model (CHM)" "440x" >}}

```
import numpy as np
import os
import gdal, osr
import matplotlib.pyplot as plt
import sys
import matplotlib.pyplot as plt
from scipy import ndimage as ndi

# Import biomass specific libraries
from skimage.morphology import watershed
from skimage.feature import peak_local_max
from skimage.measure import regionprops
from sklearn.ensemble import RandomForestRegressor


def plot_band_array(
    band_array, image_extent, title, cmap_title, colormap, colormap_limits
):
    plt.imshow(band_array, extent=image_extent)
    cbar = plt.colorbar()
    plt.set_cmap(colormap)
    plt.clim(colormap_limits)
    cbar.set_label(cmap_title, rotation=270, labelpad=20)
    plt.title(title)
    ax = plt.gca()
    ax.ticklabel_format(useOffset=False, style="plain")
    rotatexlabels = plt.setp(ax.get_xticklabels(), rotation=90)


def array2raster(newRasterfn, rasterOrigin, pixelWidth, pixelHeight, array, epsg):

    cols = array.shape[1]
    rows = array.shape[0]
    originX = rasterOrigin[0]
    originY = rasterOrigin[1]

    driver = gdal.GetDriverByName("GTiff")
    outRaster = driver.Create(newRasterfn, cols, rows, 1, gdal.GDT_Float32)
    outRaster.SetGeoTransform((originX, pixelWidth, 0, originY, 0, pixelHeight))
    outband = outRaster.GetRasterBand(1)
    outband.WriteArray(array)
    outRasterSRS = osr.SpatialReference()
    outRasterSRS.ImportFromEPSG(epsg)
    outRaster.SetProjection(outRasterSRS.ExportToWkt())
    outband.FlushCache()


chm_file = "/home/splanzer/git/chm/chm.tif"


# Get info from chm file for outputting results
just_chm_file = os.path.basename(chm_file)
just_chm_file_split = just_chm_file.split(sep="_")

# Open the CHM file with GDAL
chm_dataset = gdal.Open(chm_file)

# Get the raster band object
chm_raster = chm_dataset.GetRasterBand(1)

# Get the NO DATA value
noDataVal_chm = chm_raster.GetNoDataValue()

# Get required metadata from CHM file
cols_chm = chm_dataset.RasterXSize
rows_chm = chm_dataset.RasterYSize
bands_chm = chm_dataset.RasterCount
mapinfo_chm = chm_dataset.GetGeoTransform()
xMin = mapinfo_chm[0]
yMax = mapinfo_chm[3]
xMax = xMin + chm_dataset.RasterXSize / mapinfo_chm[1]
yMin = yMax + chm_dataset.RasterYSize / mapinfo_chm[5]
image_extent = (xMin, xMax, yMin, yMax)


# Plot the original CHM
plt.figure(1)
chm_array = chm_raster.ReadAsArray(0, 0, cols_chm, rows_chm).astype(np.float)

# PLot the CHM figure
plot_band_array(
    chm_array,
    image_extent,
    "Canopy height Model",
    "Canopy height (m)",
    "Greens",
    [0, 9],
)
plt.savefig(
    "/home/splanzer/git/chm/CHM.png",
    dpi=300,
    orientation="landscape",
    bbox_inches="tight",
    pad_inches=0.1,
)

# gaussian filtering
chm_array_smooth = ndi.gaussian_filter(
    chm_array, 2, mode="constant", cval=0, truncate=2.0
)
chm_array_smooth[chm_array == 0] = 0

array2raster(
    "/home/splanzer/git/chm/chm_filter.tif",
    (xMin, yMax),
    1,
    -1,
    np.array(chm_array_smooth / 10000, dtype=float),
    32611,
)

local_maxi = peak_local_max(chm_array_smooth, indices=False, footprint=np.ones((5, 5)))

plt.figure(2)
plot_band_array(local_maxi, image_extent, "Maximum", "Maxi", "Greys", [0, 1])
plt.savefig(
    "/home/splanzer/git/chm//Maximums.png",
    dpi=300,
    orientation="landscape",
    bbox_inches="tight",
    pad_inches=0.1,
)
```