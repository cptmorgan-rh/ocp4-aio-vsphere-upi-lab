All-in-One OpenShift vSphere UPI Lab with Static IPs
===========================================

## Description
------------

The goal of this repository is to provide a simple, reproducible way to deploy an OpenShift Container Platform lab using the vSphere UPI method with Static IP Addresses.

## Quickstart
This is a concise summary of everything you need to do to use the repo. Rest of the document goes into details of every step.
1. Edit `group_vars/all.yml`, the following must be changed while the rest can remain the same
   * pull secret
   * ip and host/domain names
   * enable/disable fips mode
   * vcenter details
     * datastore name
     * datacenter name
     * username and passwords of admin/service accounts
   * validate current Govc version is set
   * enable ntp with their details, as required
2. If you wish to run a specific Channel and Version modify the following in `group_vars/all.yml`:
   * download.channel
   * download.version
3. For the CoreDNS VM to be able to pull the image from Quay.io you must specify an `coredns_vm.upstream_dns`. It cannot have itself as a primary DNS Server

## Infrastructure Prerequisites

1. vSphere ESXi and vCenter 6.7U3 or 7.0 installed.
   * 6.5 is not supported by this repository due to HW Version 15.
2. A datacenter created with a vSphere host added to it, a datastore exists and has adequate capacity
3. Ansible (preferably latest) on the machine where this repo is cloned.
   * Before you install Ansible, install the `epel-release`, run `yum -y install epel-release`
4. Your DNS Provider (PiHole, AdGuard, etc) should be configured to lookup your `base_domain` from your `coredns_vm.ipaddr`
   * Optionally, you configure the `coredns_vm.upstream_dns` to be your primary DNS Server and then configure your workstation or bastion host to use the CoreDNS Server as your primary DNS Server.
   * If you wish to use the CoreDNS as your primary DNS Server see the [deploy-aio-lab.yml Extra Variables](https://github.com/cptmorgan-rh/ocp4-aio-vsphere-upi-lab#deploy-aio-labyml-extra-variables) section below.

   ***NOTE: If you are going to use the CoreDNS vm as your primary DNS Server you must specify your vcenter in group_vars/all.yml as an IP address as no A Record will exist.***

 ## Installation Steps

 ### Set Global Variables
 > Pre-populated entries in **group_vars/all.yml** are ready to be used unless you need to customize further. Any updates described below refer to [group_vars/all.yml](group_vars/all.yml) unless otherwise specified.
 1. Get the ***pull secret*** from [here](https://cloud.redhat.com/OpenShift/install/vsphere/user-provisioned). Update the file on the line with location of your `pull_secret`. ex. ~/openshift/pull-secret.json  
 2. Get the vCenter details:
    1. IP address
    2. Service account username (can be the same as admin)
    3. Service account password (can be the same as admin)
    4. Admin account username
    5. Admin account password
    6. Datacenter name *(created in the prerequisites mentioned above)*
    7. Datastore name
    8. Absolute path of the vCenter folder to use *(optional)*. If this field is not populated, its is auto-populated and points to `/${vcenter.datacenter}/vm/${infraID}`
 3. Downloadable link to `govc` (vSphere CLI, *pre-populated*)
 4. OpenShift cluster
    1. base domain *(pre-populated with **example.com**)*
    2. cluster name *(pre-populated with **ocp4**)*
 5. If you wish to install without enabling the Kubernetes vSphere Cloud Provider (Useful for mixed installs with both Virtual Nodes and Bare Metal Nodes), change the `provider: ` to `none` in all.yaml.
    ```
    config:
      provider: none
      base_domain: example.com
      ...
    ```
 6. If you wish to enable custom NTP servers on your nodes, set `ntp.custom` to `True` and define `ntp.ntp_server_list` to fit your requirements.
    ```
    ntp:
      custom: True
      ntp_server_list:
      - 0.rhel.pool.ntp.org
      - 1.rhel.pool.ntp.org
    ```

USAGE
------------

### Run Deploy Playbook
```sh
# Deploy the Lab and all components
ansible-playbook deploy-aio-lab.yml
```
### deploy-aio-lab.yml Extra Variables

`config_local_dns=true` - Configures /etc/resolv.conf or systemd-resolved to use CoreDNS as primary DNS after CoreDNS has been deployed.

`skip_ova=true` - Skips downloading and deploying the OVA if previous deployed to vCenter.

`skip_lb=true` - Skips deploying the LoadBalancer VM if a LoadBalancer already exists.

`skip_dns=true` - Skips deploying a DNS server if proper DNS is already configured.

`specific_version=4.6.z` - Deploys a specific version of OpenShift. Must be in 4.x.z format.

### Run Destroy Playbook
```sh
# Destroy the Lab and all components
ansible-playbook destroy-aio-lab.yml -e cluster=true

# Destroy the Lab and all components while retaining the ova
ansible-playbook destroy-aio-lab.yml -e cluster=true -e skip_ova=true

# Destroy the Lab and all components and revert DNS Configuration
ansible-playbook destroy-aio-lab.yml -e cluster=true -e config_local_dns=true
```
### destroy-aio-lab.yml Extra Variables

`cluster=true` - Required to delete the entire cluster, ova, folder, and vwware tag.

`skip_ova=true` - Skips downloading and deploying the OVA if previous deployed to vCenter.

`bootstrap=true` - To delete the bootstrap VM

***NOTE: The OVA file needs to be outside of the Cluster Folder in VMware for `-e skip_ova=true` to work***


### Expected Outcome

1. Necessary Linux packages installed for the installation
3. Necessary folders [bin, downloads, downloads/ISOs, install-dir] created
4. OpenShift client, install and .ova binaries downloaded to the **downloads** folder
5. Unzipped versions of the binaries installed in the **bin** folder
6. In the **install-dir** folder:
   1. master.ign and worker.ign
   2. Copy of the install-config.yaml
7. A folder is created in the vCenter under the mentioned datacenter and the template is imported
8. The template file is edited to carry certain default settings and runtime parameters common to all the VMs
9. VMs (coredns, lb, bootstrap, master0-2, worker0-2) are generated in the designated folder and (in state of) **poweredon**

```sh
# In the root folder of this repo run the following commands
export KUBECONFIG=$(pwd)/install-dir/auth/kubeconfig
export PATH=$(pwd)/bin:$PATH

# OpenShift Client Commands
oc whoami
oc get co
```

### Add additional worker nodes after deploying the cluster_name

1. Specify the worker_prefix in `group_vars\all.yml` under the config section.
2. Add a new line to `group_vars\all.yml` under `worker_vms`.

   Ex. `- { name: "worker3", ipaddr: "10.0.0.26", cpus: "2", memory_mb: "8192", disk_size_gb: "120"}`
3. Running `add-new-nodes.yml` will add all additional new worker nodes and redeploy CoreDNS and the HAProxy LB with the new node information.
   * To not redeploy the CoreDNS vm add the extra variable `skip_dns=true`
   * To not redeploy the HAProxy LB vm add the extra variable `skip_lb=true`
4. If you choose to redeploy the HAProxy LB vm you can scale the ingress controller by following these steps: [Scaling an Ingress Controller
](https://docs.openshift.com/container-platform/4.6/networking/ingress-operator.html#nw-ingress-controller-configuration_configuring-ingress)

## Special Thanks:

Vijay Chintalapati, Mike Allmen, and all the contributors to [ocp4-vsphere-upi-automation Repo](https://github.com/RedHatOfficial/ocp4-vsphere-upi-automation) that inspired this repository.

AUTHOR
------
Morgan Peterman
