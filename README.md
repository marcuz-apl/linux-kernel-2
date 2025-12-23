# Legacy Kernel Linux2.4

Init'ed by  [TreeNewBeer]  | Mod'ed by marcuz-apl | 2025-12-17                                                                                                



## Intro

As of 1 December 2025, the kernel has been envoleved to 6.18, but a older and simpler kernel shall help us to get the know-how of the Kernel.

Occassionally, the "__Linux Kernel Source Code Scenario Analysis__ (PDF file attached)" pops up with way clearer interpretation of how kernel works. The Book and its code are based on Kernel 2.4.0.

Let's get hands dirty.



## Env Establishment

Kernel 2.4, released in 2001, is too old to be compatible with the current GCC compiler.

Then, docker container is introduced to resolve the compatibility issue, using Dedian Linux 3.1 (a.k.a. Sarge) released on 6 June 2005, which embeds GCC3.3. as such we could apply the QEMU on the host to test-run it.

Please note, Kernel 2.4.0 introduces quite a lot of compiling errors, then we select Kernel 2.4.37.



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
## Copy bzImage to boot folder
cd ../
cp kernel2.4-lab/arch/i386/boot/bzImage boot/
```



## Generate or Use the filesystem for testing

The generating of **hda.img** could be done as below:

Actually, what we need to do are: 

- create an image with the `ext2` filesystem (that's the predominant Linux filesystem in 2001)
- Prepare the user space with busybox
- Create an `init` script  the image to avoid kernel panic

#### A- Prepare the user space with BusyBox

Busybox is a bundle of many basic Linux utilities, for example `ls`, `grep`, and `vi` are all included. Populate sub volume with the folders needed for the install. SInce the kernel is 32-bit, then the busybox has to be 32-bit as. well.

```shell
## For a 32-bit hosts
sudo apt install build-essential gcc make ncurses-dev wget
## For 64-bit hosts
sudo apt install build-essential libc6-dev-i386 gcc-multilib g++-multilib wget

## Back to project folder
cd ../
mkdir busybox && cd busybox
```

Now you need to build a statically linked busybox binary. If you don't want to build it yourself, you can download a precompiled static busybox build.

```shell
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
# wget https://busybox.net/downloads/busybox-1.21.1.tar.bz2
tar -xvf busybox-1.37.0.tar.bz2
cd busybox-1.37.0
```

**Set the `ARCH` and `CROSS_COMPILE` environment variables**. If building natively on a 32-bit system, `CROSS_COMPILE` may not be necessary. When building on a 64-bit system, the native `gcc` can often target i386 using the `-m32` flag if the 32-bit libraries are installed.Set the architecture:

```shell
export ARCH=i386
```

(Note: `i386` is often an alias for `i486` or similar x86 32-bit architectures in the kernel/BusyBox build system). 

**Create a default configuration**:

```shell
make defconfig
```

**Run the menu-based configuration tool** to select which applets to include, choose between static or dynamic linking, and configure other options. For embedded systems, static linking is often preferred to avoid library dependencies.

```shell
make menuconfig
```

In the menu, ensure you check:

- **BusyBox Settings --> Build Options --> Build BusyBox as a static binary (no shared libs)**.

**Compile BusyBox**
Run the `make` command to compile the source code. The build system will use the configuration set in the previous steps. You can also pass the `-m32` flag to the compiler via `CFLAGS` and `LDFLAGS` if needed for 64-bit hosts:

```
# (Optional CFLAGS for 64-bit hosts if multilib is set up)
# export CFLAGS="-m32"
# export LDFLAGS="-m32"

make
```

**Install the Compiled Binary**
After compilation finishes, install the binary and associated symlinks into a local directory (e.g., `_install` in your source directory).

```
make install
```

The resulting i386 BusyBox binary will be located in the specified installation directory, ready for use in a minimal Linux environment or root filesystem. 



#### B- Create an init script

The kernel needs something to execute. It looks in `/init` for this file. On a normal linux system, this is a systemd binary, but here, it's just a shell script.

```shell
cd $HOME/initramfs
nano init
```

Put the following contents into the file **init**:

```text
#!/bin/sh

# 1. mount proc filesystem (kernel info here)
/bin/busybox mount -t proc proc /proc

# 2. Print welcome info
echo "--------------------------------"
echo "Hello! Linux Kernel 2.4 is Alive!"
echo "--------------------------------"

# 3. List root filesystem
echo "Listing / directory:"
/bin/busybox ls -l /

# 4. Start the interactive Shell
echo "Starting Shell..."
# exec indicates using sh to replace current process to save memory
exec /bin/busybox sh
```

The change the mode of **init**.

```shell
chmod +x ./init
```

#### C- Create the initramfs archive

An initramfs is a `cpio archive`. This archive needs to contain all of the files in `initramfs/`, and pass it to `cpio` to create the archive saving to the parent folder (in this specific format):

```sh
find . | cpio -o -H newc > ../initrd.img
## the file name was init.cpio, i renamed as initrd.img, the same way as in Ubuntu
```

`-o` creates/outputs a new archive, and `-H newc` specifies the type of the archive.

#### D- Create the raw image

```shell
## create a mounting folder
mkdir $HOME/linuxfs
docker run --privileged -it -v $(pwd):/code debian:bookworm bash
## you will be in a # prompt of `debian:bookworm`
```

Then, create a raw disk image, basically it's a blank file and format it as `fat`.

```shell
## Create the blank file
dd if=/dev/zero of=hda100.img bs=1M count=100
## Update and install package of `dosfatools`
apt update
apt install dosfstools
## Format as msdos filesystem
mkfs -t fat hda100.img
```

(OPTIONALLY) Verify the image inside the docker container:

```shell
## resume from above
losetup -fP --show hda100.img
#### This will return a /dev/loop0 ID, e.g. /dev/loop0

## Mount it up
mount /dev/loop0 /code
## Enter the directory to see if it's formatted correctly
cd /code
ls -alh
#### total 20K
#### drwxr-xr-x 3 root root 4.0K Dec  5 23:47 .
#### drwxr-xr-x 5 root root  160 Dec  5 23:46 ..

apt install nano
nano ./init
#### Put the following lines into the file


## Umount it
umount /linuxfs
```

Then Copy the image to the mapped folder and exit the container.

```shell
## Copy the image to the folder
mv hda100.img /linuxfs

## Exit the container
exit
```

#### E- Install the boot loader (syslinux)

We can install `syslinux` onto the `hda100.img` file created at Step A:

```shell
syslinux $HOME/linuxfs/hda100.img
```

#### F- Copy the initramfs files to the image

We need to mount that image:

```shell
mkdir m
mount $HOME/linuxfs/hda.img m
## Copy the files over
cp initrd.img m/
du -sh m
## umount and remove
umount m
rmdir m
```



Simply, use the ready-to-use `had.img` (64GB) for later practice.



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