
# CIFAR10 image recognition

## Introduction

This document describes a CIFAR10 image recognition application based on a convolutional neural network (CNN) on the GAP8 processor.
The example can be compiled and executed on the virtual model of GAP8 available in the SDK.

In the example, 32x32 pixels images are stored in the L2 memory of the fabric controller.
The images are then processed by the CNN on the 8 cores of the cluster. The CNN classifies the images in 10 categories, by selecting the maximum confidence value at the output as the inferred category.

The network topology (number of layers, number of inputs and outputs for the layers) has been taken from an open source project on the Internet. In this example, we demonstrate the implementation 
of this network on GAP8 using adapted data management through DMA and optimized parallelized filter kernels taking benefits of the 8 cores of the cluster.

The CNN has been trained off-line using the CIFAR10 dataset. The recognition ratio reached after training is 68%, which is very close to the asymptotic ratio announced by the authors.
Actually more sophisticated topologies can achieve a better rate, but this example is given more as a simpler tutorial for CNN implementation principles on GAP8 architecture.
Specifically, it demonstrates how to perform data management between L2 memory in the fabric controller and the L1 cluster shared memory so that no memory overflow occurs while keeping all the compute resources busy.
It also shows how the basic filter kernels (2D convolution filter and linear filters) can be described to use the 8 cores of the cluster efficiently so as to reduce computation time and hence power consumption of the application.

In the next sections, we will describe the structure of the example directory, the structure of the CNN and its software components and finally how to run the application on the GAP8 simulator.

## Structure of the directory

In the GAP8 installation repository, the cifar10 example is located in:

  examples/pulp-examples/autotiler_examples/Cifar10


The directory contains the following files:

Application files:

| Name                |         Description             |
|---------------------|---------------------------------|
|Cifar10.c            | Application entry code          |
|Makefile             | Makefile of the application     |
|CifarCoeff.def       | CNN Coefficients                |
|CifarData.h          | Test Image                      |


Auto-tiler files:

| Name                           |         Description             |
|--------------------------------|---------------------------------|
|Cifar10Model.c                  | Auto-tiler Model                |


## Automatic layer code generation

### Cifar10 with automatic tiling generation

For this cifar10 example, the low level code for the CNN layers (convolutional and dense linear layers) has been generated using the auto-tiling tool available in this SDK.

The generator assumes that the data and weights are stored on the L2 memory (in the Fabric Controller). The available size of L1 shared memory for processing is specified to the tiler.
The data and weights are split into tiles, forming  data sets that will be loaded through DMA to the L1 shared memory of the cluster. The cores will process the data and when finished, the results are stored back into the L2.
The tool takes the "high level" parameters of the layers as input and generates the low level functions describing the data management and the sync run time functions to execute the layers in parallel on the eight cores of the cluster.

Data transfers and processing are pipelined. Doing so, the DMA processes and the data processing can be executed in parallel on different data sets to optimize the execution time.

The following files are automatically generated:
- Cifar10Kernels.c	    : contains the generated code for the layers, with the calls to run time DMA process and processor synchronization
- Cifar10Kernel.h	      : contains the header for Convlayer and dense layer functions
- Cifar10KernelInit.c   : Contains the parameters for the DMA transfers and processing, including the tiling management and pipeline processing
- Cifar10KernelsInit.h  : contains data-type definition for the parameter structures

### How to automatically generate the CIFAR10 CNN code with tiling 

The input file containing the "high level" description of the layers used in cifar10 is Model_kernel.c.

Following is the description of the different sections in the input file:

~~~~~c
  SetInlineMode(SINGLE_INLINE);
~~~~~

This instruction specifies that a single function will be generated if several occurrences of it are generated.

In this case, a structure containing the execution parameters is passed to the function.  The parameters are defined in the CnnKernelInit.c file.
To Generate one function per call the argument to pass to the SetInlineMode function is "ALWAYS_INLINE".

The following instructions specify the name of the library header files (SetUsedFIlesNames), the names of the generated files (SetGeneratedFilesNames), and the declaration of the amount of L1 memory to be used by the auto-tiler generator.

~~~~~c
  SetUsedFilesNames("KernelLibStdTypes.h",1 , "CNN_BasicKernels.h");
  SetGeneratedFilesNames("CnnKernelsInit.c", "CnnKernelsInit.h", "CnnKernels.c", "CnnKernels.h");
  
  SetL1MemorySize(L1Memory);
~~~~~

The kernel libraries are loaded for usage by the tiler:

~~~~~c
  CNN_LoadSoftwareKernelLibrary();
  CNN_LoadHWCEKernelLibrary();
~~~~~

The layer parameters are defined by the code below: 

~~~~~c
  // cifar10 config 
  CNN_TiledConvNxNReLUPool2x2_SW_fp("ConvLayer", 5, 1, 8, 32, 32, 3);
  CNN_TiledConvNxNReLUPool2x2_SW_fp("ConvLayer", 5, 8, 12, 14, 14, 3);
  CNN_TiledLinearLayer     ("Dense", 12, 5, 5, 10, 1, 0, 0);
~~~~~

## Compile and run the application

### Single execution

To execute the application on the specific image contained in RGB.img, just type the following in the cifar10 directory:

~~~~~sh
	make clean all run
~~~~~

The output of the program is a set of 10 soft confidence values, one for each category. The maximal value is the category detected by the CNN.

~~~~~text
Start execution on GAP8
FC Launched
Allocating 3136: Ok
Allocating 600: Ok
Allocating 20: Ok
Layer0:   41003 Cycles,   34691 instruments
Layer1:   82442 Cycles,   74336 instruments
Layer2:    1421 Cycles,     412 instruments
Test success
Loop exited
commands completed
~~~~~

