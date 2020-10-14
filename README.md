## Overview

Scripts and ansible playbooks to deploy OpenStack in a lab environment

```bash
git clone git@github.com:szeidat/OpenStack.git stack
```

## Workflow

To create a stack follow these steps

- Create machine source images (vm templates and lxd containers)
- Create a stack-master machine manually
- Clone this repo to the master and copy demo images folder
- Modify variables if needed (see variables section below for details)
- Run ansible/setup script to setup the master machine
- Run ansible/provision script to provision machines
- Run ansible/deploy script to deploy openstack services
- Run ansible/demo script to demo various openstack services
- To access haproxy stats web page  
  https://stack-management-vip:3100
- To access openstack dashboard  
  https://stack-management-vip:3200
- Monitor logs collected on master by running  
  `tail -F /var/log/stack | grcat conf.stack`


## Directories

Below is the directories and file structure under ansible

- inventory/hosts: ansible's hosts file created by the provisioning script
- inventory/host_vars: ansible's host vars folder containing host variables created by the provisioning script
- playbooks/setup.yml: playbook to setup the stack master machine
- playbooks/provision.yml: playbook to provision machines
- playbooks/deploy.yml: playbook to deploy openstack services
- playbooks/roles: folder for ansible roles
- variables/common.json: ansible openstack common variables
- variables/environment.json: ansible environment specific variables
- variables/machines.json: machine definetions


## Networks

- Management: internal for management and default route
- Access: public for consumer access to load balancer
- External: vlan trunk for provider vlans and flat network using untagged vlan
- Data: vlan trunk for tenant vlans and vxlan overlay using untagged vlan
- Storage: storage access to ceph osds and iscsi/lvm providers if deployed
- Replication: storage replication between ceph osds


## Environments

An environment represents a selection of deployment variables and machine definetions:

- It allows switching between different deployment scenarios
- Each environment is defined by a folder under the variables main folder
- File environment.json holds ansible variables specific to the environment
- File machines.json contains machine definetions for the environment
- File ansible.cfg is ansible's configuration specific to the environment
- Environments can inherit from each other
- Variables are combined when environments are inherited with child environment overriding parent variables if defined
- Machines and ansible config are not combined. If defined in child environment they're used otherwise we fall back to parent

Each environment.json file must include at minimum the following variables:

- environment_backend: defines the host technology for creating machines (lxd, esx)
- environment_manager: defines the tool used for provisioning machines (ansible, foreman)
- environment_distro: distribution used for provisioning machines (centos, opensuse, ubuntu)
- environment_description: short description of the environment


## Variables

Variables are defined in three places:

- Common variables to all environments are defined in variables/common.json file
- Environment specific variables are defined in environment.json file in each environment folder
- Host speicific variables (host vars) are defined in the machines.json file in each environment folder (as "parameters")
- Switching to an environment causes the environment variables to be copied to variables/environment.json file and machines file
  to be copied to variables/machines.json file

Below are the different host variables used by the different ansible roles:

- glance: glance store type (ceph, vmdk)  
  `glance_store_type: ceph, vmdk`
- glance: glance store vmware datastore (for vmdk store)  
  `glance_vmware_datastore: datastore11`
- horizon: alternate theme (for RedHat and SUSE themes on centos and opensuse distros)  
  `horizon_alt_theme: true`
- neutron gateway: networking type (linuxbridge, openvswitch)  
  `neutron_networking_type: openvswitch`
- nova compute: networking type (linuxbridge, openvswitch, dvs, ovsvapp)  
  `neutron_networking_type: openvswitch`
- nova compute: compute type (libvirt, vmware, lxd)  
  `nova_compute_type: libvirt`
- nova compute: libvirt type (qemu, kvm)  
  `nova_compute_libvirt_type: qemu`
- nova compute: esx hostname (for ovsvapp networking type)  
  `nova_compute_esx_hostname: hosts04.lab.local`
- nova compute: vmware cluster (for vmware compute type)  
  `nova_compute_vmware_cluster: production`
- nova compute: vmware datastore (for vmware compute type)  
  `nova_compute_vmware_datastore: datastore41`
- cinder volume: volume backends (ceph, vmdk, lvm)  
  `cinder_volume_backends: ceph, vmdk, lvm`
- cinder volume: cluster (for vmdk volume type)  
  `cinder_volume_cluster: production`
- cinder volume: lvm device (for lvm volume type)  
  `cinder_volume_lvm_device: /dev/sdb`
- cinder volume: backup type (ceph, swift)  
  `cinder_backup_type: swift`
- neutron gateway: networking type (linuxbridge, openvswitch)  
  `neutron_networking_type: openvswitch`
- manila share: share backends (ceph, vmdk) list  
  `manila_share_backends: ceph, vmdk`
- manila share: networking type (linuxbridge, openvswitch)  
  `neutron_networking_type: openvswitch`


## Provisioning Managers

### Foreman

- Setup vcenter networking 
- Set promisc accept on data and external switches
- Setup vcenter vm templates for opensuse, ubuntu and centos
- Templates must have two disks as foreman can't add disks to a vm
- Master vm will run forman and provide dns and syslog
- Routing is handled by lab vsr router
- Connect master vm network interface to management network
- Foreman can provide console to vms if firewall ports on esxi are opened
- To access foreman web interface  
  https://stack-master:3000
- Following must be done manually in vcenter (not able to automate using ansible):
  change portgroups data and security policies to accept all
  change portgroups data and security vlan to trunking (range 0-4094)


## Backends

### LXD

- Install lxd on host machine
- Setup bridge networks for stack containers and master container nat to access the internet
- Setup lxd template containers for opensuse, ubuntu and centos
- Deploy stack-master container from ubuntu template
- Master container will provide dns and syslog services

### ESX

- The vmware-setup role creates all dv switches and dv portgroups. it can also create/configure clusters, and add
  hosts to them if variables are set correctly. currently limited to two specific clusters (management, compute)
  and also limited to one host per cluster.
- Ansible vmware_guest module used for host provisioning
- Create opensuse, ubuntu and centos vm templates
  templates must have one disk only as vmware_guest module will fail if the vm to be
  created has less disks tha the template
- Deploy stack-master vm from ubuntu template
- Master vm will run ansible playbooks and provide dns, ntp, syslog, and apt-cache services
- Routing is handled by lab vsr router
- Connect master vm network interface to management network (record mac address and update variables file)
- Update vmware-variables file
- Run vmware-setup and follow the sequence to setup the master vm and compute vcenter (stack-vcs-01)
- Following must be done manually in vcenter:
  change portgroups data, external, and security policies to accept all
  change portgroups data, external, and security vlan to trunking (range 0-4094)


