# tfNestedEsxiVcenterMultipleVdsNsxAvi

## Goal

This Infrastructure as code will deploy a nested ESXi/vCenter/NSX/Avi (on the top of vCenter environment which does not support 802.1q or vlan tagged).

## underlay infrastructure

### tested against
- vCenter underlay version:
```
6.7.0
```

### Orchestrator OS and softwares
= OS
```shell
NAME="Ubuntu"
VERSION="20.04.3 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.3 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal

```

- Terraform:
```
Terraform v1.0.6
on linux_amd64
```

- govc:
```
govc v0.24.0
```

### Files required to build the underlay VM(s)

- dns_ntp:
  - "bionic-server-cloudimg-amd64.ova" defined in "vcenter_underlay.cl.file"
- external-gw:
  - "focal-server-cloudimg-amd64.ova defined" in "vcenter_underlay.cl.file_external_gw"  
- ESXi(s):
  - "VMware-VMvisor-Installer-7.0U3d-19482537.x86_64.iso" defined in vcenter_underlay.cl.file

### VM(s)

On the top of an underlay/outer vCenter environment, this repo will create the following:

![img.png](imgs/img01.png)

### VM(s) connectivity

![img.png](imgs/underlay_architecture.png)

### Network details

The following variables need to be configured:

- For the management network:
  - vcenter.dvs.portgroup.management.netmask like "255.255.252.0"
  - vcenter.dvs.portgroup.management.prefix like "22"
  - vcenter.dvs.portgroup.management.gateway like "10.41.132.1"
- For the VMotion network:
  - vcenter.dvs.portgroup.VMotion.netmask like "255.255.255.0"
  - vcenter.dvs.portgroup.VMotion.prefix like "24"
- For the vSAN network:
  - vcenter.dvs.portgroup.VSAN.netmask like "255.255.255.0"
  - vcenter.dvs.portgroup.VSAN.prefix like "24"

### VM(s) IPs

- For the management network, static IPs are configured as followed:
    - the ESXi VMs IPs are defined in vcenter.dvs.portgroup.management.esxi_ips[]
    - the ESXi VMs temporary IPs are defined in vcenter.dvs.portgroup.management.esxi_ips_temp - these IPs are released once the infra is ready
    - the dns_vm VM IP is defined in vcenter.dvs.portgroup.management.dns_ntp_ip
    - the external-gw VM IP is defined in vcenter.dvs.portgroup.management.external_gw_ip

- For the Vmotion network, static IPs are configured as followed:
    - the ESXi VMs IP are defined in vcenter.dvs.portgroup.VMotion.esxi_ips[]

- For the vSAN network, static IPs are configured as followed:
    - the ESXi VMs IP are defined in vcenter.dvs.portgroup.VSAN.esxi_ips[]

- For the NSX external network, static IPs are configured as followed:
    - the external-gw VM IP is defined in vcenter.dvs.portgroup.nsx_external.external_gw_ip

- For the NSX overlay network, static IPs are configured as followed:
    - the external-gw VM IP is defined in nsx.config.ip_pools[0].gateway

- For the NSX overlay edge network, static IPs are configured as followed:
    - the external-gw VM IP is defined in nsx.config.ip_pools[1].gateway

### VM(s) usage

- the dns_ntp VM is used as a DNS server (for A records and PTR records) for all the VM in the nested infrastructure
- the external-gw VM is used as:
  - an external gateway for traffic coming from the NSX Edge nodes to the Internet (this will source NAT the traffic)
  - a router between the transport nodes and the edge nodes for the overlay (geneve traffic) 
- ESXi VM(s) will be the foundation of the nested infrastructure

## Nested infrastructure

### Files required to build the nested VM(s)

- vCenter:
  - "VMware-VCSA-all-7.0.3-19480866.iso" defined in "vcenter.iso_source_location"
- NSX Manager:
  - "nsx-unified-appliance-3.2.1.0.0.19801963.ova" defined in "nsx.content_library.ova_location"
- Avi Controller:
  - "controller-21.1.4-9210.ova" defined in "avi.content_library.ova_location"
- Avi App:
  - "focal-server-cloudimg-amd64.ova" defined in "avi.app.ova_location"  


### vCenter deployment

It will deploy a single vCenter on the top of the management port group (connected to the management network of the underlay infrastructure).
IP is defined in vcenter.dvs.portgroup.management.vcenter_ip.

Here is below the nested infrastructure created:

![img.png](imgs/nested_vcenter_overview.jpg)  

### vCenter configuration:

- networking (vds and port group)
- adding ESXi hosts
- vSAN
- ...
- NSX VDS(s) and port group(s):

![img.png](imgs/nested_vcenter_vds_pg.jpg)

### NSX Manager deployment

It will deploy a single NSX Manager on the top of the management PG (connected to the management network of the underlay infrastructure).
IP is defined in vcenter.dvs.portgroup.management.nsx_ip.

### NSX configuration:
- NSX license
- ip pools

![img.png](imgs/nested_nsx_ip_pools.jpg)  

- uplink profiles

![img.png](imgs/nested_nsx_uplink_profiles.jpg)

- transport node profiles

![img.png](imgs/nested_nsx_transport_node_profiles.jpg)

- transport zones

![img.png](imgs/nested_nsx_transport_zones.jpg)

- compute managers
- transport nodes

![img.png](imgs/nested_nsx_transport_nodes.jpg)

- edge transport nodes

![img.png](imgs/nested_nsx_edge_nodes.jpg)

- edge clusters

![img.png](imgs/nested_nsx_edge_clusters.jpg)  
- tier0(s):
  - interfaces
  - routing
  - HA VIP configuration

![img.png](imgs/nested_nsx_tier0s.jpg)
- tier1(s):
  - IP
  - route advertisement
  
![img.png](imgs/nested_nsx_tier1s.jpg)
- overlay segment(s):
  
![img.png](imgs/nested_nsx_segments.jpg)

### Avi controller deployment

It will deploy a single Avi controller on the top of the management overlay segment defined in nsx.config.segments_overlay[0].display_name.

### Avi App deployment and configuration

It will deploy server(s) on segment nsx.config.segments_overlay[1] based on the amount of IP(s) defined in nsx.config.segments_overlay[1].avi_app_server_ips
Each server will listen:
- on port defined in var.avi.app.hello_world_app_tcp_port with a basic hello world message
- on port defined in var.avi.app.avi_app_tcp_port with a nice app
- on port defined in var.avi.app.hackazon_tcp_port with hackazon app

All the server VMs (-1) will be tagged in NSX with the tag defined in var.avi.app.nsxt_vm_tags:

![img.png](imgs/nested_nsx_vm_tags.jpg)

A nsx group is configured based on this VM tag:

![img.png](imgs/nested_nsx_group.jpg)

### Avi controller configuration

- Bootstrap controller with password defined in environment variable (see below) 
- System parameters
- DNS profile
- IPAM profile
- NSX Cloud

![img.png](imgs/nested_avi_clouds.jpg)
- VSVIP Config
- Pool config

![img.png](imgs/nested_avi_pools.jpg)

- VS config

![img.png](imgs/nested_avi_vs.jpg)


## variables

### non sensitive variables

All the non sensitive variables are stored in variables.json

### sensitive variables

All the sensitive variables are stored in environment variables as below:

```bash
export TF_VAR_esxi_root_password=******              # Nested ESXi root password
export TF_VAR_vsphere_username=******                # Underlay vCenter username
export TF_VAR_vsphere_password=******                # Underlay vCenter password
export TF_VAR_bind_password=******                   # Bind password
export TF_VAR_vcenter_password=******                # Overlay vCenter admin password
export TF_VAR_nsx_password=******                    # NSX admin password
export TF_VAR_nsx_license=******                     # NSX license
export TF_VAR_avi_password=******                    # AVI admin password
export TF_VAR_avi_old_password=******                # AVI old passwors
```

## start the script (create the infra)

```shell
git clone https://github.com/tacobayle/tfNestedEsxiVcenterMultipleVdsNsxAvi ; cd tfNestedEsxiVcenterMultipleVdsNsxAvi ; /bin/bash apply.sh
Cloning into 'tfNestedEsxiVcenterMultipleVdsNsxAvi'...
remote: Enumerating objects: 293, done.
remote: Counting objects: 100% (293/293), done.
remote: Compressing objects: 100% (195/195), done.
remote: Total 293 (delta 130), reused 250 (delta 87), pack-reused 0
Receiving objects: 100% (293/293), 2.05 MiB | 2.30 MiB/s, done.
Resolving deltas: 100% (130/130), done.
-----------------------------------------------------
Build of a folder on the underlay infrastructure - This should take less than a minute
Starting timestamp: Thu Mar 31 14:58:15 UTC 2022
Ending timestamp: Thu Mar 31 14:58:19 UTC 2022
-----------------------------------------------------
Build of a DNS/NTP server on the underlay infrastructure - This should take less than 5 minutes
Starting timestamp: Thu Mar 31 14:58:19 UTC 2022
Ending timestamp: Thu Mar 31 15:01:28 UTC 2022
-----------------------------------------------------
Build of an external GW server on the underlay infrastructure - This should take less than 5 minutes
Starting timestamp: Thu Mar 31 15:01:28 UTC 2022
Ending timestamp: Thu Mar 31 15:07:04 UTC 2022
-----------------------------------------------------
Build of the nested ESXi/vCenter infrastructure - This should take less than 45 minutes
Starting timestamp: Thu Mar 31 15:07:04 UTC 2022
Ending timestamp: Thu Mar 31 15:49:46 UTC 2022
waiting for 15 minutes to finish the vCenter config...
-----------------------------------------------------
Build of NSX Nested Networks - This should take less than a minute
Starting timestamp: Thu Mar 31 16:04:46 UTC 2022
Ending timestamp: Thu Mar 31 16:04:55 UTC 2022
-----------------------------------------------------
Build of the nested NSXT Manager - This should take less than 20 minutes
Starting timestamp: Thu Mar 31 16:04:55 UTC 2022
Ending timestamp: Thu Mar 31 16:22:30 UTC 2022
waiting for 5 minutes to finish the NSXT bootstrap...
-----------------------------------------------------
Build of the config of NSX-T - This should take less than 60 minutes
Starting timestamp: Thu Mar 31 16:27:30 UTC 2022
Ending timestamp: Thu Mar 31 17:20:04 UTC 2022
-----------------------------------------------------
Build of Nested Avi Controllers - This should take around 15 minutes
Starting timestamp: Thu Mar 31 17:20:04 UTC 2022
Ending timestamp: Thu Mar 31 17:34:07 UTC 2022
-----------------------------------------------------
Build of Nested Avi App - This should take less than 10 minutes
Starting timestamp: Thu Mar 31 17:34:07 UTC 2022
Ending timestamp: Thu Mar 31 17:42:27 UTC 2022
-----------------------------------------------------
Build of the config of Avi - This should take less than 20 minutes
Starting timestamp: Thu Mar 31 17:42:27 UTC 2022
Ending timestamp: Thu Mar 31 17:59:59 UTC 2022
```

## destroy the infra

```shell
/bin/bash destroy.sh
```
