# Simulation Platform

The `simulation-platform` repository contains documentation on how to setup and manage a simulation cluster on your cloud provider.

## Introduction

The simulation cluster provides capability to run large scale simulations on cloud infrastructure using a Kubernetes&reg; cluster and cloud-native components.  Once the cluster is setup, then users can submit sim jobs to the cluster using the *Large-scale cloud simulation for Simulink&reg;* [support package](https://www.mathworks.com/help/slcompiler/large-scale-cloud-simulation-for-simulink-support-package.html?s_tid=CRUX_lftnav) for the MATLAB&reg; software as a client.

Using your cloud-provider account, the simulation cluster provisions a group of cloud resources, open-source components and default configuration to stand up the necessary infrastructure.

# mwcsim

`mwcsim` is a command line tool to easily manage simulation clusters on a cloud provider. It provides convenience functionality to create, delete and manage a cluster.  Advanced users are [encouraged to modify](advanced-configuration.md) the Terraform&reg; file and Kubernetes manifests underlying the cluster and apply them to get the customized cluster you need, especially for massive simulation needs or custom integration requirements with existing systems.

## Prerequisites

Currently the simulation cluster is only supported on the **Amazon Web Services (AWS)**&trade; cloud provider.  The `mwcsim` CLI is only supported on the **Linux**&reg; operating system.

- [AWS account](https://aws.amazon.com/)
- [AWS command line tool v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [OpenTofu&reg; software](https://opentofu.org/docs/intro/install/) for terraform management
- [Docker&reg; engine](https://docs.docker.com/engine/) is used to install the CLI, but not necessary for operation

### AWS security requirements

#### Cluster creation from scratch

If the simulation cluster is being created from scratch, we recommend a dedicated clean AWS account to minimize potential conflicts with existing resources or quota limits.  The AWS managed [PowerUserAccess](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/PowerUserAccess.html) security policy is required for the user creating the cluster.  More details about the AWS managed policies for job function can be read [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html).

#### Cluster creation on existing resources

*Feature currently unavailable, coming soon*

A simulation cluster can be created with reduced permission requirements using existing resources.  This feature is coming soon.

## Installation

The easiest method is to pull the `csim-mwcsim` container, which brings all the configuration files along with the CLI.

```
docker pull containers.mathworks.com/simulation-platform/csim-mwcsim:v0.5.27
```

Install the CLI with the following commands.  Note the mounted directory should remain `/tmp/mwcsim` for a smooth installation.  The `install.sh` command may prompt for `sudo` access.

```
mkdir -p /tmp/mwcsim
docker run --mount type=bind,source=/tmp/mwcsim,target=/mount containers.mathworks.com/simulation-platform/csim-mwcsim:v0.5.27
/tmp/mwcsim/install.sh
```

This installs the CLI into `/usr/local/bin` and the configuration files to `$HOME/.mwcsim`, which is the default working directory of mwcsim.  You can test the installation by checking the version.

```
mwcsim version
```

## Getting started

This section outlines quick introductory setups to get a cluster ready. Advanced usage documentation are further below.

For all commands below, *my-cluster*, *my-username*, *mydomain.example.com*, *loadbalancer-endpoint* are placeholders that should be replaced by your desired values. If you encounter any issues, please check the [known-issues](known-issues.md) page as well. A lot of problems are related to resource provisioning delays and a simple retry after a few minutes may resolve it.

### Manually provisioned domain

```
# STEP 1
# make sure aws access is setup
aws configure

# STEP 2
# create stable components of cluster first, that supports MATLAB version R2024bUpdate2 clients
# note that if you encounter issues, you can safely try rerunning the create command
# occasionally there are resource provisioning limits or delays that may cause an error
mwcsim create cluster my-cluster --domain mydomain.example.com -v R2024bUpdate2 -x stableblock

# STEP 3
# obtain loadbalancer endpoint, hereon referred to as loadbalancer-endpoint
mwcsim get cluster my-cluster -o yaml | grep loadbalancer

# STEP 4
# manually provision domains on your DNS provider
# the following records are necessary (all 4 point to the same loadbalancer endpoint)
#   CNAME mydomain.example.com -> Loadbalancer-endpoint
#   CNAME idp.mydomain.example.com -> loadbalancer-endpoint
#   CNAME oauth.mydomain.example.com -> loadbalancer-endpoint
#   CNAME registry-access.mydomain.example.com -> loadbalancer-endpoint

# STEP 5
# create the second half of the cluster components that are dependent on the domain
# note that if you encounter issues, you can safely try rerunning the create command
# occasionally there are DNS propagation and HTTPS certification delays that may cause an error
mwcsim create cluster my-cluster -x dynamicblock

# STEP 6
# create a first user, this will return an auto-generated password
# note the username and password
mwcsim create user my-username -c my-cluster

# STEP 7
# get the cluster endpoint and note this down
mwcsim get cluster my-cluster
```

Once the cluster is ready, use the MATLAB client [documentation](https://www.mathworks.com/help/slcompiler/large-scale-cloud-simulation-for-simulink-support-package.html?s_tid=CRUX_lftnav) to send large-scale simulation jobs to the cluster, using the cluster endpoint and user credentials from above.

### Automatically provisioned domain (beta)

`mwcsim` can automatically provision domains as an experimental feature. This feature is only supported on versions `R2024bUpdate3` and later. Automatically provisioned domains don't require DNS access or expertise. **Contact us if you're interested in learning more and using it.**

```
# STEP 1
# make sure aws access is setup
aws configure

# STEP 2
# create cluster
mwcsim create cluster my-cluster

# STEP 3
# create a first user, this will return an auto-generated password
# note the username and password
mwcsim create user my-username -c my-cluster

# STEP 4
# get the cluster endpoint and note this down
mwcsim get cluster my-cluster
```

Once the cluster is ready, use the MATLAB client [documentation](https://www.mathworks.com/help/slcompiler/large-scale-cloud-simulation-for-simulink-support-package.html?s_tid=CRUX_lftnav) to send large-scale simulation jobs to the cluster, using the cluster endpoint and user credentials from above.

## Create and destroy simulation clusters

For all commands below, *my-cluster* and *my-region* are placeholders that should be replaced by your desired values.

### Get help on the mwcsim command

You can get help on the root `mwcsim` command or any sub-commands like below.

```
mwcsim -h
mwcsim create -h
mwcsim create cluster -h
```

### Create a simulation cluster

Create a cluster in the default region (`us-east-1`) with a given name.

```
mwcsim create cluster my-cluster
```

Create a cluster in a specific region.

```
mwcsim create cluster my-cluster -r my-region -v R2024bUpdate2
```

Note the `-v` flag for the cluster version.  The currently supported versions are `R2024bUpdate2` and `R2024bUpdate3`.  The version is relevant as the cluster only supports the corresponding MATLAB clients of the same version or later.

#### Partially create a cluster

```
mwcsim create cluster my-cluster -x <operation>
```

The supported operations are:

```
"init"          initialize cluster in mwcsim backend
"tfplan"        run terraform plan
"tfapply"       run terraform apply
"kustomize"     build kustomization files
"kubeapply0"    apply wave 0 kubernetes resources
                  after "kubeapply0", the loadbalancer endpoint is available for DNS records
"domain"        provision domain automatically if not manually provisioned
"kubeapply1"    apply wave 1 kubernetes resources
"configure"     configure kubernetes resources
                  after "configure", the cluster should be ready to use
  ---
"all"           run all operations above (top > bottom) except tfplan
                  only successful with auto-provisioned domain
"stableblock"   run "init", "tfapply", "kustomize" and "kubeapply0" operations
                  this block of operations do not require a valid domain
"dynamicblock"  run "domain", "kubeapply1", "configure" operations
                  this block of operations require a valid domain, if manually provisioned
(default "all")
```

All operations above can be safely rerun if there are any issues during setup, so you can remedy the issue (such as exceeding max number of VPC or IP slots available) on the cloud-provider and retry where you left off.  The creation process proceeds from the top to bottom until the cluster is ready.  You can resume an operation at any point.

The partial setup is especially useful if you want to modify cluster terraform files or kubernetes files midway, after certain resources are provisioned or details are available.

### Get simulation clusters

Get a list of current simulation clusters.

```
mwcsim get cluster

# example output
NAME        ENDPOINT                  REGION     STORAGE                   STATE       VERSION
my-cluster  mycluster.example.com     us-east-2  csim-my-cluster-3iaopg0e  ITK0D--(-)  R2024bUpdate2
cluster2    ecgyp7msgpj7r1.mwcsim.io  us-east-1  csim-cluster2-bzdxbn4f    ITK0D1C(R)  R2024bUpdate3
```

Get a specific cluster.

```
mwcsim get cluster my-cluster
```

Output details in JSON or yaml.

```
mwcsim get cluster -o json
mwcsim get cluster my-cluster -o yaml
```

#### Meaning of the state field

The state field signifies in short hand what the cluster state is currently. You can use this as a guide to understand which operations to perform when partially creating or destroying a cluster.  The states are summarized as follows:

```
(I)initialized     mwcsim knows about the cluster and tracks it, but no cloud provider resources are provisioned
(T)erraform        analogous to "terraform apply", cloud resources are provisioned 
(K)ustomize        similar to step before "kustomize build", where the cluster specific kustomization files are generated
KubernetesWave(0)  analogous to "kubectl apply", for certain subset of objects that are domain independent
(D)omain           step where a cluster domain is provisioned, this can be automatically or manually done
KubernetesWave(1)  analogous to "kubectl apply", for the rest of objects that are domain dependent
(C)onfigured       configure identity provider and other microservices
(R)eady            cluster is ready for use
```

So a cluster with state `I------(-)` signifies that it's only in the initialized state, with no terraform or kubernetes resources applied (hence no cloud resources provisioned at all).

A cluster in the `ITK0D1C(R)` state is fully provisioned and ready to use.

As another example, a cluster in the `ITK----(R)` has the terraform resources applied, and the cluster specific kustomization files generated, but no kubernetes resources are created or configured yet.

**Note that the cluster state field is a guideline only.** It is not the source of truth for the cluster state. `mwcsim` uses the actual cluster resources as a source of truth. So the state field might be inaccurate depending on actions or circumstances outside of the `mwcsim` usage (for example, someone deleted a resource manually).

### Delete a simulation cluster

Delete a cluster and all associated resources, except the data.

```
mwcsim delete cluster my-cluster
```

The command above will error out at the end with one remaining resources which is the storage location, if it's not empty.

If you want to delete the data, you can do so with a flag.

```
mwcsim delete cluster my-cluster --delete-data
```

**WARNING: The data is deleted permanently and is unrecoverable.**

#### Partial destruction of a cluster

As with the creation commands, you can partially delete the cluster as below.

```
mwcsim delete cluster my-cluster -x <operation>
```

The supported operations are:

```
specify granular operations to perform, must be one of:
  "tfplan"         run terraform plan
  "kubedelete1"    delete wave 1 kubernetes resources
  "releasedomain"  release domain if auto-provisioned
  "kubedelete0"    delete wave 0 kubernetes resources
  "deletedata"     delete storage contents (--delete-data flag also required)
  "tfdestroy"      run terraform destroy
  "deinit"         de-initialize cluster from mwcsim backend
  "all"            run all operations (top > bottom) except tfplan
(default "all")
```

All operations above can be rerun if there are any issues during deletion, so you can remedy the issue (such as cannot remove non-empty storage) and retry where you left off.

## Configure the simulation cluster

### Get the simulation cluster endpoint

Once a cluster is created, you can use the endpoint returned by `mwcsim` to submit jobs and access the cluster administration tools.  Follow the domain manually provisioning steps above in the getting started section to provide your own domain.

```
mwcsim get cluster my-cluster
```

Output:

```
# example output
NAME        ENDPOINT                  REGION     STORAGE                   STATE       VERSION
my-cluster  mycluster.example.com     us-east-2  csim-my-cluster-3iaopg0e  ITK0D--(-)  R2024bUpdate2
```

The main endpoint is used to submit jobs to the cluster from clients.  The admin endpoint is on another URL similar to the endpoint and is used to manage the cluster and users.  The admin endpoints are different depending on the version of the cluster, as outlined below. 

```
#admin endpoint
FORMAT           EXAMPLE                     VERSION
idp.<endpoint>   idp.mycluster.example.com   R2024bUpdate2
<endpoint>/auth  mycluster.example.com/auth  R2024bUpdate3
```

### Admin credentials

After the cluster is successfully created, the admin credentials can be obtained as below.

```
mwcsim get user admin -c my-cluster --credential
```

Note the automatically created `admin` user can only be used for cluster management.  You'll need to create users to actually utilize the cluster.

### Create users

A convenience function is provided to easily create a user.

```
mwcsim create user my-username -c my-cluster
```

For full user management features, access the admin endpoint. The cluster uses the [Keycloak&reg; software](https://www.keycloak.org/) by default to manage users and authentication. You can manage users under the `mwcsim` realm created for the cluster using the Keycloak instructions [here](https://www.keycloak.org/getting-started/getting-started-kube#_create_a_user).  **Make sure to create and manage users in the `mwcsim` realm and not the `master` realm.**  Users created in the `master` realm will not work.

### Optional single sign-on (SSO)

If you want to connect to an external identity provider and use SSO to log into the simulation cluster, you can use the keycloak admin interface to set it up.  Instructions can be found in the Keycloak [documentation](https://www.keycloak.org/docs/latest/server_admin/).

## Using the simulation cluster

Currently only the MATLAB client is supported to work with the cluster.  New clients will be added in the future.

### Connect to the simulation cluster

Once you have the cluster endpoint and user created, you can connect to it from MATLAB. Follow the instructions on the MATLAB support package "Large-scale cloud simulation for Simulink" [documentation](https://www.mathworks.com/help/slcompiler/large-scale-cloud-simulation-for-simulink-support-package.html?s_tid=CRUX_lftnav) to run large scale simulations.

# Advanced Configuration

Instructions on advanced configuration of the underlying Terraform files and Kubernetes manifests can be found [here](advanced-configuration.md).

## Security

The default simulation cluster created is publicly accessible by authenticated users via encrypted HTTPS. The simulation and model data is stored in a private S3&trade; bucket, where the workers read input data from and write output data to. It is recommended that users modify the Terraform, following the advanced configuration instructions above, to set the VPC to a private [peered network](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html) or set a [security group](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html) such that the access is restricted to your VPN IP ranges.

Additionally, comprehensive security monitoring services such as the [AWS GuardDuty&trade;](https://aws.amazon.com/guardduty/) service can be setup on your own to monitor the cluster.

# Support

### Known issues and troubleshooting

Known issues and troubleshooting details can be found [here](known-issues.md).

### Request enhancements

To suggest additional features or capabilities, see
[Request Reference Architectures](https://www.mathworks.com/products/reference-architectures/request-new-reference-architectures.html).

### Get technical support

If you require assistance, contact [MathWorks Technical Support](https://www.mathworks.com/support/contact_us.html).

### License

MATHWORKS CLOUD REFERENCE ARCHITECTURE LICENSE © 2024 The MathWorks, Inc.
