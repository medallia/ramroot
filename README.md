# Ramroot
Ramroot is a lightweight solution for live-booting Ubuntu or Debian over HTTP,
similar to how NFS roots are supported. It is implemented as a plugin to the
standard Ubuntu initrd images.

Possible use cases include, but are not limited to:

 - Rescue images
 - Automated maintenance tasks (firmware upgrades, etc)
 - Souped-up bootloader using kexec

# Advantages over live-booting from NFS

The main advantage of using Ramroot over NFS-based live booting is that you
don't need an NFS server. NFS can be tricky to make reliable at scale, but if
scale is an issue you probably already have scalable infrastructure for serving
HTTP.

# Dependencies

Ramroot has the following dependencies:

 - Ubuntu or Debian for creating the initrd and root image (a Docker image will do)
 - pixz for compressing/uncompressing the image

# Usage

To install Ramroot, simply copy the `initramfs-tools` folder into
`/etc/initramfs-tools` in an Ubuntu or Debian system. Whenever you install a
new kernel, Ramroot will be included in the initrd image. To regenerate the
initrd of an already installed kernel in order to include Ramroot, use
`update-initramfs`.

The Ramroot image itself can be generated using whatever tools you prefer, as 
long as the output is an xzipped tarball of the root filesystem stored in a
file ending with .tar.xz. We have included instructions on how to create both
the kernel/initrd images and the ramroot filesystem using Docker.

When configuring your PXE boot, configure it as you normally would, except you
skip creating the nfs share and replace

```
boot=casper netboot=nfs nfsroot=server:/path
```

with

```
boot=ramdisk ramroot=http://your.server/ramroot.tar.xz
```

in the kernel parameters.

We recommend using a PXE solution capable of transferring the kernel and initrd
over HTTP, such as iPXE or lpxelinux. Using HTTP instead of TFTP for transferring
the initrd is significantly faster.

You may also want to configure your PXE solution to provide the mac address of
the NIC you booted from in the `BOOTIF` kernel parameter. This will allow
[the gen_network_interfaces script](initramfs-tools/scripts/ramdisk-bottom/gen_network_interfaces)
to configure the linux system to use the same NIC you booted from. This allows
networking to work smootly both during and after booting.

## Creating the initrd using Docker

See [Dockerfile.kernel-example](Dockerfile.kernel-example) for an example
Dockerfile for generating an initrd image.

To generate the initrd and extract it from the docker image, you can use or
adapt the following commands:

```bash
# Build the Docker image
docker build -f Dockerfile.kernel-example -t ramroot-kernel .
# Extract the kernel and initrd
docker run --rm -v $PWD:/host ramroot-kernel bash -c 'cp -v /boot/initrd.img-* /boot/vmlinuz-* /host'
```

You will now find a kernel image (`vmlinuz-*`) and an initrd image 
(`initrd.img-*`) in the current working directory.

You can customize the Dockerfile if you need more included in your initrd.

## Creating a Ramroot image using Docker

To create a Ramroot image using Docker, simply create a docker image and
convert it using docker-to-ramroot in the tools folder. See
[Dockerfile.ramroot-example](Dockerfile.ramroot-example) for a bare-bones image
based on the official ubuntu:16.04 image:

```bash
# Build the Docker image
docker build -f Dockerfile.ramroot-example -t ramroot .
# Convert the Docker image to a ramroot
tools/docker-to-ramroot ramroot ramroot.tar.xz
```

Notice how this image does not need to include the kernel and modules. This
allows you to upgrade the ramroot and the kernel independently, and also makes
the image smaller.

# Customization

Ramroot supports the same hooks a standard Ubuntu initrd does. To run custom
scripts in the end of the initrd execution, add them to
`/etc/initramfs-tools/scripts/ramdisk-bottom`. A few scripts for kernel and
network configuration are already provided.

# Licence
Ramroot is released under the MIT licence.
