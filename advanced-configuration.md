# Advanced Configuration

The `mwcsim` CLI is meant to facilitate easy setup and management of simulation clusters, using sensible defaults that work out of the box. However, there are many cases where advanced configuration is necessary to create customized clusters that meet unique requirements. As such, simulation clusters are based on industry standard infrastructure-as-code tooling: Terraform&reg;, Kubernetes&reg; manifests and Kustomize&reg;. These can be modified directly to create the desired cluster.

Configuration can be broadly categorized into three major pieces:
- Configuration of the `mwcsim` CLI itself
- Terraform files describing the resources to provision from the cloud provider
- Kustomization and Kubernetes manifests describing the microservices to apply onto the kubernetes cluster

## Configuration of the `mwcsim` CLI

Most relevant `mwcsim` options can be found in the `-h` help commands.

### Persistent configuration of `mwcsim`

Normally the `mwcsim` CLI is configured with flags at the time of the command call. The flag configurations can be applied persistently with two methods: a config file or environment variables. The order of precedence if multiple conflicting configuration are provided are:

```
flag > environment variable > config file > default
```

#### `mwcsim` configuration with environment variables

Environment variables follow the form `MWCSIM_FLAG` where the `FLAG` is the `mwcsim` options obtained from the help `-h` output, all capitalized.

For example, for setting region:

```
# set config
export MWCSIM_REGION=us-west-1

# call mwcsim which will create cluster in us-west-1 region
mwcsim create cluster mycluster
```

#### `mwcsim` configuration with a config file

`mwcsim` reads a default config file at `$HOME/.mwcsim/config` if it exists. This can be modified with the `--config` option. The config file follows the format:

```
flag: value
```

Where the `flag` is the `mwcsim` options obtained from the help `-h` output.  For example:

```
# create config file in default location
cat <<EOF > $HOME/.mwcsim/config
region: us-west-1
EOF
```

### Location of the Terraform and Kubernetes files

When `mwcsim` is installed, it uses a default working directory in the user's home directory and sets up the configuration files there. From now on this directory is referred to as the *workdir*.

```
# default workdir
$HOME/.mwcsim

# workdir can be updated with the following command
mwcsim -w /path/to/custom/working/directory
```

Note that if the flag `-w` is used, the workdir must be provided to every call of the command, otherwise it'll revert to the default. See `mwcsim` persistent configuration above to set it more permanently.

## Terraform configuration

The terraform files describe the infrastructure resources that are provisioned from the cloud-provider.  They reside in:

```
workdir/terraform/v1-tofu
```

These can be modified as needed to fine-tune the cluster parameters.  Most relevant are the nodegroup minimum and maximum number of instances defined in the `cluster.tf` file.

```
    generalscale = {
      instance_types = ["t3.xlarge"]

      min_size = 0
      max_size = 100
      # this value is ignored after the initial creation
      # cluster autoscaler controls desired size
      desired_size = 2

      labels = {
        nodepool = "generalscale"
      }
    }
```

To create a larger cluster, increase the maximum number of instances in the `generalscale` nodegroup specifically. The `generalscale` nodegroup is designed to host the microservices that scale up by large numbers to accommodate large scale simulations.  The `critical` nodegroup is designed to host the critical services (some are singleton) for the cluster that shouldn't be preempted or moved (such as what happens during a scale down).

**Note: It is recommended to only modify the terraform files while no cluster is created.** Otherwise a state mismatch may occur that'll prevent deletion or management of existing clusters.

### `mwcsim` corresponding operation

The Terraform files are applied during the `tfapply` operation. This operation also runs during the `all` and `stableblock` operations.

```
mwcsim create cluster my-cluster -x tfapply

# also runs during
mwcsim create cluster my-cluster
mwcsim create cluster my-cluster -x stableblock
```

Any Terraform operations can be analyzed before applying, with the `tfplan` operation.

```
mwcsim create cluster my-cluster -x tfplan
```

The above `tfplan` and `tfapply` operations are similar to the direct Tofu/Terraform commands:

```
cd $HOME/.mwcsim/terraform/v1-tofu
tofu plan
tofu apply
```

With the exception of the input variables that need to be provided.

### Terraform backend

`mwcsim` uses the cloud provider storage system to store the Terraform state backend. So multiple users can edit the same cluster and it'll be safe. Additionally, any local Terraform files can be safely deleted and recreated from a fresh `mwcsim` install without losing existing cluster state.

## Kubernetes manifests

After the cloud provider resources are provisioned and the Kubernetes cluster is created, `mwcsim` applies the Kubernetes objects, using Kustomize, from the following directories.

```
workdir/manifests/v1/my-cluster-sdf24d2/deploy0
workdir/manifests/v1/my-cluster-sdf24d2/deploy1
```

The above directories are generated after the cluster is created in `mwcsim` and the `kustomize` operation is run.  The `kubeconfig` files to access the cluster via `kubectl` are also created in the cluster directories above.  So the following command can be used to administer the cluster directly:

```
# use the kubeconfig file directly
kubectl get pods -A --kubeconfig $workdir/manifests/v1/$mycluster/kubeconfig-$mycluster

# download the kubeconfig from AWS
aws eks update-kubeconfig --region $(mwcsim get cluster $mycluster -o json | jq -r .region) --name $(mwcsim get cluster $mycluster -o json | jq -r .rid)
kubectl get pods -A
```

### `mwcsim` corresponding operation

#### Kustomize build

The `kustomize` operation builds the `kustomization.yaml` files for the cluster, based off the `base` resources and the template files:

```
clusterKustomization0.tmpl
clusterKustomization1.R2024bUpdate2.tmpl
clusterKustomization1.tmpl
```

And outputs them into the `deploy0` and `deploy1` folders.  Be aware that the `kustomization1.yaml` in the `deploy1` folder for wave 1 is not accurate until right before the `kubeapply1` operation.

#### Kubernetes wave 0

Wave 0 encompasses the kubernetes resources that don't have a dependency on the ingress and cluster domain. These are applied immediately after the terraform resources are provisioned and correspond to the `kubeapply0` operation.

```
mwcsim create cluster my-cluster -x kubeapply0

# equivalent to
kubectl apply -k workdir/manifests/v1/my-cluster-sdf24d2/deploy0
```

Most of the manifests in this stage can be modified from the base wave 0 location:

```
workdir/manifests/v1/base/deploy0
```

**Note: It is recommended to only modify the base manifest files while no cluster is created.** Otherwise a state mismatch may occur that'll prevent deletion or management of existing clusters. If the cluster specific `kustomization.yaml` files are modified, this is not an issue. However, note that the `kustomization.yaml` files will be deleted and regenerated for each `kubectl apply` operation, so the templates should be modified instead.

#### Kubernetes wave 1

Wave 1 encompasses the kubernetes resources that depend on the ingress and cluster domain. These are applied after the loadbalancer and a domain is secured and correspond to the `kubeapply1` operation.

```
mwcsim create cluster my-cluster -x kubeapply1

# equivalent to
kubectl apply -k workdir/manifests/v1/my-cluster-sdf24d2/deploy1
```

Most of the manifests in this stage can be modified from the base wave 1 location:

```
workdir/manifests/v1/base/deploy1
```

**Note: It is recommended to only modify the base manifest files while no cluster is created.** Otherwise a state mismatch may occur that'll prevent deletion or management of existing clusters. If the cluster specific `kustomization.yaml` files are modified, this is not an issue. However, note that the `kustomization.yaml` files will be deleted and regenerated for each `kubectl apply` operation, so the templates should be modified instead.

## Recovering from mistakes and resetting

If the Terraform and Kubernetes manifests are ever mistakenly messed up where the cluster cannot be created, the original files can be recovered simply by deleting the workdir and its contents and re-installing the same version of `mwcsim`.  Any existing clusters will be resynced with the state saved on the cloud storage backend.
