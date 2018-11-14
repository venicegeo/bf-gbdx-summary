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

GBDX tasks alone are building blocks for processing workflows. We've already mentioned one such workflow that uses multiple GBDX tasks, the Sentinel-2 shoreline extraction. Even for WorldView and Landsat, there are steps we have not yet mentioned, from ordering and preprocessing imagery to posting results to S3. AnswerFactory, also part of the GBDX platform, allows us to define such workflows (called recipes) such that users can define an area of interest and launch a predefined workflow, in much the same way they would currently use Beachfront. This section documents three such workflows, one each for WorldView, Landsat, and Sentinel-2.

### WorldView

The full WorldView shoreline detection recipe is defined by the following JSON.

```json
{
    "id": "dgis-create-worldview-shoreline-vectors-beta2",
    "version": "0.1.0",
    "name": "Create WorldView Shoreline Vectors beta2",
    "owner": validation_response.json()['username'],
    "accountId": [
        validation_response.json()['account_id']
    ],
    "description": "Create shoreline vectors using WorldView imagery.",
    "recipeType": "workflow",
    "inputType": "acquisition",
    "outputType": "vector-service",
    "properties": {
        "image_bands": "Pan_MS1_MS2",
        "crop_to_project": True
    },
    "definition": {
        "name": "Create WorldView Shoreline Vectors",
        "tasks": [
            {
                "name": "AOP_Strip_Processor_e5f9fd90",
                "taskType": "AOP_Strip_Processor:0.0.4",
                "impersonation_allowed": True,
                "containerDescriptors": [],
                "inputs": [
                    {
                        "name": "data",
                        "value": "{raster_path}"
                    },
                    {
                        "name": "enable_pansharpen",
                        "value": "false"
                    },
                    {
                        "name": "ortho_epsg",
                        "value": "UTM"
                    },
                    {
                        "name": "enable_acomp",
                        "value": "true"
                    },
                    {
                        "name": "bands",
                        "value": "MS"
                    },
                    {
                        "name": "enable_dra",
                        "value": "false"
                    }
                ],
                "outputs": [
                    {
                        "name": "log"
                    },
                    {
                        "name": "data"
                    }
                ]
            },
            {
                "name": "DGIS_CreateShorelineVectors_beta",
                "taskType": "DGIS_CreateShorelineVectors_beta:0.0.1",
                "timeout":7200,
                "impersonation_allowed":True,
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"image",
                        "source":"AOP_Strip_Processor_e5f9fd90:data"
                    },
                    {
                        "name":"cat_id",
                        "value":"{acquisition_id}"
                    }
                ],
                "outputs":[
                    {
                        "name":"vector"
                    },
                    {
                        "name":"raster"
                    }
                ]
            },
            {
                "name":"ingest_DGIS_CreateShorelineVectors_beta",
                "taskType":"IngestGeoJsonToVectorServices",
                "impersonation_allowed":True,
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"items",
                        "source":"DGIS_CreateShorelineVectors_beta:vector"
                    },
                    {
                        "name":"mapping",
                        "value":"vector.crs=EPSG:4326\\nvector.ingestSource=Create WorldView Shoreline Vectors\\nvector.itemType=0"
                    }
                ],
                "outputs":[
                    {
                        "name":"result"
                    }
                ]
            },
            {
                "name":"s3_ingest_DGIS_CreateShorelineVectors_beta",
                "taskType":"StageDataToS3",
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"data",
                        "source":"ingest_DGIS_CreateShorelineVectors_beta:result"
                    },
                    {
                        "name":"destination",
                        "value":"s3://{vector_ingest_bucket}/{recipe_id}/{run_id}/{task_name}"
                    }
                ],
                "outputs":[]
            }
        ]
    }
}
```

Again, a full description of AnswerFactory is beyond the scope of this document, but a few details to highlight are that the workflow definition is comprised of the following basic steps:

1. As a prerequisite, the source data must be `Pan_MS1_MS2` and will be cropped to the area of interest.
2. **AOP Strip Processor** - Used to atmospherically compensate the input data.
3. **DGIS Shoreline Detection Beta** - Our previously documented shoreline detection tasks, which takes as input atmospherically compensated multispectral bands and outputs shoreline vectors.
4. **Ingest GeoJSON to Vector Services** - Makes vector results available to AnswerFactory.
5. **Stage Data to S3** - Stages results to S3.

### Landsat

The full Landsat shoreline detection recipe is defined by the following JSON.

```json
{
    "id": "dgis-create-landsat-shoreline-vectors-beta",
    "version": "0.1.0",
    "name": "Create Landsat Shoreline Vectors beta",
    "owner": validation_response.json()['username'],
    "accountId": [
        validation_response.json()['account_id']
    ],
    "description": "Create shoreline vectors using Landsat imagery.",
    "recipeType": "workflow",
    "inputType": "acquisition",
    "outputType": "vector-service",
    "properties": {
        "acquisition_types": "LandsatAcquisition",
        "crop_to_project": True
    },
    "definition": {
        "name": "Create WorldView Shoreline Vectors",
        "tasks": [
            {
                "name": "DGIS_CreateShorelineVectors_beta",
                "taskType": "DGIS_CreateShorelineVectors_beta:0.0.1",
                "timeout":7200,
                "impersonation_allowed":True,
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"image",
                        "value":"{raster_path}"
                    },
                    {
                        "name":"cat_id",
                        "value":"{acquisition_id}"
                    }
                ],
                "outputs":[
                    {
                        "name":"vector"
                    },
                    {
                        "name":"raster"
                    }
                ]
            },
            {
                "name":"ingest_DGIS_CreateShorelineVectors_beta",
                "taskType":"IngestGeoJsonToVectorServices",
                "impersonation_allowed":True,
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"items",
                        "source":"DGIS_CreateShorelineVectors_beta:vector"
                    },
                    {
                        "name":"mapping",
                        "value":"vector.crs=EPSG:4326\\nvector.ingestSource=Create WorldView Shoreline Vectors\\nvector.itemType=0"
                    }
                ],
                "outputs":[
                    {
                        "name":"result"
                    }
                ]
            },
            {
                "name":"s3_ingest_DGIS_CreateShorelineVectors_beta",
                "taskType":"StageDataToS3",
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"data",
                        "source":"ingest_DGIS_CreateShorelineVectors_beta:result"
                    },
                    {
                        "name":"destination",
                        "value":"s3://{vector_ingest_bucket}/{recipe_id}/{run_id}/{task_name}"
                    }
                ],
                "outputs":[]
            }
        ]
    }
}
```

The key differences between the Landsat workflow and the WorldView workflow are that the acquisition type is set specifically to `LandsatAcquisition` and there is no AOP Strip Processor task for Landsat.

1. As a prerequisite, the acquisition type must be `LandsatAcquisition` and will be cropped to the area of interest.
2. **DGIS Shoreline Detection Beta** - Our previously documented shoreline detection tasks, which takes as input atmospherically compensated multispectral bands and outputs shoreline vectors.
3. **Ingest GeoJSON to Vector Services** - Makes vector results available to AnswerFactory.
4. **Stage Data to S3** - Stages results to S3.

### Sentinel-2

The full JSON payload to register the Sentinel-2 workflow is given below.

```json
{
    "id": "dgis-create-sentinel2-shoreline-vectors-beta",
    "version": "0.1.2",
    "name": "Create Sentinel-2 Shoreline Vectors beta",
    "owner": validation_response.json()['username'],
    "accountId": [
        validation_response.json()['account_id']
    ],
    "description": "Create shoreline vectors using Sentinel-2 imagery.",
    "recipeType": "workflow",
    "inputType": "acquisition",
    "outputType": "vector-service",
    "properties": {
        "acquisition_types": "SENTINEL2",
        "crop_to_project": True
    },
    "definition": {
        "name": "Create Sentinel-2 Shoreline Vectors",
        "tasks": [
            {
                "name": "convert",
                "taskType": "gdal-cli-multiplex:0.0.1",
                "timeout": 36000,
                "inputs": [
                    {
                        "name": "data",
                        "value": "{raster_path}"
                    },
                    {
                        "name": "command",
                        "value": "mkdir -p $outdir/data; gdal_translate -of GTiff -a_nodata 0 $indir/data/B03.jp2 $outdir/data/B03.tif; gdal_translate -of GTiff -a_nodata 0 $indir/data/B11.jp2 $outdir/data/B11.tif; gdalwarp -tr 10 10 $outdir/data/B11.tif $outdir/data/output.tif"
                    }
                ],
                "outputs": [
                    {
                        "name": "data"
                    }
                ]
            },
            {
                "name": "crop",
                "taskType": "CropGeotiff",
                "timeout": 36000,
                "inputs": [
                    {
                        "name": "data",
                        "source": "convert:data"
                    },
                    {
                        "name": "no_data",
                        "value": "0"
                    }
                ],
                "outputs": [
                    {
                        "name": "data"
                    }
                ]
            },
            {
                "name": "convert2",
                "taskType": "gdal-cli-multiplex:0.0.1",
                "timeout": 36000,
                "inputs": [
                    {
                        "name": "data",
                        "source": "crop:data"
                    },
                    {
                        "name": "command",
                        "value": "mkdir -p $outdir/data; gdal_translate -ot Float32 $indir/data/B03.tif $outdir/data/B03.tif; gdal_translate -ot Float32 $indir/data/output.tif $outdir/data/B11.tif"
                    }
                ],
                "outputs": [
                    {
                        "name": "data"
                    }
                ]
            },
            {
                "name": "mndwi_tif",
                "taskType": "gdal-cli-multiplex:0.0.1",
                "timeout": 36000,
                "inputs": [
                    {
                        "name": "data",
                        "source": "convert2:data"
                    },
                    {
                        "name": "command",
                        "value": "mkdir -p $outdir/data; gdal_calc.py -A $indir/data/B03.tif -B $indir/data/B11.tif --outfile=$outdir/data/mndwi.tif --calc=\"(A-B)/(A+B)\";"
                    }
                ],
                "outputs": [
                    {
                        "name": "data"
                    }
                ]
            },
            {
                "name": "otsu",
                "taskType": "DGIS_OtsuThreshold_beta:0.0.1",
                "timeout": 7200,
                "impersonation_allowed": True,
                "containerDescriptors": [],
                "inputs": [
                    {
                        "name": "image",
                        "source": "mndwi_tif:data"
                    }
                ],
                "outputs": [
                    {
                        "name": "threshold"
                    }
                ]
            },
            {
                "name": "potrace",
                "taskType": "DGIS_Potrace_beta:0.0.1",
                "timeout": 7200,
                "impersonation_allowed": True,
                "containerDescriptors": [],
                "inputs": [
                    {
                        "name": "threshold",
                        "source": "otsu:threshold"
                    },
                    {
                        "name": "image",
                        "source": "mndwi_tif:data"
                    }
                ],
                "outputs": [
                    {
                        "name": "result"
                    }
                ]
            },
            {
                "name":"ingest_DGIS_Potrace_beta",
                "taskType":"IngestGeoJsonToVectorServices",
                "impersonation_allowed":True,
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"items",
                        "source":"potrace:result"
                    },
                    {
                        "name":"mapping",
                        "value":"vector.crs=EPSG:4326\\nvector.ingestSource=:{recipe_name}\\nvector.itemType=0"
                    }
                ],
                "outputs":[
                    {
                        "name":"result"
                    }
                ]
            },
            {
                "name":"s3_ingest_DGIS_Potrace_beta",
                "taskType":"StageDataToS3",
                "containerDescriptors":[],
                "inputs":[
                    {
                        "name":"data",
                        "source":"ingest_DGIS_Potrace_beta:result"
                    },
                    {
                        "name":"destination",
                        "value":"s3://{vector_ingest_bucket}/{recipe_id}/{run_id}/{task_name}"
                    }
                ],
                "outputs":[]
            }
        ]
    }
}
```

This workflow is significantly more complex than the earlier examples. First, a number of new tasks are inserted (using GDAL command line tools) to reformat the Sentinel-2 data and to compute the Modified NDWI raster. From there, we proceed to call the Otsu threshold detection and Potrace vectorization tasks that were explained earlier in the document. The final result is still ingest into Vector Services and staged to S3.