# Azure-AKS-ARC Setup on Windows

## Step 1

When setting up your Kubernetes cluster, add a second user pool using windows 2022.  you can optionally add a taint in the setup screen, but these instructions have you add it later through the cmd line.  It is assumed that the userpool is named `usrpl` throughout these instructions, as windows enforces a 6 character limit on pool names

## Step 2

Login to your cluster normally

```
az account set --subscription <your_subscription_number>
az aks get-credentials --resource-group <resource_group_name> --name <cluster_name> --overwrite-existing
```

## Step 3

apply a taint, so that the cluster knows to assign workflows to the windows node pool

```
kubectl taint nodes aksusrpl000000 'kubernetes.io/os=windows:NoSchedule'
```

if you follwed the instructions and used the same variable names, the windows node should be named `aksusrpl000000`, otherwise you can find it with:

```
kubectl get nodes -A
```

### Optional Alternative

You can also complete steps 1 and 3 after connecting to your cluster using the following command

```
az aks nodepool add --resource-group <resource group> --cluster-name <cluster name> --name usrpl --node-count 1 --node-osdisk-size 100 --node-vm-size Standard_D8ds_v5 --os-type Windows --os-sku Windows2022 --kubernetes-version "1.29.8" --node-taints 'kubernetes.io/os=windows:NoSchedule'
```

this creates the node pool and adds the taint

## Step 4

Install the controller
```
helm install arc --namespace arc-systems --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

## Step 5

create the external azure storage disk, that will be mounted to the runner

```
kubectl apply -f windows-config-files/pvc-disk.yaml
```

## Step 6

Install the control set

```
helm install "arc-runner-set" -f windows-config-files/values.yaml --namespace "arc-runners" --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Using a docker image that can be found here https://github.com/eliasj42/arc-windows-runner/pkgs/container/arc-windows-runner

## Possible error

If running step 5 gives an error about namespaces, do step 6, then step 5, then run the following command

```
helm upgrade --install "arc-runner-set" -f windows-config-files/values.yaml --namespace "arc-runners" --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```
