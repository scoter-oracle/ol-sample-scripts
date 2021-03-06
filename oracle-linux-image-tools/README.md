Oracle Linux image tools
========================

# Description
This repository provides tools to build Oracle Linux images for cloud deployment.

The images built by these tools are based on distribution flavours and target packages.
Image building is accomplished using Packer to build images from the Oracle Linux ISO using Oracle VM VirtualBox.

The tool currently supports:
- Distributions:
  - Oracle Linux 7 update 7 -- Slim  
    A minimal set of packages is installed
- Clouds:
  - Microsoft Azure cloud  
    Target packages: WALinuxAgent  
    Image format: VHD
  - Oracle Cloud Infrastructure (OCI)  
    Target packages: qemu-guest-agent  
    Image format: QCOW2
  - Oracle Linux Virtualization Manager (OLVM)  
    Target packages: qemu-guest-agent  
    Image format: OVA
  - Oracle VM Server (OVM)  
    Target packages: oracle-template-config + vmapi  
    Image format: OVA
  - Generic (No cloud setup)  
    Target packages: none  
    Image format: OVA

# Build instructions
1. Install packer and VirtualBox:  
  `yum --enablerepo=ol7_developer install packer VirtualBox-6.0`
1. For `OCI` or `OLVM` images, install the ` qemu-img` package:  
  `yum install qemu-img`
1. Clone this repo:  
  `git clone https://github.com/oracle/ol-sample-scripts.git`
1. The build script need root privileges during the build.
  Ensure `sudo` is properly configured for your build user
1. Set up a separate workspace directory where the image will be built.
  Ensure there is enough free space in the workspace partition, the builder will need up the two times the image size.
1. Configure your build environment in the `env.properties` file (or in a copy).  
  Minimal configuration:
    - `WORKSPACE`: path of your workspace directory
    - `ISO_URL`: location of the Oracle Linux distribution ISO
    - `ISO_SHA1_CHECKSUM`: SAH1 checksum for the ISO file
    - `CLOUD`: cloud target (azure, oci, olvm, ovm or none)
1. Run the buider:  
  `./bin/build-image.sh --env ENV_PROPERTY_FILE`

# Advanced configuration
For a given Oracle Linux distribution and target Cloud, the following properties are taken in consideration:
- Global `env.properties.default` file
- Distribution `env.properties` file
- Cloud `env.properties` file
- Cloud distribution specific `env.properties` file
- Local `env.properties` file (passed as parameter to the builder)

Files are processed in that order.  
As user you should only make changes in your local `env.properties` where you can override any definition from the previous property files.  
Relevant parameters are documented in the distributed [`env.properties`](env.properties) file.

# Cloud specific notes
## OCI
The Oracle Cloud Infrastructure `oci` cloud target generates an `QCOW2` file which can be uploaded in an _Object Storage_ bucket and imported as _Custom Image_.
This can be done from the Console, or using the [Command Line Interface (CLI)](https://docs.cloud.oracle.com/en-us/iaas/Content/API/Concepts/cliconcepts.htm). E.g.:
```shell
# Upload in the Object Storage Bucket
oci os object put -bn my_bucket \
  --file /workspace/OL7U7_x86_64-oci-b0/OL7U7_x86_64-oci-b0.qcow
# Import as Custom image
oci compute image import from-object -bn my_bucket \
  --namespace my_namespace \
  --name OL7U7_x86_64-oci-b0.qcow \
  --display-name MyImage \
  --launch-mode PARAVIRTUALIZED \
  --source-image-type QCOW2
# Import might take some time, you can monitor the progress with:
oci compute image get \
  --image-id my_image_ocid \
  --query 'data."lifecycle-state"'
# or
oci work-requests work-request get \
  --work-request-id my_work_request_ocid \
  --query 'data."percent-complete"'
# my_image_ocid and my_work_request_ocid OCIDs are returned  by the import command
```

## OLVM
The `olvm` cloud target generates an OVA file. The process to import OVA files in the Oracle Linux Virtualization Manager is described in this [blog post](https://blogs.oracle.com/scoter/import-configure-oracle-linux-7-template-for-oracle-linux-kvm).

For cloud-init support, you will need to specify `CLOUD_INIT="Yes"` in your `env.properties` file.

# Builder architecture
## Directory structure
The `build-image` script relies on the following directory structure:
- distr: directory for all Oracle Linux distribution
  - _distribution name_
    - env.properties: distribution parameters
    - _name_-ks.cfg: kickstart file for the distribution
    - provision.sh: provisioning script which is run in the VM after installation
    - image-scripts: parameter validation, kickstart customisation and image cleanup scripts run on the host
    - files (directory): all files in this directory will be copied in the VM during provisioning
- cloud: directory for all target clouds
  - _cloud name_
    - env.properties: cloud parameters
    - provision.sh: provisioning script which is run in the VM after installation
    - image-scripts: parameter validation and image cleanup and packaging scripts run on the host
    - files (directory): all files in this directory will be copied in the VM during provisioning
    - _distribution name_: in case a a distribution specific action needs to be done for this cloud target, it can be defined in this directory.

Most of the files are optional, only define what is needed.

## Build process
The builder will process the directories in the following order:
1. Read properties files as described in [advanced configuration](#advanced-configuration).  
  The properties are available in all scripts, on the host and in the VM during provisioning.  
  Properties can be validated at distribution / cloud level:
  - distr::validate
  - cloud::validate
  - cloud_distr::validate
1. Select a kickstart file from _distr_ and customise it. The following hooks are called if defined:
    - distr::kickstart
    - cloud_distr::kickstart
1. Stage files from the _files_ directories. These files are copied during provisioning in `/tmp/packer_files` in the VM.
1. Run packer to provision the VM image.
  During provisioning the `provision.sh` scripts run in the following order:
    - distr::provision
    - cloud::provision
    - cloud_distr::provision
    - cloud_distr::cleanup
    - cloud::cleanup
    - distr::cleanup
1. Image cleanup: the generated image is mounted on the host and the `image-scripts` scripts are run:
    - cloud_distr::cleanup
    - cloud::cleanup
    - distr::cleanup
1. Image packaging: the generated image is packaged in its final format.
  Only the first script found is executed:
    - cloud_distr::image_package
    - cloud::image_package

# Feedback
Please provide feedback of any kind via GitHub issues on this repository.

# Contributing
See [CONTRIBUTING](CONTRIBUTING.md) for details.
