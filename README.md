

# Intel RealSense Depth Camera on Raspberry Pi 4B
Author: Jeff He

### Disclaimer
This guide is based on existing work from [Ai\_Demos\_RPi](https://github.com/datasith/Ai_Demos_RPi/wiki/Raspberry-Pi-4-and-Intel-RealSense-D435). While much of the original code was used, it has been updated to work in 2025, as the original version is outdated and incompatible with the latest Raspberry Pi OS. This guide is shared purely for educational purposes, detailing how I was able to get the Intel Realsense SDK and the pyrealsense2 Python package working on a Raspberry Pi 4B using an older version of Pi OS.
## Introduction

This guide provides an updated method to install the `pyrealsense2` Python package on a Raspberry Pi. `pyrealsense2` enables control of Intel RealSense D400 depth cameras. Since Intel does not officially support the RealSense SDK for Raspberry Pi, existing installation guides are outdated and do not work with current software versions. This guide aims to offer a working solution for 2024/2025.

### Tested Hardware and Software:

- **Computer:** Raspberry Pi 4B (4GB RAM)
- **Depth Cameras:** Intel RealSense D435 & D435i
- **Software Versions:**
    - Python: 3.9.2
    - Intel RealSense SDK: 2.50.0
    - Protobuf: 3.10.0
    - g++/gcc: 7.5.0

* * *

## Installation Steps

### 1. Install Raspberry Pi OS Bullseye (Release: 2021-10-30)

Download and flash the image using the [official Raspberry Pi Imager](https://www.raspberrypi.org/software/).

[Download Image](https://downloads.raspberrypi.org/raspios_full_armhf/images/raspios_full_armhf-2021-11-08/2021-10-30-raspios-bullseye-armhf-full.zip)

### 2. Update System and Install Dependencies

    sudo apt-get update
    sudo apt-get install automake libtool cmake libusb-1.0-0-dev libx11-dev xorg-dev libglu1-mesa-dev

### 3. Expand Filesystem

    sudo raspi-config

Navigate to:

    6. Advanced Options -> A1. Expand Filesystem -> Yes (Reboot)

### 4. Increase Swap Memory to 2GB

    sudo nano /etc/dphys-swapfile

Modify CONF_SWAPSIZE to 2048: 

    CONF_SWAPSIZE=2048

Then hit Ctrl + O, Enter, then Ctrl + X to save.

Apply changes:

    sudo /etc/init.d/dphys-swapfile restartswapon -s

### 5. Clone and Configure RealSense SDK

    cd ~
    git clone https://github.com/IntelRealSense/librealsense.git
    cd librealsense
    git checkout v2.50.0
    sudo cp config/99-realsense-libusb.rules /etc/udev/rules.d/

Apply the changes:

    sudo su
    udevadm control --reload-rules && udevadm trigger
    exit

### 6. Update Environment Variables

    sudo nano ~/.bashrc

Add the following line at the end:

    export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

Apply the changes:

    source ~/.bashrc

### 7. Install Required Packages

#### Install Protobuf

    cd ~
    git clone --depth=1 -b v3.10.0 https://github.com/google/protobuf.git
    cd protobuf
    ./autogen.sh
    ./configure
    make -j4
    sudo make install

Python bindings:

    cd python
    export LD_LIBRARY_PATH=../src/.libs
    python3 setup.py build --cpp_implementation
    python3 setup.py test --cpp_implementation
    sudo python3 setup.py install --cpp_implementation
    export PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=cpp
    export PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION_VERSION=3
    sudo ldconfig
    protoc --version

#### Install `libtbb-dev`

    cd ~
    wget https://github.com/PINTO0309/TBBonARMv7/raw/master/libtbb-dev_2018U2_armhf.deb
    sudo dpkg -i ~/libtbb-dev_2018U2_armhf.deb
    sudo ldconfig
    rm libtbb-dev_2018U2_armhf.deb

#### Downgrade `gcc` & `g++` to Version 7

    sudo apt install gcc-7 g++-7
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 10
    sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 10
    sudo update-alternatives --config gcc
    sudo update-alternatives --config g++
    gcc --version    # Should show up as version 7.0
    g++ --version

#### Install RealSense SDK (`librealsense`)

    cd ~/librealsense
    mkdir build && cd build
    sudo apt-get install libssl-dev
    cmake .. -DBUILD_EXAMPLES=true -DCMAKE_BUILD_TYPE=Release -DFORCE_LIBUVC=true
    make -j4
    sudo make install

#### Install `pyrealsense2` Python Bindings

    cd ~/librealsense/build
    cmake .. -DBUILD_PYTHON_BINDINGS=bool:true -DPYTHON_EXECUTABLE=$(which python3)
    make -j4
    sudo make install

Update environment variables:

    sudo nano ~/.bashrc

Add:

    export PYTHONPATH=$PYTHONPATH:/usr/local/lib

Apply changes:

    source ~/.bashrc

#### Install OpenGL and OpenCV

    sudo apt install python3-opengl
    sudo -H pip3 install pyopengl
    sudo -H pip3 install pyopengl_accelerate==3.1.3rc1
    sudo apt-get install libatlas-base-dev libhdf5-dev libhdf5-serial-dev libjasper-dev libqt5gui5 libqt5core5a libqt5widgets5 libqt5test5
    sudo apt install python3-opencv

* * *

## Running Intel RealSense SDK

After installation, you can open the RealSense viewer:

    realsense-viewer

### Using `pyrealsense2` in Python

Instead of:

    import pyrealsense2

Use:

    import pyrealsense2.pyrealsense2 as rs

### Sample Code

This script captures and saves depth and color images every X seconds:

    import pyrealsense2.pyrealsense2 as rs
    import numpy as np
    import cv2
    import time
    import os
    import json
    SAVE_INTERVAL = 30 # seconds
    SAVE_PATH = "images"
    
    pipeline = rs.pipeline()
    config = rs.config()
    config.enable_stream(rs.stream.color, 1280, 720, rs.format.bgr8, 30)
    config.enable_stream(rs.stream.depth, 1280, 720, rs.format.z16, 30)
    pipeline.start(config)
    try:
      while True: 
        frames = pipeline.wait_for_frames() 
        color_frame = frames.get_color_frame()
        depth_frame = frames.get_depth_frame()
        color_image = np.asanyarray(color_frame.get_data())
        depth_image = np.asanyarray(depth_frame.get_data())
        
        timestamp = time.strftime("%Y%m%d_%H%M%S")
        cv2.imwrite(f"{SAVE_PATH}/color_{timestamp}.png", color_image)
        cv2.imwrite(f"{SAVE_PATH}/depth_{timestamp}.png", depth_image)
        
        if cv2.waitKey(1) == ord("q"): 
          break
    finally:
      pipeline.stop()
      cv2.destroyAllWindows()

* * *

## References

- [Intel RealSense SDK for Raspberry Pi](https://github.com/IntelRealSense/librealsense/blob/master/doc/installation_raspbian.md)
- [AI Demos on Raspberry Pi](https://github.com/datasith/Ai_Demos_RPi/wiki/Raspberry-Pi-4-and-Intel-RealSense-D435)
