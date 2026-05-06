# GPU Passthrough Scripts for QEMU/libvirtd with KVMFR and Looking Glass
Scripts and file structure to setup dynamic dGPU passthrough for qemu on Ubuntu. Everything in the hooks folder belongs in /etc/libvirt/hooks
The shutdown scripts are for shutdown protection in Ubuntu 24.04 LTS and org.gnome.Shell@wayland.service contains a block to be added to the service on Ubuntu to prevent gnome-shell from creating a process on the NVIDIA dGPU on startup.
