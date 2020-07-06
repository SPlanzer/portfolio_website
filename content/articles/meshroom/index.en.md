+++
title = "Photogrammetry with Meshroom"
summary = "This article explorse building a Digital Surface Model (DSM) from drone photos at the Newtown DIY skatepark"
date = "2020-05-09T21:34:30+02:00"
tags = ["Photogrammetry", "Structure From Motion", "SFM"]
author = ""
images = ["articles/meshroom/banner.png"]
+++





{{< imgScale "images.png" "the images used to build the model" "1000x" >}}


{{< imgScale "cameras.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}



```

# Variables
inputFolderDrive = "/content/drive/My\ Drive/sfm/ghetto_v2" # G Drive mount point/ folder with the input images
outputFolderDrive = "/content/drive/My\ Drive/sfm/ghetto_v2/output" # G Drive mount point/ folder for the outcome
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


{{< imgScale "overview.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}



{{< imgScale "quarter.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}

{{< imgScale "curb.png" "The camera positions and there angles as calculated by meshroom" "1000x" >}}

