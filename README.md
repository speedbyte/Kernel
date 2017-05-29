Upload kernel.tar.gz

Upload helikopter SD card image.

Upload the config file and System.map 

# Kernel Compiling
Here you can find the raspberry kernel source code and following is reported the guide about how to compile it.

To differenciate the commands to be run on the computer and the ones on the raspberry, they will be defined respectively as:
    ```
    user@host ~/linux$      Computer
    pi@raspberry$           Raspberry
    ```

## Download the kernel
Clone the kernel git repository into the helikopter-kernel directory
```
user@host ~ git clone -b rpi-4.9.y https://github.com/raspberrypi/linux/
```

## Download the Real Time patch
We are using the Linux RT Preempt patch version 4.9.20-rt16
```
user@host ~ wget -P ~/linux/ https://www.kernel.org/pub/linux/kernel/projects/rt/4.9/older/patch-4.9.20-rt16.patch.gz
```

## Compile

1. Define the path where you stored the repositories (the tools for the compiling and the linux kernel source folder) in the RPATH variable:
    ```
    user@host ~$ export RPATH_TOOLS=global_path_helikopter-tools
    user@host ~$ export RPATH_LINUX=global_path_helikopter-kernel/linux
   	```

2. Define the other paths and variables we would need

	
	if you are using a 64bit computer:
    ```
	user@host ~$ export CROSS_COMPILE=${RPATH_TOOLS}/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-
	```
	
	otherwise:
	```
	user@host ~$ export CROSS_COMPILE=${RPATH_TOOLS}/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian/bin/arm-linux-gnueabihf-
	```
    
    then:
    ```
	user@host ~$ export INSTALL_MOD_PATH=${RPATH_LINUX}/../kernel`
	user@host ~$ export ARCH=arm
	user@host ~$ export KERNEL=kernel7
	user@host ~$ cd ${RPATH_LINUX}
	user@host ~/linux$ mkdir ../kernel
	```
	The kernel image in Rasberry 3 is always known as kernel7.img and is not changeable, because it is dependant on the dtbs modules that are built along durin the compilation stage.

3. Install the patch for the RealTime Kernel
	```
    user@host ~/linux$ zcat patch patch-4.9.20-rt16.patch.gz | patch -p1
    ```

4. Now you can decide whether take the configuration file from a kernel altrady installed in the raspberry or create a new one

	To use an old configuration (for example the one provided in this repository) you just need to copy it in the linux source folder and the execute the following command
	```
	user@host ~/linux$ make oldconfig
	```

	To create a new the .config file from the default configuration (valid only for Raspberry Pi 2/3 Model B)
	```
	user@host ~/linux$ make bcm2709_defconfig
	user@host ~/linux$ make menuconfig
	```

5. During the configuration of the kernel we need to define a couple of parameters in order to get the Fully Preemptible Kernel, in case you used the already provided kernel, just check if they were correctly set. In the menu of the configuration file, be sure the following parameters are set:
        ```
        CONFIG_PREEMPT_RT_FULL: Kernel Features → Preemption Model (Fully Preemptible Kernel (RT)) → Fully Preemptible Kernel (RT)
        Enable HIGH_RES_TIMERS: General setup → Timers subsystem → High Resolution Timer Support
        ```
6. We can give a custom name to our kernel (visible then using the "uname -a" command after we installed the kernel) modifying in the MakeFile inside the linux source folder the parameter "EXTRAVERSION". Please pay attention to not insert empty spaces in the name.

7. Now we can compile the kernel, generate the modules and the data tree files and install the modules in the directory we defined before in the INSTALL_MOD_PATH variable
    ```
	user@host ~/linux$ make zImage modules dtbs modules_install
	```

8. At this point we are done with the compiling and we have to proceed to the next section to install the kernel in the raspberry

# Kernel Installation
## with compiled Kernel
If you are using the files provided just skip to the point #4, otherwise if you just compiled the kernel you can go on reading.

1. Create a folder which would contain the files to be copied in the boot partition
	```
	user@host ~/linux$ mkdir $INSTALL_MOD_PATH/boot
	```

2. Generate the kernel image and copy the data tree files
	```
	user@host ~/linux$ ./scripts/mkknlimg ./arch/arm/boot/zImage $INSTALL_MOD_PATH/boot/$KERNEL.img
	user@host ~/linux$ cp ./arch/arm/boot/dts/*.dtb $INSTALL_MOD_PATH/boot/
	user@host ~/linux$ cp -r ./arch/arm/boot/dts/overlays $INSTALL_MOD_PATH/boot
	```

3. Compress all the generated files
	```
	user@host cd $INSTALL_MOD_PATH
	user@host tar czf kernelAndModules-RT-4.9.tar.gz *
	```
	
4. Copy the compressed file kernelAndModules-RT-4.9 into the raspberry
    ```
	user@host scp kernelAndModules-RT-4.9 pi@raspberry:/tmp
	```

5. Finally from the raspberry we need to decompress the tarball to install the new kernel
	```
    pi@raspberry ~$ tar xzf tmp/kernel.tar.gz
    ```

9. Reboot your raspberry and you will have the new kernel installed

## starting from Raspbian
1. Download the image of the Rasbian OS
    ```
    user@host wget https://downloads.raspberrypi.org/raspbian/images/raspbian-2017-04-10/2017-04-10-raspbian-jessie.zip
    ```
    
2. Copy the image in the SD card. First of all we need to discover the mounting point if the SD card using the command:
    ```
    user@host df -h
    ```
    let's suppose the SD card is mounted at "/dev/sdX", then we have to unmount all the partitions related to it (e.g. every partition would be defined as sdX1, sdX2...)
    ```
    umount /dev/sdX1 /dev/sdX2 ...
    ```
    then we can burn the image into the SD card:
    ```
    user@host dd bs=4M if=2017-04-10-raspbian-jessie.img of=/dev/sdX
    ```
3. Copy the compressed file kernelAndModules-RT-4.9 into the raspberry
    ```
	user@host scp kernelAndModules-RT-4.9 pi@raspberry:/tmp
	```
4. Finally from the raspberry we need to decompress the tarball to install the new kernel
	```
    pi@raspberry ~$ tar xzf tmp/kernel.tar.gz
    ```