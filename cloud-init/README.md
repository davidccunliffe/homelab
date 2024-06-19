# CLOUD-INIT

This repository contains the cloud-init files used to autoinstall ubuntu onto the homelab environment.

To perform the actions required for the USB devices which will be plugged into each server, the following steps are required:

- Format the USB device to FAT32
- Name the USB device "CIDATA"
- Copy the cloud-init file in as user-data
- Create a blank file on the USB device named "meta-data"

e.g. the commands below will create a USB device which will autoinstall the k8s-worker2 node.

```bash
# Set this first
export K8S_ID="match name of cloud-init file in this repo"

# Format the USB device pay attention to what disk is the USB device
diskutil list
export USB="match the disk number of the USB device"

diskutil unmountDisk /dev/$USB
sudo diskutil eraseDisk FAT32 CIDATA $USB
sudo diskutil mount /dev/$USB
sudo diskutil rename /dev/$USB CIDATA

# Copy the cloud-init file
touch /Volumes/CIDATA/meta-data
cp $K8S_ID /Volumes/CIDATA/user-data

# Eject the USB device
sudo diskutil eject /dev/$USB
```

In order to boot and use the USB device for cloud-init, the following steps are required:

- Boot the server and select the USB drive that has the Ubuntu ISO on it

  - Press F12 to select the boot device, if it doesn't show up disconnect the USB device and reconnect it then attempt to boot again

- On the Ubuntu installer select "Try Ubuntu" and then open a terminal and press `e` to edit the boot options
- Move down to the that shows linux /casper/vmlinuz and modify the line to add `autoinstall` between the /casper/vmlinuz and the quiet portion of the line. It should look like this:

  - `linux /casper/vmlinuz autoinstall quiet --`

- Press F10 to boot the system
