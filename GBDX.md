# Beachfront Algorithms on DigitalGlobe's GBDX Platform

## Overview

Under the GEOINT Services contract, Radiant Solutions replicated a small subset of Beachfront functionality to operate on DigitalGlobe's GBDX Platform. The intent of this work was to demonstrate the feasibility of deploying the shoreline detection algorithm only to an alternate platform that was "close to the data" and able to efficiently scale to enable processing of large volumes of data.

This document is intended to summarize that effort, documenting the tasks that were developed and deployed to GBDX, and the processing workflows (recipes) that were developed and deployed to AnswerFactory.

### Background

We refer the reader to GBDX documentation at GBDX University (https://gbdxdocs.digitalglobe.com/) for in depth documentation of GBDX. Of particular interest will be the "Tasks and Workflows Guide" at https://gbdxdocs.digitalglobe.com/docs/task-and-workflow-course. Though not required, most of our interaction with the GBDX API is marshaled by gbdxtools (https://gbdxtools.readthedocs.io/en/latest/), a Python package that simplifies API access, including authentication and authorization.

## GBDX Tasks

Three tasks were developed to support shoreline detection in GBDX.

1. **Shoreline Detection** - The original task developed to produce shorelines from WorldView and Landsat imagery, was a close port of the Python code used in Beachfront. Supporting libraries and the shoreline algorithm itself were compiled into a Docker image and posted to the venicegeo organization on DockerHub. From there, GBDX ingests the Docker image and starts containers as needed.
2. **Otsu Thresholding** - For Sentinel-2, we opted to veer slightly from the original implementation. We can use existing GBDX tasks to compute the Modified NDWI raster. The next step in the Beachfront processing workflow is to compute the optimal Otsu threshold to binarize the NDWI result. Because this may be considered a general purpose task that others could use, we split it out as a separate GBDX task. Again, the code is derived from the original Beachfront codebase and turned into a Docker image for use within GBDX.
3. **Potrace Vectorization** - The final step in the Beachfront workflow is to vectorize the thresholded raster using potrace. Again, because this is not unique to shoreline detection, we isolated it as a separate GBDX tasks that can be reused by others.

These tasks are described in greater detail in the subsections below.

### Shoreline Detection

WorldView and Landsat imagery both use the same shoreline detection task within GBDX. The task combines all steps of the shoreline processing, including NDWI, Otsu thresholding, and potrace vectorization. A detailed description of the processing workflow is beyond the scope of this document. Suffice to say that our intent was to mirror as closely as possible the Beachfront algorithmic approach.

#### Building the Shoreline Detection Image

The shoreline detection source code can be found at https://github.com/venicegeo/shoreline-task. To build the Docker image, users must have Docker installed on their machine. The command to rebuild (and tag) the image from the command prompt is simply

```bash
docker build -t venicegeo/shoreline-task .
```

To push it to Docker Hub, enter the following at the command prompt.

```bash
docker push venicegeo/shoreline-task
```

For Docker images to be accessible to GBDX, we must add `tdgdeploy` as a collaborator through the Docker Hub interface.

#### Registering the Shoreline Detection Task

Here is the JSON payload used to create the existing shoreline detection task.

```json
{
    "name": "DGIS_CreateShorelineVectors_beta",
    "version": "0.0.1",
    "description": "Writes tide prediction JSON, shoreline GeoJSON, and NDWI GeoTIFF.",
    "taskOwnerEmail": "bradley.chambers@radiantsolutions.com",
    "properties": {
        "isPublic": False,
        "timeout": 7200
    },
    "inputPortDescriptors": [
        {
            "name": "cat_id",
            "type": "string",
            "description": "Catalog ID of the image to be processed.",
            "required": True
        },
        {
            "name": "minsize",
            "type": "string",
            "description": "Minimum coastline enclosed area/speckle suppression. (Default: 1000.0 pixels)",
            "required": False
        },
        {
            "name": "smooth",
            "type": "string",
            "description": "Corner smoothing from 0 (no smoothing) to 1.33 (no corners). (Default: 1.0)",
            "required": False
        },
        {
            "name": "image",
            "type": "directory",
            "description": "The multispectral image (NDWI and shoreline detection).",
            "required": True
        }
    ],
    "outputPortDescriptors": [
        {
            "name": "vector",
            "type": "directory",
            "description": "Output directory containing vector data."
        },
        {
            "name": "raster",
            "type": "directory",
            "description": "Output directory containing NDWI raster."
        }
    ],
    "containerDescriptors": [
        {
            "type": "DOCKER",
            "properties": {
                "image": "venicegeo/shoreline-task"
            },
            "command": "python /shoreline-task.py"
        }
    ]
}
```

Whenever the `venicegeo/shoreline-task` Docker image changes and the task needs to be updated, the version number of the task should be incremented accordingly in the JSON shown above. Of course, if anything else about the task (inputs, outputs, command, etc.) need to be changed, this should be reflected as well.

### Otsu Threshold

In contrast to the current WorldView and Landsat shoreline detection workflows, which use a single GBDX task to generate the shoreline vector, Sentinel-2 computes the Modified NDWI raster in a separate GBDX task. This leaves the Otsu threshold and Potrace vectorization steps to be completed. We outline the Otsu task here.

#### Building the Otsu Threshold Image

The Otsu threshold source code can be found at https://github.com/venicegeo/dg-otsu-task. To build the Docker image, users must have Docker installed on their machine. The command to rebuild (and tag) the image from the command prompt is simply

```bash
docker build -t venicegeo/dg-otsu-task .
```

To push it to Docker Hub, enter the following at the command prompt.

```bash
docker push venicegeo/dg-otsu-task
```

For Docker images to be accessible to GBDX, we must add `tdgdeploy` as a collaborator through the Docker Hub interface.

#### Registering the Otsu Threshold Task

Here is the JSON payload used to create the existing Otsu threshold task.

```json
{
    "name": "DGIS_OtsuThreshold_beta",
    "version": "0.0.1",
    "description": "Use Otsu's method to caluclate optimal threshold ignoring nodata values.",
    "taskOwnerEmail": "bradley.chambers@radiantsolutions.com",
    "properties": {
        "isPublic": False,
        "timeout": 7200
    },
    "inputPortDescriptors": [
        {
            "name": "image",
            "type": "directory",
            "description": "The data.",
            "required": True
        }
    ],
    "outputPortDescriptors": [
        {
            "name": "threshold",
            "type": "string",
            "description": "Otsu's threshold."
        }
    ],
    "containerDescriptors": [
        {
            "type": "DOCKER",
            "properties": {
                "image": "venicegeo/dg-otsu-task"
            },
            "command": "python /otsu-task.py"
        }
    ]
}
```

Whenever the `venicegeo/dg-otsu-task` Docker image changes and the task needs to be updated, the version number of the task should be incremented accordingly in the JSON shown above. Of course, if anything else about the task (inputs, outputs, command, etc.) need to be changed, this should be reflected as well.

### Potrace

In contrast to the current WorldView and Landsat shoreline detection workflows, which use a single GBDX task to generate the shoreline vector, Sentinel-2 computes the Modified NDWI raster in a separate GBDX task. This leaves the Otsu threshold and Potrace vectorization steps to be completed. We outline the Potrace task here.

#### Building the Potrace Image

The Potrace source code can be found at https://github.com/venicegeo/dg-potrace-task. To build the Docker image, users must have Docker installed on their machine. The command to rebuild (and tag) the image from the command prompt is simply

```bash
docker build -t venicegeo/dg-potrace-task .
```

To push it to Docker Hub, enter the following at the command prompt.

```bash
docker push venicegeo/dg-potrace-task
```

For Docker images to be accessible to GBDX, we must add `tdgdeploy` as a collaborator through the Docker Hub interface.

#### Registering the Potrace Task

Here is the JSON payload used to create the existing Potrace task.

```json
{
    "name": "DGIS_Potrace_beta",
    "version": "0.0.1",
    "description": "Trace raster image using potrace and return geolocated or lat-lon coordinates.",
    "taskOwnerEmail": "bradley.chambers@radiantsolutions.com",
    "properties": {
        "isPublic": False,
        "timeout": 7200
    },
    "inputPortDescriptors": [
        {
            "name": "image",
            "type": "directory",
            "description": "The data.",
            "required": True
        },
        {
            "name": "threshold",
            "type": "string",
            "description": "Threshold the input image.",
            "required": True
        },
        {
            "name": "minsize",
            "type": "string",
            "description": "Minimum coastline enclosed area/speckle suppression. (Default: 1000.0 pixels)",
            "required": False
        },
        {
            "name": "smooth",
            "type": "string",
            "description": "Corner smoothing from 0 (no smoothing) to 1.33 (no corners). (Default: 1.0)",
            "required": False
        }
    ],
    "outputPortDescriptors": [
        {
            "name": "result",
            "type": "directory",
            "description": "Potrace result."
        }
    ],
    "containerDescriptors": [
        {
            "type": "DOCKER",
            "properties": {
                "image": "venicegeo/dg-potrace-task"
            },
            "command": "python /potrace-task.py"
        }
    ]
}
```

Whenever the `venicegeo/dg-potrace-task` Docker image changes and the task needs to be updated, the version number of the task should be incremented accordingly in the JSON shown above. Of course, if anything else about the task (inputs, outputs, command, etc.) need to be changed, this should be reflected as well.

## AnswerFactory Recipes

### WorldView

### Landsat

### Sentinel-2
