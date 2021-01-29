# DeepThings
DeepThings is a framework for locally distributed and adaptive CNN inference in resource-constrained IoT edge clusters. DeepThings mainly consists of:
- A Fused Tile Partitioning (FTP) method for dividing convolutional layers into independently distributable tasks. FTP fuses layers and partitions them vertically
in a grid fashion, which largely reduces communication and task migration overhead.
- A distributed work stealing runtime system for IoT clusters to adaptively distribute FTP partitions in dynamic application scenarios.

For more details of DeepThings, please refer to [1].

<div align="center">
  <img src="https://zoranzhao.github.io/images/deepthings.png" width="400px" />
  <p>Overview of the DeepThings framework.</p>
</div>

This repository includes a lightweight, self-contained and portable C implementation of DeepThings. It uses a [NNPACK](https://github.com/digitalbrain79/NNPACK-darknet)-accelerated [Darknet](https://github.com/UT-AFC/darknet-nnpack) as the default inference engine. More information on porting DeepThings with different inference frameworks and platforms can be found below. 

## Platforms and Prerequisites
The current implementation has been tested on [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) running [Raspbian](https://www.raspberrypi.org/downloads/raspbian/). Our network setup consisted of six edge devices and a single gateway device, although more edge devices are possible.

Deepthings also runs on top of the darknet

## Compiling
Edit the configuration file [include/configure.h](https://github.com/UT-AFC/DeepThings/blob/master/include/configure.h) according to your IoT cluster parameters. 

Note: If the start signal is desired to come from the gateway rather than a host machine, set GATEWAY_PUBLIC_ADDR to the local ip address of the gateway. If the start signal comes from a host machine, then set GATEWAY_PUBLIC_ADDR to the public ip address.

Then, run:
```bash
make clean
make 

```
This will automatically compile all related libraries and generate the DeepThings executable. If you want to run DeepThings on Raspberry Pi with NNPACK acceleration, you need first follow install [NNPACK](https://github.com/zoranzhao/darknet-nnpack/blob/2f2da6bd46b9bbfcd283e0556072f18581392f08/README.md) before running the Makefile commands, and set the options in Makefile as below:
```
NNPACK=1
ARM_NEON=1
```

## Downloading pre-trained CNN models and input data
In order to perform distributed inference, you need to download pre-trained CNN models and put it in [./models](https://github.com/UT-AFC/DeepThings/tree/master/models) folder.
Current implementation is tested with YOLOv2, which can be downloaded from [YOLOv2 model](https://github.com/UT-AFC/DeepThings/blob/master/models/yolo.cfg) and [YOLOv2 weights](https://pjreddie.com/media/files/yolo.weights). If the link doesn't work, you can also find the weights [here](https://utexas.box.com/s/ax7f0j0qwnc4yb9ghjprjd93qwk3t4uw).

For input data, images need to be numbered (starting from 0) and renamed as <#>.jpg, and placed in [./data/input](https://github.com/UT-AFC/DeepThings/tree/master/data/input) folder.

## Running in a IoT cluster
An overview of DeepThings command line options is listed below:
```bash
#./deepthings -mode <execution mode: {start, gateway, data_src, non_data_src}> 
#             -total_edge <total edge number: t> 
#             -edge_id <edge device ID: {0, ... t-1}>
#             -n <FTP dimension: N> 
#             -m <FTP dimension: M> 
#             -l <number of fused layers: L>
```

The mode option will vary depending on device. In each configuration, at least one device will need to be a data source edge device, capable of loading and partitioning images. More than one data source is possible, with each one loading different images to be processed by the gateway. Exactly one device will need to be the gateway: the final device which will classify and label each image. An optional number of non-data source edge devices is also possible, which will attempt to steal running jobs from the data sources to speed up computation.

For example, assuming you have a host machine H, gateway device G, and two edge devices E0 (data source) and E1 (idle), while 
you want to perform a 5x5 FTP with 16 fused layers, then you need to follow the steps below:

In gateway device G:
```bash
./deepthings -mode gateway -total_edge 2 -n 5 -m 5 -l 16
```
In edge device E0:
```bash
./deepthings -mode data_src -edge_id 0 -n 5 -m 5 -l 16
```
In edge device E1:
```bash
./deepthings -mode non_data_src -edge_id 1 -n 5 -m 5 -l 16
```
Now all the devices will wait for a trigger signal to start. It is important to wait for each device to load the weights and be awaiting the trigger signal before issuing it (this shouldn't take more than a few seconds). Once ready, you can simply signal from the gateway: 
```bash
./deepthings -mode start
```

If a error is experienced, it is best to exit all running instances of DeepThings on all devices (making sure none are left running in the background). Make sure the weights are loaded before restarting. Additionally, it may help to check that all configure files are the identical and remake if necessary.

## Running in a single device
Many people want to first try the FTP-partitioned inference in a single device. Now you can find a single-device execution example in [./examples](https://github.com/UT-AFC/DeepThings/tree/master/examples) folder. To run it:
```bash
cd examples
make clean
make
./deepthings
```

This will first initialize a gateway context and a client context in different local threads. FTP partition inference results will be transferred between queues associated with each context to emulate the inter-device communication.

## Use of Images
The images used in the examples are found in [./data/input/](https://github.com/UT-AFC/DeepThings/blob/master/data/input) and must be numbered sequentially starting from 0 and stored with JPEG compression. The number of images that the example will run through is determined by the FRAME_NUM variable in [./include/configure.h](https://github.com/UT-AFC/DeepThings/blob/master/include/configure.h). There are 6 example images, however these can be changed to other images by following the naming convention present. Upon completion, the labeled images will be outputted into the same folder as the executable as PNG files.

## Data Reuse
DeepThings also has the capability to reuse some of the data generated by each task in the queue for greater efficiency. The gateway will manage all reuse data that has been accumulated using an Overlapped Data Pool. Should an edge device (data source or non data source) receive a task from the queue to run, the edge device will request reuse data from the gateway. Should there be data available, the device will use it, or the device will proceed without the additional data. After the processed task is finished, it will send the reuse data back to the gateway. The data reuse functionality can be toggled via the DATA_REUSE flag in [.include/configuration.h](https://github.com/UT-AFC/DeepThings/blob/master/include/configure.h). The functions in the code addressing data reuse will not be compiled due to pre-processor directives, and cannot be toggled once compiled.

## Code Structure
Directory Tree<br>
├── darknet-nnpack<br>
│   ├── cfg<br>
│   │   └── yolov1<br>
│   ├── data<br>
│   │   └── labels<br>
|   ├── examples<br>
│   ├── include<br>
│   ├── obj<br>
│   ├── python<br>
│   ├── results<br>
│   ├── scripts<br>
│   └── src<br>
├── data<br>
│   ├── input<br>
│   └── labels<br>
├── distriot<br>
│   ├── examples<br>
│   ├── include<br>
│   ├── obj<br>
│   └── src<br>
├── examples<br>
│   ├── data<br>
│   │   ├── input<br>
│   │   └── labels<br>
│   ├── obj<br>
│   └── profile<br>
│       └── data<br>
├── include<br>
├── models<br>
├── obj<br>
└── src<br>

DeepThings is organized into a set of high-level function files found in the [./src](https://github.com/UT-AFC/DeepThings/blob/master/src) folder. Additional helper functions are found in [./distriot/src/](https://github.com/UT-AFC/DeepThings/blob/master/distriot/src). These include items such as code for low-level communication and FIFO queue structure and operations.

## Porting DeepThings
One just needs to simply modify the corresponding abstraction layer files to port DeepThings.
If you want to use a different CNN inference engine, modify: 
- [src/inference_engine_helper.c](https://github.com/UT-AFC/DeepThings/blob/master/src/inference_engine_helper.c)
- [src/inference_engine_helper.h](https://github.com/UT-AFC/DeepThings/blob/master/src/inference_engine_helper.h)

If you want to port DeepThings onto a different OS (Currently using UNIX pthread), modify: 
- [distriot/src/thread_util.c](https://github.com/UT-AFC/DeepThings/blob/master/distriot/src/thread_util.c)
- [distriot/src/thread_util.h](https://github.com/UT-AFC/DeepThings/blob/master/distriot/src/thread_util.h)

If you want to use DeepThings with different networking APIs (Currently using UNIX socket), modify: 
- [distriot/src/network_util.c](https://github.com/UT-AFC/DeepThings/blob/master/distriot/src/network_util.c)
- [distriot/src/network_util.h](https://github.com/UT-AFC/DeepThings/blob/master/distriot/src/network_util.h)


## References:
[1] Z. Zhao, K. Mirzazad and A. Gerstlauer, "[DeepThings: Distributed Adaptive Deep Learning Inference on Resource-Constrained IoT Edge Clusters](https://zoranzhao.github.io/docs/deepthings_2018.pdf)," CODES+ISSS 2018, special issue of IEEE Transactions on 
Computer-Aided Design of Integrated Circuits and Systems (TCAD).

## Contact:
Jackson Farley, <jackson_farley@utexas.edu>
