# Beachfront Algorithms on DigitalGlobe's GBDX Platform

Under the GEOINT Services contract, Radiant Solutions replicated a small subset of Beachfront functionality to operate on DigitalGlobe's GBDX Platform. The intent of this work was to demonstrate the feasibility of deploying the shoreline detection algorithm only to an alternate platform that was "close to the data" and able to efficiently scale to enable processing of large volumes of data.

This document is intended to summarize that effort, documenting the tasks that were developed and deployed to GBDX, and the processing workflows (recipes) that were developed and deployed to AnswerFactory.

## GBDX Tasks

Three tasks were developed to support shoreline detection in GBDX.

1. **Shoreline Detection** - The original task developed to produce shorelines from WorldView and Landsat imagery, was a close port of the Python code used in Beachfront. Supporting libraries and the shoreline algorithm itself were compiled into a Docker image and posted to the venicegeo organization on DockerHub. From there, GBDX ingests the Docker image and starts containers as needed.
2. **Otsu Thresholding** - For Sentinel-2, we opted to veer slightly from the original implementation. We can use existing GBDX tasks to compute the Modified NDWI raster. The next step in the Beachfront processing workflow is to compute the optimal Otsu threshold to binarize the NDWI result. Because this may be considered a general purpose task that others could use, we split it out as a separate GBDX task. Again, the code is derived from the original Beachfront codebase and turned into a Docker image for use within GBDX.
3. **Potrace Vectorization** - The final step in the Beachfront workflow is to vectorize the thresholded raster using potrace. Again, because this is not unique to shoreline detection, we isolated it as a separate GBDX tasks that can be reused by others.

These tasks are described in greater detail in the subsections below.

### Shoreline Detection

WorldView and Landsat imagery both use the same shoreline detection task within GBDX. The task combines all steps of the shoreline processing, including NDWI, Otsu thresholding, and potrace vectorization. A detailed description of the processing workflow is beyond the scope of this document. Suffice to say that our intent was to mirror as closely as possible the Beachfront algorithmic approach.

#### Building the Shoreline Detection Image

The shoreline detection source code can be found at https://github.com/venicegeo/shoreline-task. To build the docker image

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

### Otsu

### Potrace

## AnswerFactory Recipes

### WorldView

### Landsat

### Sentinel-2

## Source Code

## iPython/Jupyter Notebooks

* Register DGIS_Potrace_beta GBDX task
	* Creates task "DGIS_Potrace_beta"
	* Current version 0.0.1
	* Creates vector by tracing binary input raster
* Monday, 25 June 2018
	* Tasks in DGIS account (with corresponding notebooks for how created)
		* DGIS_CreateShorelineVectors_beta (was ShorelineDetection_beta)
		* DGIS_OtsuThreshold_beta (was Otsu_beta)
		* DGIS_Potrace_beta (was Potrace_beta)
	* Recipes in DGIS AnswerFactory (with corresponding notebooks)
		* dgis-create-worldview-shoreline-vectors-beta2
		* dgis-create_landsat-shoreline-vectors-beta
		* dgis-create-sentinel2-shoreline-vectors-beta