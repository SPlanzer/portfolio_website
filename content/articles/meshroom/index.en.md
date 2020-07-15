+++
title = "Photogrammetry with Meshroom"
summary = "This article explorse building a Digital Surface Model (DSM) from drone photos at the Newtown DIY skatepark"
date = "2020-05-09T21:34:30+02:00"
tags = ["Photogrammetry", "Structure From Motion", "SFM"]
author = ""
images = ["articles/meshroom/banner.png"]
+++

{{< imgScale "pouring.png" "Construction of the Newtown Community Skatepark that is to be surveyed using photogrammetry techniques" "500x" >}}
## Intro
This is an investigation into using Meshroom for building a Digitial Surface Model of the Newtown Community skate park. This was undertaken to help plan the future extentions to the facility and better liase with City Council.


## Meshroom
Meshroom is an opensource photogrammetry project by...

The software finds common clusters of pixels known as "features" in multiple images. It then uses these common clusters and the images metadata to calculate the position and angle of the camera when each image was captured. Once the camera values are calculated, point clouds, meshes and textures can them be generated.
TODO add detail from - https://alicevision.org/#photogrammetry/sfm

## Capturing the images
For this project a DJI Phanton3 drone was deployed

When taking pictures for photogramtry there are several factors that must be followed to ensure a good model is produces:
* Full coverage of all terrain must be captured from all angles
* All terrain must be captured in at least two images. Therefore each image should overlap by about 60-80% the succeeding image.
* Do not capture moving objects in the images such as people or drone cases that may be picked up and moved during the survey
* Avoid shadows. This is best done by collecting imagery on a lightly overcast day.
* Avoid harsh light. The day the imagery was flown for this project much of the terrain was wet. This meant even with the diffussed light provided by the cloud cover much of the terrain gave of glare that reduces the information the camera can capture due to over exposure.

For the reason outlined in the last point. This survey only captured the western quarter of the community saktepark and more surveys will be required in more optimal conditions to capture the remained proportion of the facility.







{{< imgScale "images.png" "The above shows Meshroom with all the images used to build the model" "1000x" >}}


{{< imgScale "cameras.png" "The camera positions and their angles as calculated by meshroom" "1000x" >}}


## Running Meshroom
To build the surface model Meshroom requires access to a NVIDIA CUDA GPU. In my case I do not have access to a machine with this GPU. But fortuantly Meshroom has a command line input and the application can therefore be run using cloud computing resources that make the required GPU available.

In this case I used Google Colab as the cloud computing resource provides free access to a GPU for up to 12 hours.

Below is the Google Colab script for mounting data installed in Google Drive, installing Meshroom to Google Colab and running Meshroom itself.

```

# Variables
inputFolderDrive = "/content/drive/My\ Drive/sfm/NewtownCommunitySkatepark" # G Drive mount point/ folder with the input images
outputFolderDrive = "/content/drive/My\ Drive/sfm/NewtownCommunitySkatepark/output" # G Drive mount point/ folder for the outcome
workingDir = "/content/currentProject" #notebook folder to work in
meshroomRoot = workingDir + "/meshroom"
filesInput = workingDir + "/input"
filesOutput = workingDir + "/output"
# piplelineFile = inputFolderDrive+"/config.mg"
# Mount G Drive
from google.colab import drive
drive.mount('/content/drive')

# Create Dirs
!mkdir -v $workingDir
!mkdir -v $meshroomRoot
!mkdir -v $filesOutput
!mkdir -v $filesInput
!mkdir -v $outputFolderDrive

# Change root to working dir
%cd $workingDir

# Copy Input

%cp -arv $inputFolderDrive/* $filesInput

# Install Meshroom
import pathlib
file = pathlib.Path("/content/currentProject/meshroom/Meshroom-2019.2.0")

if not file.exists ():
  !wget -N https://github.com/alicevision/meshroom/releases/download/v2019.2.0/Meshroom-2019.2.0-linux.tar.gz
  !tar xzf Meshroom-2019.2.0-linux.tar.gz -C $meshroomRoot
  %mv -v /content/currentProject/meshroom/Meshroom-2019.2.0/* $meshroomRoot
else :
   print ("Meshroom is allredy installed!")

# Run Meshroom
meshroomRun = meshroomRoot+"/meshroom_photogrammetry"
!$meshroomRun --input $filesInput --output $filesOutput

# Copy Output
%cp -arv $filesOutput/* $outputFolderDrive

# Clean
%rm -Rv $filesInput
%rm -Rv $filesOutput
```

# The Output

TODO
* Mention there outputs. Point cloud, mesh and texture.
* Mention meshlab
* Mention the ability to measure.


{{< imgScale "overview.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}



{{< imgScale "quarter.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}

{{< imgScale "curb.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}

# Next Steps
The next step for this project is to capture the remaineder of the park when optimal lighting conditions allow.

I also have access a u-blox GNSS reciever. A low cost receiver capable of centermeter accuracy. Once the entire facility is captured by point cloud, reference points will be surveyed using this reciever. This will allow the model to be tied to spatial reference system. Once this is the case it can be tied into other course DSM that exist to give further context and spatial analysis can also be performed on the model.