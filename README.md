# packer-macOS-11

This a packer template for macOS 11 built on VMware fusion 12. It's created using the newer hcl2 syntax which currently has some known issues.  [HCL2: implementation list](https://github.com/hashicorp/packer/issues/9176) 

## Discussion thread for usage questions
Please see this tread for general usage questions.
https://discuss.hashicorp.com/t/building-macos-11-x-vms-with-packer-and-fusion/

## What's working
* [scripts/buildprerequs.sh](buildprerequs.sh) creates a macOS installer .iso
* Using voiceover and boot commands to open terminal.app !!
* Downloading .pkg and script payloads to the recover environment 
* Running the payload scripts that handle the install process
* packer user creation and autologin
* Clearing setup screens
* Enable remotelogin system settings
* Install Xcode & cli tools

## What's missing
I'll give the community a few months to sort out any reasonable options for these. Not interested in creating a full MDM workflow here.
* Get rid of feedback assistant popup
* Approve VMware tools system extension and helper tool
* Profile to adjust a bunch of settings

## Building macOS 11 with this packer template
* Minimum packer version is 1.6.5
* VMware Fusion 12.0 or greater

## Update submodules
After cloning this repo you must pull down the submodules by running the following command from the root of the repo.

    git submodule update --init

## Prerequisite installer bits
I have imported two projects as submodules to create the needed macOS installer .iso. Running the [scripts/buildprerequs.sh](buildprerequs.sh) will call those to generate it. If you have a macOS 11 install .iso from some other method that will work as well. 

Thanks to all contributors of the following project that are imported as submodules:

* [create_macos_vm_install_dmg](https://github.com/rtrouton/create_macos_vm_install_dmg)
* [macadmin-scripts](https://github.com/munki/macadmin-scripts)

With the customize build I'm installing Xcode 12.3. Grab both the latest Xcode .xip and matching command line tools installer dmg from [developer.apple.com](https://developer.apple.com). Toss them into the `install_bits` directory. 

Here is what your `install_bits` directory should look like to successfully build the full image:
```
install_bits/
├── Command_Line_Tools_for_Xcode_12.3.dmg
├── Xcode_12.3.xip
├── dmg
├── macOS_1100_installer.iso
└── macOS_1100_installer.shasum
```

## Named builds
This template has three named builds `base`, `customize`, and `full`. The idea here is splitting the lengthy process of macOS installation (baking the image) from the customization (frying the image). The `base` build does the os install with the vmware-iso builder and `customize` takes the output VM from that and customizes it. Re-running the customization quickly gets allows for quicker testing of that phase. The `full` build does all the steps at once and if you're not testing the customizations likely what you want to use. 

### Building the full image 
Builds the VM with all the options including Xcode

    packer build -force -only=full.vmware-iso.macOS_11 macOS_11.pkr.hcl

### Building the base image
Builds just the OS including VMware tools

    packer build -force -only=base.vmware-iso.macOS_11_base macOS_11.pkr.hcl

### Building the customize image
Useful for testing customizations without waiting for the whole OS to install.

    packer build -force -only=customize.vmware-vmx.macOS_11_customize macOS_11.pkr.hcl

### Input variables
This template uses input variables for a few customizable values. Run packer inspect to see the defaults and what can be changed. Editing the variable default in the macOS_11.pkr.hcl file works as well.

    packer inspect macOS_11.pkr.hcl

## Adjust resources
It's likely you will need to adjust the cpu and RAM requirements to match your available resources. The variables can be edited in the template directly or passed on the cli. 

    packer build -force -only=full.vmware-iso.macOS_11 -var cpu_count=2 -var ram_gb=6 macOS_11.pkr.hcl

### Username & Password
The build process created a packer user with UID 502. It's recommended to login with that account and create a new user with the appropriate password when you start using the VM. 

    Username: packer
    Password: packer

Additionally the username is embeded in packages and scritps using durring the install process. Update scripts/setupsshlogin.sh, scripts/makepkgs.sh & packages/sesetupsshlogin.pkgproj

If you want to override the username and password they can be specified on the cli

    packer build -force -only=full.vmware-iso.macOS_11 -var user_password=vagrant -var user_username=vagrant macOS_11.pkr.hcl

### Customize computer serial and model
Variables have been added to customize board id, hardware model & serial number. This can be handy for testing DEP workflows.

    packer build -force -only=full.vmware-iso.macOS_11 -var board_id=Mac-27AD2F918AE68F61 -var serial_number=M00000000001 -var hw_model="MacPro7,1" macOS_11.pkr.hcl

### Apple GPU support on Big Sur hosts
If the host system is running macOS 11.x enabling the virtualized GPU provides a dramatic speedup of the GUI. Running the pvapplegpu.sh script with add the appropriate vmx entries to the specified vmx file. The VM needs to be powered off for this change.

    scripts/pvapplegpu.sh output/macOS_11/macOS_11.vmx

### Big Sur host workaround
As of Fusion 12.1 & packer, 1.6.5 ssh connections will fail unless a dhcp lease file is populated by an external process. This has to do with adoption of hypervisor.framwork on macOS 11. The below workaround is known to resolve this but will fail if multiple VMs are running. 

    (sleep 120 && scripts/fakelease.sh)& packer build -force -only full.vmware-iso.macOS_11 macOS_11.pkr.hcl

A fix for packer 1.6.6 has been submitted and known to resolve this issue as well.
