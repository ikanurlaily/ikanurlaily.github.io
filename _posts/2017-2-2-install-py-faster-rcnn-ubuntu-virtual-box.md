---
layout: article
title: Install py-faster-rcnn Ubuntu on Virtual Box
share: true
modified: 2017-02-02T14:18:57-04:00
image:
    teaser: faster-rcnn.png
categories: articles
---

This is how I install `py-faster-rcnn`, up to running `demo.py` to make sure the installation is succesful. Since we can't virtualize graphic device, the library will be **running on CPU**, so there are bunch of modifications here. Replace any `<user>` below with your real username.

### **Hardware / Software**

I used **Dell Aspire E 15** running **NVIDIA GeForce 940MX** with 2 GB dedicated VRAM. The **Virtual Box** version is  5.1.14 r112924 with **Ubuntu 16.04** installed in it (2GB RAM, 16GB VDI). I have successfully re-run these steps on Microsoft Surface Pro 3 with the same VBox and Ubuntu..

---

## **0. Virtual Box Configuration**

#### 0.1. Update Ubuntu
After Ubuntu was installed, install updates, build essentials, and latest version of kernel headers.

```
sudo apt-get updates

sudo apt-get install build-essential

sudo apt-get install linux-headers-`uname -r`
```

#### 0.2. Install Virtual Box Additions:

Click menu `Devices` then click `Insert Guest Addition CD Image ..`. The virtual CD drive will be mounted. Then in your terminal:

```
cd /media/<user>/VBOXADDITIONS_5.1.14_112924 

sudo ./VBoxLinuxAdditions.run
```

to install the Virtual Box additions. The name of the virtual CD drive may vary depends on your Virtual Box version. **Restart** your Ubuntu to make sure the additions are installed perfectly.

---

##  **1. Install Dependencies**
Basic packages:

```
sudo apt-get install -y build-essential cmake git pkg-config
```

Caffe dependencies:

```
sudo apt-get install -y libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libboost-all-dev libhdf5-serial-dev protobuf-compiler gfortran libjpeg62 libfreeimage-dev libatlas-base-dev libgoogle-glog-dev libbz2-dev libxml2-dev libxslt-dev libffi-dev libssl-dev libgflags-dev liblmdb-dev

sudo easy_install pillow
```

Python:

```
sudo apt-get install -y python-dev python-pip python-yaml python-opencv
```

---

## **2. Install CUDA**

#### 2.1. Donwload CUDA
Download [CUDA here](https://developer.nvidia.com/cuda-downloads) by selecting your platform. I used **CUDA 8.0** and chose `runfile (local)` at the last option to get the `.run` file.

#### 2.2. Run the CUDA installer
Make the downloaded file runnable first

```
cd ~/Downloads

chmod +x cuda_8.0.44_linux.run
```

Then run the installer

``` 
sudo ./cuda_8.0.44_linux.run --kernel-source-path=/usr/src/linux-headers-`uname -r`/
```

Hit `Enter` up until the option appears:

- Accept EULA: type `accept`
- **Do not** install graphic card drivers since we don't have graphic card in our Virtual Box:  type `n`
- Install toolkit: type `y`
- Enter toolkit location: jus tpress `enter`
- Install symbolic link: type `y`
- Install samples: type `y`
- Enter CUDA samples location, in my case I leave it as is: press `enter`

Update the library path:

```
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc

echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/usr/local/lib' >> ~/.bashrc

source ~/.bashrc
```

---

## **3. Download `py-faster-rcnn`**
Now the easiest part, download the `py-faster-rcnn`

```
cd ~

git clone --recursive https://github.com/rbgirshick/py-faster-rcnn.git
```

Then create enviromental variables for `py-faster-rcc` and `caffe-fast-rcnn` folder paths.

```
echo 'export FRCN=/home/<user>/py-faster-rcnn' >> ~/.bashrc

echo 'export CAFFE=/home/<user>/py-faster-rcnn/caffe-fast-rcnn' >> ~/.bashrc

source ~/.bashrc
```

---

## **4. Compile Caffe and Pycaffe**
Here comes the most annoying part because I had to do lots of adjustment back and forth, before making it work. 

#### 4.1. Install python requirements

```
cd $CAFFE/python

for req in $(cat requirements.txt); do pip install $req; done

pip install easydict
```

Add couple of symbolic links


```
sudo ln -s /usr/include/python2.7/ /home/<user>/.local/lib/python2.7 

sudo ln -s /home/<user>/.local/lib/python2.7/site-packages/numpy/core/include/numpy/
```

#### 4.2 Edit Makefile config
In the caffe root folder (`$CAFFE`), copy the Makefile.config example


```
cp Makefile.config.example Makefile.config

gedit Makefile.config
```

Then edit some necessary parts:

- uncomment `CPU_ONLY` and `WITH_PYTHON_LAYER`

```
# CPU-only switch (uncomment to build without GPU support). 
CPU_ONLY := 1
```

```
WITH_PYTHON_LAYER := 1
```

- edit `PYTHON_INCLUDE`, `INCLUDE_DIRS`, `LIBRARY_DIRS`, and `CUDA_DIR`

```
PYTHON_INCLUDE := /usr/include/python2.7 /home/<user>/.local/lib/python2.7/site-packages/numpy/core/include
```

```
INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/local/include /usr/include/hdf5/serial
```

```
LIBRARY_DIRS := $(PYTHON_LIB) xcd
```

```
CUDA_DIR := /usr/local/cuda-8.0 
```

#### 4.3. Build caffe
Make sure you are inside the caffe root in your terminal, then

```
cd $CAFFE

make pycaffe

make all
```

---

## **5. Build cython modules**

#### 5.1.  Edit `config.py`

```
gedit $FRCN/lib/fast_rcnn/config.py
``` 

then set `USE_GPU_NMS` to `FALSE`.

#### 5.2. Edit `test_net` and `train_net`

```
gedit $FRCN/tools/test_net.py

gedit $FRCN/tools/train_net.py
``` 

In **both files** replace `caffe.set_mode_gpu()` with `caffe.set_mode_cpu()`.

#### 5.3. Edit `setup.py`

```
cd $FRCN/lib

cp setup.py setup-original.py

gedit setup.py
```
**DELETE** or comment these whole rows

```
Extension('nms.gpu_nms',
        ['nms/nms_kernel.cu', 'nms/gpu_nms.pyx'],
        library_dirs=[CUDA['lib64']],
        libraries=['cudart'],
        language='c++',
        runtime_library_dirs=[CUDA['lib64']],
        # this syntax is specific to this build system
        # we're only going to use certain compiler args with nvcc and not with
        # gcc the implementation of this trick is in customize_compiler() below
        extra_compile_args={'gcc': ["-Wno-unused-function"],
                            'nvcc': ['-arch=sm_35',
                                     '--ptxas-options=-v',
                                     '-c',
                                     '--compiler-options',
                                     "'-fPIC'"]},
        include_dirs = [numpy_include, CUDA['include']]
    ),
```

#### 5.4. Modify `nms_wrapper.py`

```
gedit $FRCN/lib/fast_rcnn/nms_wrapper.py
```

Comment these lines:

```
#from nms.gpu_nms import gpu_nms

    #if cfg.USE_GPU_NMS and not force_cpu:
    #    return gpu_nms(dets, thresh, device_id=cfg.GPU_ID)
```

#### 5.5. Compile the libraries

```
cd $FRCN/lib

make
```

---

## **6. Run Demo**
If you are not getting any error from the previous steps, or finally able to figure them out and debug your errors, congratulations! Now let's run the demo.

#### 6.1. Download pre-computed Faster R-CNN detectors

```
cd $FRCN

./data/scripts/fetch_faster_rcnn_models.sh
```

#### 6.2. Run demo

```
cd $FRCN/tools

python demo.py --cpu --net zf
```

If you are able to run the demo smoothly, it will show something like this
![Success demo](https://lh3.googleusercontent.com/ya-SiQcQCe9eHCDgyI6iQjwWUcVn1HAKiOqcIN1ULbKW9aHimXuIowHIwbNxeRyqsF2YhN9xnao1rBd87H5uYn81H9kJZte8vaZI8e2qI5PBZD9Z0VJdmnlRKXsk485PCBZ9NHTgoNDn8ijvDal3tiIlY_ZEjtHwOwKoJU57wokpwYWi4rvNIzLIRhdgH-a0b4Dq72JqGoFglcUwtVx7mYsK73YDn7BVwfeb9tWgr-xT9nXE7sIGlcPpdTIXQ9CuBPU0xSz-_k2PVtqsWv1F0yR3qLdH7SQ2KDq4XH0pm6YRvSBDXcLH2qPs0voAfxwCkcpxIXfPHJvpIHVekgbGFSs80ecyWVssZ8muVr3nEkkT-U-iU4VLEil_IaR8l7njvwr4-B_ojwUUs-lY0KrlsfnVmT9M0u3do9feoDvzGc0xm7m2ra8n-MpshdyLnbpK6g3lNdBQueLLAu_p_rTCIKyIIA-yZvkvJVLZRqzF0pwXpfQSFepygjrHgaLxNV7NiSxavq3aYvA3zb4cggzfcdP8baXhrGRzRecI_IIfnLsjV6jRW2JbDwsGHoQFqXgU1JuQmu9y=w1920-h918)

### Good luck!

## References
- [Ubuntu 14.04 VirtualBox VM](https://github.com/BVLC/caffe/wiki/Ubuntu-14.04-VirtualBox-VM)
- [py-faster-rcnn GitHub page](https://github.com/rbgirshick/py-faster-rcnn)
- [Ubuntu 16.04 or 15.10 Installation Guide](https://github.com/BVLC/caffe/wiki/Ubuntu-16.04-or-15.10-Installation-Guide)
- [How to setup py-faster-rcnn with CPU only](https://github.com/rbgirshick/py-faster-rcnn/issues/123)
