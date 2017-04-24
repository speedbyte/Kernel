# Kernel Compiling
Here you can find the raspberry kernel source code and following is reported the guide about how to compile it.

To differenciate the commands to be run on the computer and the ones on the raspberry, they will be differentiated as:
    ```
    user@host ~/linux$      Computer
    pi@raspberry$           Raspberry
    ```

## Download the kernel
Clone the kernel git repository into the helikopter-kernel directory
```
user@host ~ git clone -b rpi-4.9.y https://github.com/raspberrypi/linux/
```

## Compile

1. Define the path where you stored the repository (the tools and the linux folder) in the RPATH variable:
    ```
    user@host ~$ export RPATH_TOOLS=global_path_helikopter-tools
    user@host ~$ export RPATH_LINUX=global_path_helikopter-kernel/linux
   	```

2. Define then the other path and variable we would need
	
	if you are using a 64bit computer:
    ```
	user@host ~$ export CROSS_COMPILE=${RPATH_TOOLS}/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-
	```
	otherwise:
	```
	user@host ~$ export CROSS_COMPILE=${RPATH_TOOLS}/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian/bin/arm-linux-gnueabihf-
	```
    
    ```
	user@host ~$ export INSTALL_MOD_PATH=${RPATH_LINUX}/../kernel`
	user@host ~$ export ARCH=arm
	user@host ~$ export KERNEL=kernel
	user@host ~$ cd ${RPATH_LINUX}
	user@host ~/linux$ mkdir ../kernel
	```

3. Install the patch for the RealTime Kernel
	```
	user@host ~/linux$ cp ../patch patch-4.9.20-rt16.patch.gz .
    user@host ~/linux$ zcat patch patch-4.9.20-rt16.patch.gz | patch -p1
    ```

4. Now you can decide whether take the configuration file from the kernel altrady installed in the raspberry or create a new one

	To use an old configuration file you need first to generate it on the raspberry and then copy it in the computer where you want to make the compiling
	```
	> pi@raspberry$ sudo modprobe configs
	> user@host ~/linux$ scp pi@raspberry:/proc/config.gz ./
	> user@host ~/linux$ zcat config.gz > .config
	> user@host ~/linux$ make oldconfig
	```

	To create a new the .config file
	```
	user@host ~/linux$ make bcmrpi_defconfig
	user@host ~/linux$ make menuconfig
	```

5. During the configuration of the kernel we need to define a couple of parameters in order to get the Fully Preemptible Kernel.
In the menu of the configuration file, modify the following parameters:
    ```
	CONFIG_PREEMPT_RT_FULL: Kernel Features → Preemption Model (Fully Preemptible Kernel (RT)) → Fully Preemptible Kernel (RT)
	Enable HIGH_RES_TIMERS: General setup → Timers subsystem → High Resolution Timer Support (Actually, this should already be enabled in the standard configuration.)
	```

6. Now we can compile the kernel, generate the modules and the data tree files and install the modules in the directory we defined before in the INSTALL_MOD_PATH variable
    ```
	make zImage modules dtbs modules_install
	```

7. At this point we have to collect all the file necessary to be copied in the raspberry

	Create a folder which would contain the files to be copied in the boot partition
	```
	mkdir $INSTALL_MOD_PATH/boot
	```

	Generate the kernal image and copy the data tree files
	```
	./scripts/mkknlimg ./arch/arm/boot/zImage $INSTALL_MOD_PATH/boot/$KERNEL.img
	cp ./arch/arm/boot/dts/*.dtb $INSTALL_MOD_PATH/boot/
	cp -r ./arch/arm/boot/dts/overlays $INSTALL_MOD_PATH/boot
	```

	Compress all the generated files and copy them into the raspberry
	```
	cd $INSTALL_MOD_PATH
	tar czf kernel.tgz *
	scp kernel.tgz pi@raspberry:/tmp
	```

8. Finally from the raspberry we need to decompress the tarball to install the new kernel
	```
    pi@raspberry ~$ tar xzf tmp/kernel.tgz
    ```

9. Reboot your raspberry and you will have the new kernel installed