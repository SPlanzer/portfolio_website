+++
title = "Canopy Height Model (CHM) from LiDAR"
summary = "Extracting vegetation height from LiDAR is straight forward. Such an approach can be  of use when distinguishing between vegetation types such as forest and scrub"
date = "2020-05-07T22:12:12+03:00"
tags = ["LiDAR"]
author = ""
images = ["articles/LiDAR-CHM/banner.png"]
+++




## Intro
To explore extracting vegetation height from LiDAR data a swath of the Land-use and Carbon Analysis System (LUCAS) LiDAR data was acquired from the Ministry for the Environment(MFE). This data is idea for such calculations as it collected in heavily forested areas and is a resolution high enough to analyse individual trees.

The concept of vegetation height is very simple.
1. A Digital Surface Model (DSM) must be extracted from the LiDAR point data. This is a model representing features (i.e vegetation) elevated above the “Bare Earth”.
2. A Digital Terrain Model (DTM) must be extracted from the LiDAR point data. This is a bare Earth model with surface features not included.
3. The DSM must be subtracted for the DTM to produce a Canopy height model (CHM)



## Extracting DSM and DTM
To be able to calculate a Canopy height model (CHM) a DSM and DTM are required. PDAL (Point Data Abstraction Library) is a very powerful tool for processing LiDAR point clouds data and very adept extracting such models.

PDAL allows the composition of operations on point clouds into 'pipelines'. These pipelines are written in the JSON format and define each step for the processing. This leads itself to highly customisable point cloud processing



#### DSM
Below is the pipeline for extracting the DSM

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
            "limits":"returnnumber[3:5]"
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
A return filter is applied that filters on just the returns in the range. In this class only the values of 3-5 are returned. These values are defined by the LAS formatted a vegetation.

##### Writers
The writer defines the pipelines outputs.

Of use in this case is the:
*resolution: Length of raster cell edges in X/Y units
* Radius: The value where each point that falls within a given radius of each raster cell center contributes to the raster’s value

#### DTM

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

Water

10


Rail

11


Road Surface
          "type":"filters.smrf",
          "ignore":"Classification[7:7]",
          "slope":0.2,
          "window":16,
          "threshold":0.45,
          "scalar":1.2The Range Filter applies filtering to the input point cloud based on a set of criteria. In this case

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
The PDAL parameters important to extracting the LiDAR ground returns that are those we want to be represented in the DTM output are disccused in the following sections

##### Classification Values
In this case we want to ignore any LiDAR classification values that may have already been calculated so that we can derive our own.
In this example we  apply a value of 0 to the Classification dimension for every point.

##### Extended Local Minimum (ELM)
This applies the Extended Local Minimum (ELM) algorithm to identify low noise points that can adversely affect ground segmentation algorithms. Noise points are classified with a value of 7.

##### Outliers
Used to identify any outlier points that may affect ground segmentation. Outliers can be a result of signal noise but also phenomena such as birds captured in a survey. Again noise points are classified with a value of 7.

##### Ground Classification
Classifies ground points based on the well accepted The Simple Morphological Filter (SMRF) approach. The points classified as noise (value 7) are filter out of the final result. Following the LAS format ground points are given the value of 2.

##### Ground Return Extraction
Having classified ground points by giving these points the classification value of 2, ground points can now be filtered out to create a model solely representing bare ground.

## Running the pipelines

once the pipelines are defined in json it is simple are running the below commands to build the models

`pdal pipeline dsm_pipeline.json`
and
`pdal pipeline dtm_pipeline.json`

\* of course the DSM and DTM pipelines could be defined in the one pipeline but it can be useful to keep them separate for modularity reasons.

## Calculating the CHM
Once we have the two models GDAL can be used to run a raster to output the CHM.

```gdal_calc.py -A dtm.tif -B dsm.tif --calc="B-A" --outfile chm.tif –overwrite```



https://gis.stackexchange.com/questions/341223/how-to-substract-2-dem-tiff-rasters-through-gdal




## Visualising the CHM model via Python and matplotlib
As a means to visualise the model Python was used as it does not require specialised software (e.g GIS software) to do so. The purpose of this exploration was to generate the CHM model so I am not going into detail here about the visualisation but the code can be found below to do this.

{{< imgScale "CHM.png" "The CHM as visualised via Python and Matplotlib" "500x" >}}

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


chm_file = "/home/splanzer/git/chm/old_tech/chm_smaller_extent.tif"


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
    "/home/splanzer/git/chm/old_tech/CHM_smaller_extent.png",
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
    "/home/splanzer/git/chm/old_tech/chm_filter.tif",
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
    "/home/splanzer/git/chm/old_tech/Maximums.png",
    dpi=300,
    orientation="landscape",
    bbox_inches="tight",
    pad_inches=0.1,
)
```