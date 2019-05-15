# vSphere UPI

You should always follow the documentation [here]](https://github.com/openshift/installer/tree/master/upi/vsphere) and [here](https://github.com/openshift/installer/blob/master/docs/user/vsphere/install_upi.md) first, it's almost certainly more up-to-date.

## Pre-reqs

* If you're using Terraform to provision VMs and manage DNS (e.g. Route53), you'll need to install it.
* You'll also need an HTTP(S) server (or somewhere public, e.g. a [gist](https://gist.github.com)), to host the bootstrap ignition file(s).

**Internet access is required!**  At this time, there is no offline, local, or disconnected install process.

## Information you'll need:

* vCenter credentials with enough permissions to create and manage virtual machines.  More specifically, cloning the RHCOS template into the datastore desired, edit VM properties and resources (CPU, RAM, network, etc.), power on/off, and removing VMs.
  
  You'll also need the following info from vCenter
  * The datacenter name
  * The cluster name
  * The network name
  * The datastore name
* Network information
  * IP addresses for all VMs (bootstrap + masters + workers)
  * CIDR, e.g. `10.0.0.0/24`
  * Gateway
  * DNS servers
* Your pull secret from [try.openshift.com](https://try.openshift.com)
* The public SSH key for the user(s) you want to have access to the nodes (username: `core`)

# Install

1. Configure the load balancer according to [the documentation](https://github.com/openshift/installer/blob/master/docs/user/vsphere/install_upi.md#load-balancers)
   
   The sample Terraform will use Route53 to configure a subdomain (`$clustername.$basedomain`), DNS records, and a round robin DNS entry as the "load balancer".
   
1. Create a working directory
   
   ```bash
   # for simplicity, use the cluster name for the folder name
   mkdir ~/clustername && cd ~/clustername
   ```
   
1. Create the `install-config.yaml` in the working directory
   
   ```yaml
   apiVersion: v1beta4
   baseDomain: MYDOMAIN.TLD
   metadata:
     name: CLUSTER_NAME
   networking:
     machineCIDR: "10.0.0.0/24"
   platform:
     vsphere:
       vCenter: VCENTER.MYDOMAIN.TLD
       username: VCENTER_USERNAME
       password: VCENTER_PASSWORD
       datacenter: VCENTER_DATACENTER
       defaultDatastore: VCENTER_DATASTORE
   pullSecret: 'YOUR_PULL_SECRET'
   sshKey: 'YOUR_SSH_KEY'
   ```
   
   **Note:** This file will be deleted in the next step, make a copy if you don't want to re-create it from scratch!
   
   * `baseDomain`, e.g. "mycompany.com" is the top-level domain used for the cluster.
   * `metadata.name`, e.g. "ocp4-dev", is the cluster name.  Along with the `baseDomain`, this will become part of the DNS name, e.g. "ocp4-dev.mycompany.com
   * `machineCIDR`, e.g. "10.0.0.0/24"
   * vCenter info
     * `vCenter` is the hostname or IP address of your vCenter
     * `username` and `password` are self explanatory
     * `datacenter` is the name of the vCenter datacenter
     * `defaultDatastore` is the datastore which the VMs will be created in
   * `pullSecret` is your pull secret from [try.openshift.com](https://try.openshift.com)
   * `sshKey` is the public key used for authenticating your private key
   
1. Create ignition configs
   
   ```bash
   # after this command completes, the install-config.yaml file is deleted, make a copy if needed
   openshift-install create ignition-configs
   ```
   
   This will create the `bootstrap.ign`, `master.ign`, and `worker.ign` files.  Additionally, the `kubeconfig` and `kubeadmin` password files are found in the `auth` directory.
   
1. Host the `bootstrap.ign`
   
   The file must be in a location the bootstrap VM will be able to access it, such as an internal HTTP(S) server.  A [gist](https://gist.github.com/) will also work.
   
1. Provision the virtual machines
   
   Some instructions for the sample Terraform are [here](#using_terraform_to_create_vms) or manually creating the VMs [here](#manually_creating_vms).
   
1. Wait for cluster bootstrap
   
   ```bash
   # append "--log-level debug" to get more details
   openshift-install wait-for bootstrap-complete
   ```
   
   * You may monitor the progress of the bootstrap by connecting to the host and following the logs.  See the [troubleshooting](#troubleshooting) section on how to connect.
     
   * If you used the sample Terraform to provision the VMs, destroy the bootstrap VM after this step using the command `terraform apply -auto-approve -var 'bootstrap_complete=true'`.
     
   * The bootstrap VM's IP address should also be removed from the load balancer at this point.  If you used the sample Terraform (with Route53 DNS), remove the DNS RR entry for the bootstrap to prevent occasional API failures.  There will be records for `api` and `api-int` which need to be updated.
   
1. Wait for install complete
   
   ```bash
   # append "--log-level debug" to get more details
   openshift-install wait-for install-complete
   ```
   
   You may need to complete some additional tasks as well...
   
   * Approve pending node CSRs
     
     ```bash
     # approve all pending CSRs
     oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
     ```

   * Modify registry storage to use an `emptyDir`
     
     ```bash
     # patch the config to use the emptyDir
     oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"filesystem":{"volumeSource": {"emptyDir":{}}}}}}'
     ```
     
   If you make the above modification to the registry, be aware of the limitations of using an `emptyDir` and decide if you want to use different storage after the initial deployment is complete
   
1. Profit
   
   At this point your cluster is deployed and ready to start hosting workloads.  Use the URL provded at the end of the last command to browse to the console or the credentails found in the `./auth` directory to connect using the `oc` client.
   
   Be sure to follow the [docs](https://docs.openshift.com/container-platform/4.1/) and use the [training repo](https://github.com/openshift/training) to get familiar with OCP4 (node scaling doesn't work with vSphere UPI).

# Teardown

If you used the sample Terraform to provision the VMs, etc. then use the command `terraform destroy -auto-approve` to remove everything.

# Other

## Troubleshooting

You can connect to the VMs using the user `core` and key based auth, e.g. `ssh -i ~/ocp/id_rsa core@bootstrapFQDNorIP`.

## The RHCOS OVA

1. Download the RHCOS OVA
   
   The RHCOS image can be found [here](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/).  
   
1. After downloading, import to vCenter
   
   You don't *need* to modify any of the configuration when importing the OVA, however you can set the network to something valid if `VM Network` doesn't exist in your environment.  There is no need to set any values for the vApp properties at this point.
   
1. After the import, convert the virtual machine to a template

## Using Terraform to create VMs

The sample Terraform can be found [here](https://github.com/openshift/installer/tree/master/upi/vsphere).  Some documentation for using the sample Terraform can be found [here](https://github.com/openshift/installer/blob/master/docs/user/vsphere/install_upi.md#example-vsphere-upi-deployment).

1. Download and import the RHCOS template
   
   Follow [the instructions above](#the_rhcos_ova).

1. Clone the installer GitHub repo, copy all of the files from the `upi/vsphere` directory
   
   ```bash
   # clone the installer repo
   git clone https://github.com/openshift/installer.git
   
   # create a working directory and copy the files needed to create the VMs using Terraform
   mkdir ~/vsphere-upi && cp -R installer/upi/vsphere/* ~/vsphere-upi
   
   # optionally, remove unnecessary files
   cd ~/vsphere-upi && rm OWNERS README.md
   ```
   
1. Complete the `terraform.tfvars` file with your values  
   
   **NOTE:** At this time, not all variables in the `variables.tf` are in the `terraform.tfvars` pulled from the repo.  For example, if you need to change the network for the deployed VMs to something other than `VM Network`, use the variable `vm_network`.  Double check to make sure that all of the things you need are in there.
   
   ```bash
   # copy the file
   cp terraform.tfvars.example terraform.tfvars
   
   # edit the file
   vim terraform.tfvars
   ```
   
   If you're using a gist to host the `bootstrap.ign`, be sure to use the URL for the raw file!
   
1. (Optionally) Customize the virtual machine configuration
   
   * `machine/ignition.tf` has the network subnetmask, gateway, and DNS info
   * `machine/main.tf` has the VM resource sizing (CPU, RAM) and disk size info
   
* Initialize Terraform (if you haven't done so before)
    
    ```bash
    # this downloads all the modules, etc.
    terraform init
    ```
    
* Create the VMs
    
    ```bash
    # use this command to get a preview of the actions it will take
    terraform plan
    
    # to deploy the resources
    terraform apply -auto-approve
    ```
    
Terraform will take several steps...

* If you are allowing Terraform to configure Route53 as your DNS and load balancer, the first step is to configure these resources using AWS.  See [below](#separationg_route53_config) for how to separate and/or remove Route53 config if you're using other tools for a load balancer and DNS
* The template VM will be cloned $MASTER_COUNT + $WORKER_COUNT + 1 times.  By default, this will be 7 (3 masters, 3 workers, and bootstrap)
* Terraform will generate Ignition configs for the VMs which include network info and where to look for additional config.  This will be added to the cloned VMs as vApp properties.
* The VMs will be automatically powered on and RHCOS will configure according to the Ignition provided
  * The bootstrap's Ignition configures the hostname and network, then points to the Ignition file hosted on the external web server
  * The master Ignition configures hostname and network, then tells them to look to the bootstrap
  * The worker Ignition configures hostname and network, then tells them to look to the masters (a.k.a. control plane)

## Manually creating VMs

This section is not complete at this time.

## Separating Route53 config

When managing the VMs using Terraform and statically assigned IPs, it probably doesn't make sense to constantly create and destroy the Route53 information over and over since it doesn't change.

To separate DNS config, we can follow a few simple steps:

1. Create a new folder location
   
   ```bash
   # create a folder
   mkdir dns
   ```
   
1. Copy/move the information needed to the folder
   
   ```bash
   # move or copy the route53 folder from the main
   mv vsphere-upi/route53 dns/
   
   # copy, or link, the variables info
   cp vsphere-upi/terraform.tfvars dns/
   cp vsphere-upi/variables.tf dns/
   ```
   
1. Create a new `main.tf`
   
   We need to provide the IP addresses in this file, so put your IPs in the right place below.

   ```
   module "dns" {
     source = "./route53"
     
     base_domain         = "${var.base_domain}"
     cluster_domain      = "${var.cluster_domain}"
     bootstrap_count     = "${var.bootstrap_complete ? 0 : 1}"
     bootstrap_ips       = ["BOOTSTRAP_IP"]
     control_plane_count = "${var.control_plane_count}"
     control_plane_ips   = ["CONTROL_1_IP","CONTROL_2_IP","CONTROL_3_IP"]
     compute_count       = "${var.compute_count}"
     compute_ips         = ["COMPUTE_1_IP","COMPUTE_2_IP","COMPUTE_3_IP"]
   }
   ```
   
1. Create the DNS resources
   
   ```bash
   # init
   terraform init
   
   # (optional) plan
   terraform plan
   
   # apply
   terraform apply -auto-approve
   ```
   
1. Remove the DNS section from the other `main.tf`
   
   Since we don't need to create/destroy DNS records with each VM instnatiation, comment out that section of the `main.tf` in the primary working directory (`~/vsphere-upi` in my example).
   
   Wrap the section in `/* */` to comment out all the lines at once:

   ```
   /*
   module "dns" {
     source = "./route53"
   
     base_domain         = "${var.base_domain}"
     cluster_domain      = "${var.cluster_domain}"
     bootstrap_count     = "${var.bootstrap_complete ? 0 : 1}"
     bootstrap_ips       = ["${module.bootstrap.ip_addresses}"]
     control_plane_count = "${var.control_plane_count}"
     control_plane_ips   = ["${module.control_plane.ip_addresses}"]
     compute_count       = "${var.compute_count}"
     compute_ips         = ["${module.compute.ip_addresses}"]
   }
   */
   ```
   
1. Continue as normal.  Create and destroy VMs using Terraform as needed, it won't include Route53 DNS in the operation.