# MNIST image recognition

## Introduction

This document describes a MNINST hand written digit recognition application based on a convolutional neural network (CNN) on GAP8 processor.

In the example, 28x28 pixels images are loaded from host PC in the L2 memory of the fabric controller.
The images are then processed by the CNN on the 8 cores of the cluster. The CNN classifies the images in 10 categories (digit from 0 to 9), by selecting the maximum confidence value at the output as the inferred category.

The network topology (number of layers, number of inputs and outputs for the layers) has been taken from an open source project on the Internet. In this example, we demonstrate the implementation 
of this network on GAP8 using adapted data management through DMA and optimized parallelized filter kernels taking benefits of the 8 cores of the cluster.

The CNN has been trained off-line using the MNIST dataset. The recognition ratio reached after training is 96%, which is very close to the asymptotic ratio announced by the authors.
Actually more sophisticated topologies can achieve a better rate, but this example is given more as a simpler tutorial for CNN implementation principles on GAP8 architecture.
Specifically, it demonstrates how to perform data management between L2 memory in the fabric controller and the L1 cluster shared memory so that no memory overflow occurs while keeping all the compute resources busy.
It also shows how the basic filter kernels (2D convolution filter and linear filters) can be described to use the 8 cores of the cluster efficiently so as to reduce computation time and hence power consumption of the application.

In the next sections, we will describe the structure of the example directory, the structure of the CNN and its software components and finally how to run the application on the GAP8 simulator.

## Structure of the directory

In the GAP8 installation repository, the mnist example is located in examples/pulp-examples/autotiler_examples/Mnist

Application files:

| Name                |         Description             |
|---------------------|---------------------------------|
|Mnist.c               | Application entry code          |
|Makefile             | Makefile of the application     |
|MnistCoeffs.def      | CNN Coefficients                |
|MnistCoeffs_HWCE.def | CNN HWCE Coefficients           |
|136.ppm              | A sample input image            |
|ImgIO {.c .h}        | PPM/PGM img format handle       |
|test_img/            | Additional Test Images          |


Auto-tiler files:

| Name                           |         Description             |
|--------------------------------|---------------------------------|
|MnistModel.c                    | Auto-tiler Model                |


## Automatic Layer Code Generation

The low level code for the CNN layers has been generated using the auto-tiler tool available in the SDK.
The generator assumes that the data and weights are stored in the L2 memory (in the Fabric Controller). The available size of L1 shared memory for processing is specified to the auto-tiler.

The data and weights are split into tiles, forming  data sets that will be loaded through DMA to the L1 shared memory of the cluster. The cores will process the data and when finished, the results are stored back into the L2.
The tool takes the "high level" parameters of the layers as input and generates the low level functions describing the data management and the run-time functions to execute the layers in parallel on the eight cores of the cluster.

In MNIST example, a different version is generated for each instance, and the parameters are hard coded in the function.

The following files are automatically generated:
- MnistKernels.c	    : contains the generated code for the layers, with the calls to run time DMA process and processor synchronization
- MnistKernel.h	    : contains the header for Convlayer and dense layer functions
- MnistKernelInit.c   : contains the parameters for the DMA transfers and processing, including the tiling management and pipeline processing
- MnistKernelsInit.h  : contains data-type definition for the parameter structures

### How to automatically generate the MNIST CNN code with tiling 

The input file containing the "high level" description of the layers used in MNIST is MnistModel.c.
Following is the description of the different sections in the input file:

~~~~~c
  SetInlineMode(ALWAYS_INLINE);
~~~~~

This instruction specifies that a  function will be generated for each occurrence of the layer.
To generate a single parameterized function, the SetInlineMode function must be called with "SINGLE_INLINE" parameter.

The following instructions specify the name of the library header files (SetUsedFilesNames), the names of the generated files (SetGeneratedFilesNames), and the declaration of the amount of L1 memory to be used by the tiler generator.

~~~~~c
  SetUsedFilesNames(0, 1, "CNN_BasicKernels.h");
  SetGeneratedFilesNames("MnistKernelsInit.c", "MnistKernelsInit.h", "MnistKernels.c", "MnistKernels.h"); 
  SetL1MemorySize(L1Memory);
~~~~~

The kernel libraries are loaded for usage by the tiler:

~~~~~c
  LoadCNNLibrary();
  CNN_LoadHWCEKernelLibrary();
~~~~~

The layer parameters are defined by the code below: 

~~~~~c
  // MNIST HWCE 
  CNN_TiledConvNxNReLUPool2x2_HWCE_fp ("Conv5x5ReLUMaxPool2x2_HWCE_0", 5,  1, 32, 28, 28, 1);
  CNN_TiledConvNxNReLUPool2x2_HWCE_fp ("Conv5x5ReLUMaxPool2x2_HWCE_1", 5, 32, 64, 12, 12, 1);
  CNN_LinearLayerReLU ("LinearLayerReLU_1",   HALF_WORD,HALF_WORD,HALF_WORD,HALF_WORD, FROM_L2,FROM_L2,FROM_L2,TO_L2, 64*4*4, 10, NO_RELU);
  // MNIST SOFT
  CNN_SmallParOutFeatConvolutionPoolReLU("Conv5x5ReLUMaxPool2x2_0",     HALF_WORD,HALF_WORD,HALF_WORD,HALF_WORD, FROM_L2, FROM_L2, FROM_L2, TO_L2,  1, 32, 28, 28, 5, 1, NO_PADDING, NO_RELU, 2, 2, NO_PADDING, RELU, MAX_POOLING);
  CNN_SmallParOutFeatConvolutionPoolReLU("Conv5x5ReLUMaxPool2x2_1",     HALF_WORD,HALF_WORD,HALF_WORD,HALF_WORD, FROM_L2, FROM_L2, FROM_L2, TO_L2,  32, 64, 12, 12, 5, 1, NO_PADDING, NO_RELU, 2, 2, NO_PADDING, RELU, MAX_POOLING);
  CNN_LinearLayerReLU ("LinearLayerReLU_1",   HALF_WORD,HALF_WORD,HALF_WORD,HALF_WORD, FROM_L2,FROM_L2,FROM_L2,TO_L2, 64*4*4, 10, NO_RELU);
~~~~~

The generated files using the actual MnistModel present in the MNIST directory are generated in the example root directory.

## Compile and run the application

To execute the application on the specific image contained in RGB.img, just type the following in the MNIST directory:

~~~~~sh
	make clean all run
~~~~~

If you want to execute the code on the Hardware Convolutional Engine (HWCE) you can change this line in the Makefile:

~~~~sh
USE_HARDWARE_CE = -DRT_HAS_HWCE=1
~~~~

The output of the program is a set of 10 soft confidence values, one for each category. The max value is the category detected by the CNN.

~~~~text
Entering main controller
Image ../../../test_img/4/1301.pgm, [W: 28, H: 28], Gray, Size: 784 bytes, Loaded sucessfully
Allocating 36864: Ok
Allocating 2048: Ok
Allocating 20: Ok
Recognized: 4
Layer0:   69756 Cycles
Layer1:  747266 Cycles
Layer2:    9702 Cycles
Total:  826724 Cycles
Test success
Loop exited
commands completed
~~~~
