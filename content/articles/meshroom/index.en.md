+++
title = "Photogrammetry with Meshroom"
summary = "This article explores building a Digital Surface Model (DSM) from UAV imagery at the Newtown DIY Skatepark"
date = "2020-05-09T21:34:30+02:00"
tags = ["Photogrammetry", "Structure From Motion", "SFM"]
author = ""
images = ["articles/meshroom/banner.png"]
+++


## Intro
This is an investigation into using Meshroom for building a Digital Surface Model of the Newtown Community Skatepark. This was undertaken to help plan the future extension to the facility and better liaise with City Council.


## Meshroom
Meshroom is an opensource photogrammetry project. The Meshroom software uses Photogrammetry and Structure From Motion (SFM) principles to build 3D models. Very simply put this is accomplish by the software finding common clusters of pixels known as "features" in multiple overlapping images. The software then uses these common clusters and the image metadata to calculate the position and angle of the camera relative to the captured image. With this information 3D points are triangulated to build a point cloud.


## Capturing the images
For this project a DJI Phanton3 drone was deployed.

When taking pictures for photogrammetry there are several factors that must be considered to ensure a good model is produced:

* All terrain in the survey area must be captured in at least two images and each image should overlap the succeeding image by 60-80%.
* Moving objects such as people or objects that may be picked up and moved during the survey must be avoided as this will result in images not being matched.
* Avoid shadows as they reduce the information captured by the camera. This is best done by collecting imagery on a lightly overcast day.
* Avoid harsh light. The day the imagery was flown for this project much of the terrain was wet. This meant even with the diffused light provided by the cloud cover much of the terrain gave of glare that reduces the information the camera can capture due to over exposure.

For the reason outlined in the last point. This survey only captured the western quadrant of the community skatepark and more surveys will be required in more optimal conditions to capture the remaining proportion of the facility.







{{< imgScale "images.png" "The above shows Meshroom with all the images used to build the model (click to enlarge)" "1000x" >}}


{{< imgScale "cameras.png" "The camera positions and their angles as calculated by Meshroom" "1000x" >}}


## Running Meshroom
To build the surface model, Meshroom requires access to a NVIDIA CUDA GPU. In my case I do not have access to a machine with such a GPU. But fortunately Meshroom has a command line interface and the application can therefore be run using cloud computing resources that make the required GPU available.

In this case I used Google Colab as the cloud computing resource provides free access to a GPU for up to 12 hours.

Below is the Google Colab script for mounting data installed in Google Drive, installing Meshroom to the Google Colab environment and running Meshroom itself.

```

# Variables
inputFolderDrive = "/content/drive/My\ Drive/sfm/NewtownCommunitySkatepark" # Google Drive mount point/ folder with the input images
outputFolderDrive = "/content/drive/My\ Drive/sfm/NewtownCommunitySkatepark/output" # Google Drive folder for the output
workingDir = "/content/currentProject" # notebook folder to work in
MeshroomRoot = workingDir + "/Meshroom"
filesInput = workingDir + "/input"
filesOutput = workingDir + "/output"
# piplelineFile = inputFolderDrive+"/config.mg"
# Mount G Drive
from google.colab import drive
drive.mount('/content/drive')

# Create Dirs
!mkdir -v $workingDir
!mkdir -v $MeshroomRoot
!mkdir -v $filesOutput
!mkdir -v $filesInput
!mkdir -v $outputFolderDrive

# Change root to working dir
%cd $workingDir

# Copy Input

%cp -arv $inputFolderDrive/* $filesInput

# Install Meshroom
import pathlib
file = pathlib.Path("/content/currentProject/Meshroom/Meshroom-2019.2.0")

if not file.exists ():
  !wget -N https://github.com/alicevision/Meshroom/releases/download/v2019.2.0/Meshroom-2019.2.0-linux.tar.gz
  !tar xzf Meshroom-2019.2.0-linux.tar.gz -C $MeshroomRoot
  %mv -v /content/currentProject/Meshroom/Meshroom-2019.2.0/* $MeshroomRoot
else :
   print ("Meshroom is allredy installed!")

# Run Meshroom
MeshroomRun = MeshroomRoot+"/Meshroom_photogrammetry"
!$MeshroomRun --input $filesInput --output $filesOutput

# Copy Output
%cp -arv $filesOutput/* $outputFolderDrive

# Clean
%rm -Rv $filesInput
%rm -Rv $filesOutput
```

# The Output

Meshroom generates the below outputs:

* 3D point cloud
* The 'Mesh'. This is a tetrahedralization of the model
* The 'texture' that sees the original photos sliced so that the can be draped over the tetrahedral mesh.

These models are an excellent tool for remote planning when gaining consensus as to future plans.

By using the camera metadata, specifically the pixel size being a key element in the SFM step allows for measurements to be taken from the model. This has proven particularly useful when calculating how many cubed metres of concrete is required on site for each build.


{{< imgScale "overview.png" "Overview of the western quadrant of the facility. The 3D point cloud is to the left and the mesh with texture on the right. " "1000x" >}}



{{< imgScale "quarter.png" "The entrance 'bowled quarter'. Note that decimeter detail can be seen." "1000x" >}}

{{< imgScale "curb.png" "'Curbed Pyramid' Note the artifacts from shadows that can be seen and thus the need to capture imagery on days with diffused light" "1000x" >}}

# Next Steps
The next step for this project is to capture the remainder of the park when optimal lighting conditions allow.

Having access to a low cost, decimeter accurate, u-blox GNSS receiver, reference points in the model are to be surveyed to tie the model to a geodetic reference system. This will allow the model to be tied into other larger scale DSM such as those provided by local government LiDAR collection. This will allow us to provide greater context as to how the community skatepark sits within the greater environs of Newtown.
