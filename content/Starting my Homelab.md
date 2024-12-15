---
draft: "true"
---
___
## Why did I want to start my homelab

- Primarily I want to increase my knowledge and skills in Kubernetes whilst dipping my toes into some home automation
- Need a live project to practice on rather than doing courses or reading about things. W
	- Want to troubleshoot in a safe environment where things will be going wrong a lot, rather than a cluster at work
- Explore ideas and projects I'd heard about but was never able to explore 
- Get to grips with the basics which I take for granted. E.g. my networking skills. I'm doing a Coursera course and some labs but this is the practical aspect. 

## Design Decisions

#### Hardware
- Have a raspberry pi at home but wanted something more reliable as well as powerful for a home server. I had heard problems with using Kubernetes on a raspberry pi
- Didn't know where to start but was recommended a mini pc by my manager, which was in budget during black friday. (Beelink ....)

#### Software
###### ProxMox + K3s
- ProxMox - Great virtualisation platform based on debian linux, used to manage and configure vms with ease
- K3s - Lightweight, easy-to-use form of Kubernetes which is good idea to run on nodes with fewer resources

###### NixOs + K3s
- Guide for this found here - https://www.youtube.com/watch?v=2yplBzPCghA&t=473s
- A declarative way to configure an os that simplifies the management and deployment of k3s clusters

###### Talos
- Immutable OS already configured to run kubernetes
- Designed for bare metal. No VM overhead
- Uses full kubernetes

##### Why I chose Talos
- Useful for a home server that won't serve any other purpose, so no overhead
- Secure and immutable
- Closer to production-grade so more useful for learning

## The Process

Followed a video guide made by the creators of Talos: https://www.youtube.com/watch?v=VKfE5BuqlSc
#### Booting the Machine with Talos

- My mini pc already came pre-installed with Windows 11. I need to flash a USB stick with the latest Talos OS bare-metal ISO and boot the mini pc with this
- Download from the Talos Releases page the latest iso
- Use Balena Etcher to flash the usb drive with talos iso
- Plug the usb into the mini pc and start the mini pc. Find a way to get the pc into the boot menu.
	- Used UEFI to change the boot order to boot from the USB drive first.
		- I was then able to start the boot from the TALOS iso. The boot started as normal but would then stop after a couple seconds and and restart the boot process, becoming an infinite loop
		- This relates to the issue https://github.com/siderolabs/talos/issues/9369 where some extensions are needed to stop the boot loop.
		- I tried to skip this and use talos `v1.9.0-beta.1` which fixes the issue on boot but installed v.1.8.4 to the disk which carried the same problem.
- Choose Talos ISO to boot from. Once the initial boot is done you will see a dashboard with useful information. Head to your router admin page so you can assign a static IP route to the mini pc.
	- The router typically allocates dynamic IPs to its clients which will make life harder when we want to keep connecting to the pc. 
- On a separate workstation, download talosctl and run `talos gen config <cluster-name> <cluster-endpoint>` where `cluster-name` is whatever you want to call the cluster. I called mine gandalf :). `cluster-endpoint` is the IP address you got from the talos dashboard.
- Checks what disks I have on my mini pc to make sure I install talos to the right place:
```
DEV          MODEL            SERIAL   TYPE      UUID   WWID                   MODALIAS      NAME   SIZE     BUS_PATH                                                              SUBSYSTEM          READ_ONLY   SYSTEM_DISK
/dev/loop0   -                -        UNKNOWN   -      -                      -             -      139 kB   /virtual                                                              /sys/class/block   *           
/dev/loop1   -                -        UNKNOWN   -      -                      -             -      4.1 kB   /virtual                                                              /sys/class/block   *           
/dev/loop2   -                -        UNKNOWN   -      -                      -             -      192 kB   /virtual                                                              /sys/class/block   *           
/dev/loop3   -                -        UNKNOWN   -      -                      -             -      4.1 kB   /virtual                                                              /sys/class/block   *           
/dev/loop4   -                -        UNKNOWN   -      -                      -             -      75 MB    /virtual                                                              /sys/class/block   *           
/dev/sda     USB Flash Disk   -        HDD       -      -                      scsi:t-0x00   -      4.0 GB   /pci0000:00/0000:00:14.0/usb1/1-2/1-2:1.0/host0/target0:0:0/0:0:0:0   /sys/class/block               
/dev/sdb     512GB SSD        -        SSD       -      naa.53a5a27122060548   scsi:t-0x00   -      512 GB   /pci0000:00/0000:00:17.0/ata1/host1/target1:0:0/1:0:0:0               /sys/class/block            
```
- `/dev/sda` is the wrong place for me to install to and so I need to change the install location to `/dev/sdb`
- I then apply this config to the pc using `talosctl apply-config --insecure -n ************* --file controlplane.yaml`
- Problem:
	- After applying the config, v1.8.4 was installed to the disk. I removed the usb stick so that my pc could boot from disk and chose the installed talos.
	- However when booting up, the boot screen would get to the intel-i95 step and then restart. The same issue as before. This meant that the vanilla v1.8.4 iso had been installed. 
	- I realised I missed a step from the image factory page telling me to add the a particular image to the installer field in my controlplane.yaml file. 
	- The final install fields looked like:
```yaml
    install:
        disk: /dev/sdb # The disk used for installations.
        image: factory.talos.dev/installer/a67adf43d3bcbfb16d4fb6227edcf6f5ff039c493372f759d2dfcf8c21c932df:v1.8.4
```
- Retrying the same steps as before I was able to boot from disk successuly.
- Bootstrapped kuberenetes using `talosctl bootstrap --nodes 192.168.68.67 --endpoints 192.168.68.67 --talosconfig=./talosconfig`
- Then to use kubectl, we generate a kubeconfig using `talosctl kubeconfig --nodes 192.168.68.67 --endpoints 192.168.68.67 --talosconfig=./talosconfig`
- We then want to be able to deploy worker nodes on the control plane. We can adjust this using:
```yaml
	cluster:
	    allowSchedulingOnControlPlanes: true
```
- Applied the change using `talosctl apply-config -n ************* -e ************* --talosconfig ./talosconfig --file controlplane.yaml`
#### Getting started with flux

- Install flux through cli
- Run:
```bash
  flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=Tomythical/homelab \
  --branch=main \
  --path=. \
  --read-write-key=true \
  --private=false \
  --personal \
  --token-auth=false
```


```sh
flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --wait=true \
  --interval=30m \
  --retry-interval=2m \
  --health-check-timeout=3m
```