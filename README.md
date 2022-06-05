# stresstesting
POC for stresstesting large customer installs before handover

- PXE boot server running on ubuntu
- TFTP server running on the same server


POC was done with 2x KVM VMs running in a private bridge, but should be easily reproducable in real hardware

Booting stresslinux
copying ssh keys to all
initiating the stress script for all
