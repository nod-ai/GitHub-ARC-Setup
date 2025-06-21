# ARC Setup

Documentation for bringing up a Kubernetes cluster integrated with GitHub Actions Runner Controller.

### Step 1: Create Azure Kubernetes Service (skip if kubernetes already setup on bare metal or other CSP)

Search for Kubernetes Service in the top search bar in Azure Portal. Once in, now click on Create -> Kubernetes Cluster. 
Choose your resource group and cluster name and proceed with default options for Basics.
Next, you should be in the Node Pools section. You will see two node pools (userpool and agentpool).
The agentpool is in System mode and is designed to host the critical system pods that Kubernetes needs to operate.
The userpool is the one we care about. It is in user mode and used is designed to host the applications and workloads that we deploy to our Kubernetes cluster.
The userpool is where our github actions jobs will be dispatched. The default VM being used for these nodes is Standard_D8ds_v5. This only has 8 cores, and we need more for our IREE/ROCm projects.

We ended up creating two user node pools with the `Standard_D96as_v5` (96 cpu cores) and `Standard_D48as_v4` (48 cpu cores) instance sizes.
Based on the resource reqeuests section of the [`azure-linux-scale-rocm.yaml`](./config-files/iree-org/azure-linux-scale.yaml) and [`azure-linux-scale-rocm.yaml`](./config-files/rocm/azure-linux-scale-rocm.yaml) values files for example, you can extrapolate how many pods can run in parallel on a node.

For the rest of the cluster creation options you can choose the default.

### Step 2: Login to your Cluster (skip if kubernetes already setup on bare metal or other CSP)

Now, to configure the cluster and all the services you need to connect to the cluster.
You can do this in your own local dev environment (just make sure you have kube, helm, and azure cli installed)
I just use the cloud shell that Azure provides. (Click `Connect` and then `Cloud Shell`)
Run these two commands to connect to your cluster:
```
az account set --subscription <your_subscription_number>
az aks get-credentials --resource-group <resource_group_name> --name <cluster_name> --overwrite-existing
```

# Latest GHA Runner Scale Set Instructions

![image](https://github.com/user-attachments/assets/e27a47e0-cdf6-4449-880e-b876a98f2bdf)


### Step 3: Install Actions Runner Controller

```
helm install arc --namespace "arc" --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

### Step 4: Configure and Deploy Runner Scale Set

```
helm upgrade --install "azure-linux-scale"     --namespace "<namespace_name_for_runners>"     --create-namespace  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set -f config-file.yaml
```

Please use the values.yaml file from `latest-config-files` folder in this repo for the above command.
The "azure-linux-scale" is the installation name and becomes the label you use for "runs-on:", so configure it however you see fit.

The values.yaml uses a custom docker image that is currently hosted in this repo: https://github.com/saienduri/docker-images/tree/main.
It also has CI configured to build and publish the image.
We need a custom image that builds off the gha scale set runner docker image provided, so that it has the deps that we require for CI and works on all our workflows.
Other things we configure in the values.yaml file is min runners = 3 and max runners = 30 for scaling.
The scaling setup is basically the same as the legacy documentation below, so please refer to that for further details.
Also, docker in docker is setup, so in our github workflows we can specify images to use if we want (iree uses cpubuilder_ubuntu_jammy image for example), but as done in iree-turbine, we can just run workflows using the preconfigured custom image here without further setup and that works too.

And you're done (just make sure label matches installation name in workflow) :)
