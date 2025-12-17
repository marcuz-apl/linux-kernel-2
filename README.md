# Legacy Kernel 2001: Linux2.4 Env

After  [TreeNewBeer] writing on *2025-12-9*                                                                                                



## Intro

As of 1 December 2025, the kernel has been envoleved to 6.18, but a older and simpler kernel shall help us to get the know-how of the Kernel.

Occassionally, the "Linux Kernel Source Code Scenario Analysis (In Chinese, Linux内核源代码情景分析)" pops up with way clearer interpretation of how kernel works. The Book and its code are based on Kernel 2.4.0.

Let's get hands dirty.



## Env Establishment

Kernel 2.4, released in 2001, is too old to be compatible with the current GCC compiler.

Then, docker container is introduced to resolve the compatibility issue, using Dedian Sarge release, which embeds GCC3.3. as such we could apply the QEMU on the host to test-run it.

Please note, Kernel 2.4.0 introduces quite a lot of compiling errors, then we select Kernel 2.4.37.



## Spin up a old Debian container

```bash
git clone https://github.com/hlleng/kernel2.4-lab.git
cd kernel2.4-lab
docker run --platform linux/386 -it -v $(pwd):/code debian/eol:sarge /bin/bash
```

Once entering into docker container, update the repo sources and install the packages as needed.

```bash
apt-get update
apt-get install -y gcc make binutils libncurses5-dev wget bzip2
## may need to update the PATH
export PATH="$PATH:/usr/bin:/usr/sbin"
```



## Compile the Kernel

```bash
cd code
# Config: --> "Exit" -> "Yes" (Save)
/usr/bin/make ARCH=i386 menuconfig
# Make the dependencies
make ARCH=i386 dep
# Make the booting linux image
make ARCH=i386 bzImage
### the kernel shall be at `/code/arch/i386/boot` folder.

# Exit to the host
exit
```



## Generate file system for testing

The making of his.img could be done as below:

```bash
# Install qemu toolkit, which could take quite a while, be patientß
# sudo apt-get install qemu
brew install qemu
# Create the hda.img of 64MB size
qemu-img create -f raw hda.img 64M
```

So there is a made-ready `had.img` for later use.



## Test Run the legacy Kernel on the Host

Execute `QEMU`, loading the kernel and file system.

```bash
## Still in kernel2.4-lab folder
qemu-system-i386\
   -kernel ./arch/i386/boot/bzImage \
   -hda ./hda.img \
   -append "root=/dev/hda init=/init console=ttyS0" \
   -nographic
```

As of now, we shall be okay to enter into the kernel system.

![legacy-kernek-2.4](./assets/kernel2-4-env-established.png)

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