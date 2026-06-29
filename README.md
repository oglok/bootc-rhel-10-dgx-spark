# RHEL 10 bootc Image for NVIDIA DGX Spark (GB10)

A bootc container image that packages a custom RHEL 10 kernel with NVIDIA GPU driver support for the DGX Spark platform (GB10 Grace Blackwell Superchip), plus CUDA.

## What's Included

- **Base image**: `registry.redhat.io/rhel10-eus/rhel-10.2-bootc:10.2`
- **Custom kernel**: `kernel-64k` built from the [nvidia-gb10](https://gitlab.com/redhat/edge/kernel/nvidia-gb10) repository (64k page size for ARM64)
- **NVIDIA GPU driver**: `kmod-nvidia-open` v595.58.03 (open-source kernel modules)
- **CUDA**: NVIDIA CUDA toolkit

## Prerequisites

- An **aarch64** (ARM64) build host
- **Podman** installed
- A **Red Hat subscription** (the build host must be registered so entitlements are shared into the container build)
- Access to `registry.redhat.io` (run `podman login registry.redhat.io`)

## Build Instructions

### 1. Clone this repository

```bash
git clone git@github.com:oglok/bootc-rhel-10-dgx-spark.git
cd bootc-rhel-10-dgx-spark
```

### 2. Clone the GB10 kernel source

The kernel source is ~1.9 GB and must be cloned on a **case-sensitive filesystem** (Linux). macOS's default filesystem is case-insensitive and will corrupt the kernel source.

On a Linux/aarch64 system:

```bash
git clone --depth 1 https://gitlab.com/redhat/edge/kernel/nvidia-gb10.git
tar cf nvidia-gb10.tar nvidia-gb10/
```

If building on macOS with podman, clone inside the podman VM:

```bash
podman machine ssh -- "git clone --depth 1 https://gitlab.com/redhat/edge/kernel/nvidia-gb10.git /tmp/nvidia-gb10"
podman machine ssh -- "cd /tmp && tar cf nvidia-gb10.tar nvidia-gb10/"
podman machine ssh -- "cat /tmp/nvidia-gb10.tar" > nvidia-gb10.tar
```

### 3. (Only if planning to use the Wayland container) Grab the egl-wayland source

```bash
wget https://github.com/NVIDIA/egl-wayland/archive/1.1.21/egl-wayland-1.1.21.tar.gz
```

### 4. Ensure the podman machine has enough memory (macOS only)

The kernel build requires significant memory. Allocate at least 16 GB:

```bash
podman machine stop
podman machine set --memory 16384
podman machine start
```

### 5. Build the image

```bash
podman build -t dgx-spark-bootc .
```

The build uses a multi-stage approach:
- **Stage 1** (builder): Compiles the custom kernel RPMs and NVIDIA driver RPM. This takes 30 minutes to several hours depending on hardware.
- **Stage 2** (final image): Installs the pre-built RPMs, EPEL, and CUDA into a clean bootc image.

### 5. a. Build the Wayland image

If you want your bootc image to also have Wayland (GUI) with GPU acceleration, you will first need to fetch the egl-wwayland library sources : 

```bash
wget https://github.com/NVIDIA/egl-wayland/archive/refs/tags/1.1.20.tar.gz
```

then build the wayland image :

```bash
podman build -t dgx-spark-bootc:wayland -f Containerfile.wayland .
```

### 5. b. Build the Installer image

If you want to create an installer iso embedding the Spark kernel, you will need to create an installer container to point to in the next steps :

```bash
podman build -t dgx-spark-bootc:installer -f Containerfile.installer .
```

### 6. Push the image (optional)

```bash
podman tag dgx-spark-bootc quay.io/<your-namespace>/rhel-10-dgx-spark:latest
podman push quay.io/<your-namespace>/rhel-10-dgx-spark:latest
```

## Deploying to a DGX Spark

### On a running system 
On the target GB10 system, switch to the bootc image:

```bash
sudo bootc switch quay.io/<your-namespace>/rhel-10-dgx-spark:latest
sudo reboot
```

After reboot, verify:

```bash
uname -r                # Should show 6.12.x-xxx.gb10.0.el10.aarch64+64k
lsmod | grep nvidia     # Should list nvidia, nvidia_uvm, nvidia_drm, nvidia_modeset, nvidia_peermem
nvidia-smi              # Should show GPU info
```

### Creating an iso to deploy from scratch 

To deploy from scratch, we will need to create an installer iso with our custom kernel. the bootc-installer image type allows exactly that : using our bootc image as the base for the installer. This requires our image to be customized a bit to add anaconda packages to handle the install (which we did while creating our :installer tag in step 5.b. ). This also requires pushing the images to an online repository 

```bash
sudo mkdir -p output; sudo podman run --rm -it --privileged --pull=newer --security-opt label=type:unconfined_t -v /var/lib/containers/storage:/var/lib/containers/storage -v $(pwd)/output:/output registry.redhat.io/rhel10/bootc-image-builder:10.2 --type bootc-installer --installer-payload-ref quay.io/<your namespace>/spark-bootc:wayland quay.io/<your namespace>/spark-bootc:installer
```

This example is using the wayland image to be installed, this can be pointed to the standard image as well

You will also need to create a kickstart and embed it in the install iso to set the right kernel cmdline and a user. an example kickstart is in this repository and may be customized to set a different user or password, add ssh rules etc ... You'll also need to have the lorax package installed locally to embed the kickstart into the iso

```bash
sudo dnf -y install lorax
sudo mkksiso kickstart.ks output/bootiso/install.iso output/bootiso/bootc-install-ks.iso
```

Finally, burn the iso to an usb key and boot your Spark device using it, it will automatically install and reboot 

## Updating the Kernel

The nvidia-gb10 kernel repository is regularly rebased. To rebuild with the latest kernel:

```bash
cd nvidia-gb10
git fetch origin
git reset --hard origin/main
cd ..
tar cf nvidia-gb10.tar nvidia-gb10/
podman build --no-cache -t dgx-spark-bootc .
```

## Notes

- This is a **Developer Preview** — NVIDIA DGX Spark support for RHEL is not intended for production use. See [Red Hat Developer Preview](https://access.redhat.com/support/offerings/devpreview).
- The `kernel-64k` variant uses 64k page sizes for better ARM64 performance at the cost of increased memory consumption.
- The NVIDIA driver version (595.58.03) is pinned in `kmod-nvidia-open.spec`. Update `driver_version` in the spec file to build a different version.
