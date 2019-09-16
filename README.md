# Notes from our Adv CL Tutorial Session: 
09/11/2019 - Fritz, David, Monique & Kat.<br>
09/12/2019 - Updated by Kat, Fritz & David.

## What needs to happen to this file:
- Add Downloading and installing OpenVINO:  **DONE**
- Add other pertinent Intro level content - refer to previous tutorial:  **DONE**
- Find a spot for making an update and changing a parameter
- Find non-blurry ArgMax image
- Possible other customizable aspect for the reader?
- Verified for all 2019 versions of OV?  **No, only R2 on Ubuntu 16.04.  Other testing required for further verification.**
- If R4 is going to be officially supported with Ubuntu 18.04, we'll need to verify independently?

Also, David will add the nearly completed sections from David's notes: 
- edit the "front" template files
- Edit the "ops" template files

Not yet written, but David believes it's relatively easy and straightforward;
- Compile a C++
- Run the sample code

Unsure of work involved:
- Convert and Optimize an Instance Segmentation....

**END SESSION NOTES**

# OpenVINO 2019 R2.0 Custom Layer Implementation Tutorial for Linux - Advanced* 
**Note:** This tutorial has been tested and confirmed on Ubuntu 16.04 LTS using the Intel® Distribution of OpenVINO™ toolkit 2019 R2.0.  Using this tutorial with any other versions may not work correctly.

# Introduction

This advanced tutorial outlines the steps for implementing custom layers and provides examples using the Intel® Distribution of OpenVINO™ toolkit.  It assumes you have a basic working knowledge of custom layer implementation for OpenVINO. The **OpenVINO 2019 R2.0 Custom Layer Implementation Tutorial for Linux*** demonstrates the process of converting and running customer layers in the OpenVINO Inference Engine, using a simple hyperbolic cosine function as the custom layer. The cosh algorithm was chosen for simplicity of conversion, and didn't require additional parameters to be provided to the Model Optimizer.  

This more advanced tutorial demonstrates custom layer implementation using a practical real-world example: The argmax function will be used to illustrate:

- Setting up the environment
- Setting up and installing prerequisites
- Generating the Extension Template Files Using the Model Extension Generator 
- Editing the "front" template files so the Model Optimizer will know how to extract *argmax* attributes 
- Editing the "ops" template files so Inference Engine will recognize the output shape of the *argmax* layer during inference 
- Compiling a C++ library for the Inference Engine to use for calculating the *argmax* values
- Converting and Optimizing an Instance Segmentation Neural Network (NN) Topology 
- Implementing the Inference Engine extension for the example model to run on CPU and GPU 

Currently, this tutorial and the Model Extension Generator tool support creating custom layers for CPU and GPU devices.  Support for the Myriad (VPU) will be available by the end of 2019. Support for other devices is yet to be announced.

# Before You Start

## Installation of the Intel® Distribution of OpenVINO™ toolkit 2019 R2.0 for Linux* 

This tutorial assumes that you have already installed the [Intel® Distribution of OpenVINO™ toolkit 2019 R2.0 for Linux*](https://software.intel.com/openvino-toolkit/choose-download/free-download-linux) into the default */opt/intel/openvino* directory.  If you have installed the toolkit to a different directory, you will need to change the directory paths that include "*/opt/intel/openvino*" in the commands below to point to your installation directory. 

The Intel® Distribution of OpenVINO™ toolkit includes the [Model Optimizer](https://docs.openvinotoolkit.org/2019_R2/_docs_MO_DG_Deep_Learning_Model_Optimizer_DevGuide.html).  This tutorial uses a TensorFlow framework model and assumes that you have already configured the Model Optimizer for use with TensorFlow.  If you did not, be sure to follow the steps for [Configuring the Model Optimizer](https://docs.openvinotoolkit.org/2019_R2/_docs_MO_DG_prepare_model_Config_Model_Optimizer.html) before proceeding.

After installing the Intel® Distribution of OpenVINO™ toolkit, the *classification_sample_async* executable binary will be located in the directory *~/inference_engine_samples_build/intel64/Release*.  This tutorial will use the *classification_sample_async* executable to run the example model.

## Directory Layout  
Model Optimizer Custom Layer Extensions are located in this directory tree:

  `/opt/intel/openvino/deployment_tools/model_optimizer/extensions/`


There are 2 subdirectories:

  front:
    Python scripts that are used for extracting layer attributes are located here. 
    The scripts are organized into subdirectories for different frameworks: Caffe, TensorFlow, MXNet, ONNX.

  ops:
    Python scripts that tell IE how to calculate the output shape (during inference) are located here. 


# Example Custom Layer: The ArgMax Function

We will showcase the steps involved for implementing a custom layer using the *argmax* (arguments of the maxima) function.  The equation for *argmax* is:

![](pics/argmax.png)

Argmax is a layer used in some Deep Learning models.  It calculates the input value that delivers the maximum output of a function.


# Getting Started

## Setting Up the Environment

To begin, always ensure that your environment is properly setup for working with the Intel® Distribution of OpenVINO™ toolkit by running the command:

```bash
source /opt/intel/openvino/bin/setupvars.sh
```

## Installing Prerequisites

1. The Model Extension Generator makes use of *Cog* which is a content generator allowing the execution of embedded Python code to generate code within source files.  Install *Cog* (*cogapp*) using the command:

   ```bash
   sudo pip3 install cogapp
   ```

2. This tutorial will be running a Python sample from the Intel® Distribution of OpenVINO™ toolkit which needs the OpenCV library for Python to be installed.  Install the OpenCV library using the command:

   ```bash
   sudo pip3 install opencv-python
   ```

## Downloading and Setting Up the Tutorial

The first things we need to do are to create a place for the tutorial and then download it.  We will create the top directory "cl_tutorial" as the workspace to store the Git repository of the tutorial along with all the other files created.

1. Create the "cl_tutorial" top directory in the user's home directory and then change into it:
    ```bash
    cd ~
    mkdir cl_tutorial
    cd cl_tutorial
    ```
2. Download the tutorial by cloning the repository:
    ```bash
    git clone https://github.com/david-drew/cl-adv.git
    ```
3. Create some environment variables as shorter, more convenient names to the directories that will be used often:
    ```bash
    export CLWS=~/cl_adv_tutorial
    export CLT=$CLWS/2019.r1.1

    From here, we will now use "$CLWS" to reference the "cl_tutorial" workspace directory and "$CLT" to reference the directory containing the files for this tutorial.
    ```

## Download the Example Segmentation Model:

We'll download a segmentation model, then later we'll convert the model to an Intel-compatible format.

   ```bash
    mkdir $CLWS/maskrcnn
    cd $CLWS/maskrcnn
    https://github.com/matterport/Mask_RCNN/releases
    wget https://github.com/matterport/Mask_RCNN/releases/download/v2.1/balloon_dataset.zip
    wget https://github.com/matterport/Mask_RCNN/releases/download/v2.1/mask_rcnn_balloon.h5
    wget https://github.com/matterport/Mask_RCNN/archive/v2.1.tar.gz

   ```

# Creating the *argmax* Custom Layer

## Generate the Extension Template Files Using the Model Extension Generator

We will use the Model Extension Generator tool to automatically create templates for all the extensions that will be needed by the Model Optimizer to convert and the Inference Engine to execute the custom layer.  The extension template files will be partially replaced by Python and C++ code to implement the functionality of *argmax* as needed by the different tools.  To create the four extensions for the *argmax* custom layer, we run the Model Extension Generator with the following options:

- --mo-tf-ext = Generate a template for a Model Optimizer TensorFlow extractor
- --mo-op = Generate a template for a Model Optimizer custom layer operation
- --ie-cpu-ext = Generate a template for an Inference Engine CPU extension
- --ie-gpu-ext = Generate a template for an Inference Engine GPU extension
- --output_dir = set the output directory.  Here we are using *$CLWS/argmax* as the target directory to store the output from the Model Extension Generator.

To create the four extension templates for the *argmax* custom layer, we run the command:

```bash
python3 /opt/intel/openvino/deployment_tools/tools/extension_generator/extgen.py new --mo-tf-ext --mo-op --ie-cpu-ext --ie-gpu-ext --output_dir=$CLWS/argmax
```

The Model Extension Generator will start in interactive mode and prompt the user with questions about the custom layer to be generated.  Use the text between the []'s to answer each of the Model Extension Generator questions as follows:

```
Enter layer name:
[argmax]

Do you want to automatically parse all parameters from the model file? (y/n)
...
[y]

Do you want to change any answer (y/n) ? Default 'no'
[n]

Do you want to use the layer name as the operation name? (y/n)
[y]

Does your operation change shape? (y/n)
[y]

## INCOMPLETE ##

Do you want to change any answer (y/n) ? Default 'no'
[n]
```

## Edit the "front" template files so MO will know how to extract *argmax* attributes 

## Edit the "ops" template files so IE will know the output shape of the *argmax* layer during inference 

## Compile a C++ library for IE to use for calculating the *argmax* values

## Convert and Optimize an Instance Segmentation NN Topology


## Run the sample code
 

# References

[Custom Layers in the Model Optimizer](https://docs.openvinotoolkit.org/latest/_docs_MO_DG_prepare_model_customize_model_optimizer_Customize_Model_Optimizer.html)

[Custom Layers Support in Inference Engine](https://software.intel.com/en-us/articles/OpenVINO-Custom-Layers-Support-in-Inference-Engine)

[Mask R-CNN by Kaiming He, Georgia Gkioxari, Piotr Dollár, Ross Girshick](https://arxiv.org/abs/1703.06870)



