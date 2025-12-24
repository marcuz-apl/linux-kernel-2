# Legacy Kernel Linux2.4

Init'ed by  [TreeNewBeer]  | Mod'ed by marcuz-apl | 2025-12-17                                                                                                



## Intro

As of 1 December 2025, the kernel has been envoleved to 6.18, but a older and simpler kernel shall help us to get the know-how of the Kernel.

Occassionally, the "__Linux Kernel Source Code Scenario Analysis__ (PDF file attached)" pops up with way clearer interpretation of how kernel works. The Book and its code are based on Kernel 2.4.0.

Let's get hands dirty.



## Env Establishment

Kernel 2.4, released in 2001, is too old to be compatible with the current GCC compiler.

Then, docker container is introduced to resolve the compatibility issue, using **Dedian Linux 3.1 (a.k.a. Sarge) released on 6 June 2005**, which embeds GCC3.3. as such we could apply the QEMU on the host to test-run it.

Please note, Kernel 2.4.0 introduces quite a lot of compiling errors, then we select **Kernel 2.4.37** (released on **18 December 2010**).



## Spin up a old Debian container

```shell
## Make folder for future boot
mkdir boot
## WOrking on kernel source code
git clone https://github.com/hlleng/kernel2.4-lab.git
cd kernel2.4-lab

docker run --platform linux/386 -it -v $(pwd):/code debian/eol:sarge /bin/bash
```

Once entering into docker container, update the repo sources and install the packages as needed.

```bash
## Update and Install packages
apt-get update
apt-get install -y gcc make binutils libncurses5-dev wget bzip2
```



## Compile the Kernel

```bash
cd code
# Config: --> "Exit" -> "Yes" (Save)
make ARCH=i386 menuconfig  ## Press "Tab" key to "Exit" the menu
# Make the dependencies
make ARCH=i386 dep
# Make the booting linux image - taking quite a while
make ARCH=i386 bzImage
### the kernel shall be at `/code/arch/i386/boot` folder.

# Exit to the host
exit
## Copy bzImage to boot folder
cp arch/i386/boot/bzImage ../boot/
cd ..
```



## Use the filesystem

The generating of **hda.img** could be done in a complex method. then we are simply gonna use the ready-to-use `had.img` (64GB) for later practice.:

```shell
cp kernel2.4-lab/hda.img boot/
```



## Test Run the legacy Kernel on the Host

Execute `QEMU`, loading the kernel and file system.

```bash
## Go to boot folder
cd boot/
qemu-system-i386 -kernel ./bzImage -hda ./hda.img -nographic
     -append "root=/dev/hda init=/init console=ttyS0"
```

As of now, we shall be okay to enter into the kernel system.

![legacy-kernek-2.4](./assets/kernel2-4-env-established.png)

Press "**Ctrl+A**" and "**x**" to exit the Qemu in Terminal.



## Copy files

If needing to copy files from the Host to the `QEMU` system, please try to. steps below on the Host.

```bash
> mkdir -p tmp/mnt
> sudo mount -o loop hda.img tmp/mnt
# execute the copying file operations
> sudo umount tmp/mnt
```

As of now, we have created an env of Kernel 2.4.37, where we could play around the legacy kernel.



## The End