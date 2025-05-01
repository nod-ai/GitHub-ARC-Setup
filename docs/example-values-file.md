# GitHub Actions Runner Scale Set (GHARS) Configuration Documentation

This document explains the YAML configuration for a GitHub Actions Runner Scale Set.  It uses the pytorch 2 gpu scale set as an example.

## Overview

This configuration defines a Kubernetes-based GitHub Actions runner deployment that:
1. Uses AMD GPUs (ROCm)
2. Integrates Docker-in-Docker (DinD) capabilities
3. Configures specific resources for PyTorch workloads
4. Sets up proper permissions and networking

## Configuration Components

### Basic Scale Set Configuration

```yaml
minRunners: 1
```
- Sets the minimum number of idle runners

### Service Account Configuration

```yaml
controllerServiceAccount:
  name: arc-gha-rs-controller
  namespace: arc-systems
```
- Specifies the Kubernetes service account that the controller will use
- The controller runs in the `arc-systems` namespace

### Pod Template Specification

#### Security Context

```yaml
template:
  spec:
    securityContext:
      supplementalGroups:
        - 110
```
- Adds the runners to the supplemental group ID 110
- This is necessary on the Kubernetes cluster, because 110 is needed to run rocminfo in that environment

#### Node Selection

```yaml
nodeSelector:
    pytorch: "yes"
```
- Ensures runners are scheduled only on nodes labeled with `pytorch: "yes"`
- This targets nodes specifically prepared for PyTorch workloads

### Init Containers

#### Init Container 1: Copy External Dependencies

```yaml
initContainers:
  - name: init-dind-externals
    image: ghcr.io/saienduri/ghascale-rocm-dev:main
    imagePullPolicy: Always
    command:
      ["cp", "-r", "/home/runner/externals/.", "/home/runner/tmpDir/"]
    volumeMounts:
      - name: dind-externals
        mountPath: /home/runner/tmpDir
```
- Copies external dependencies from the container to a shared volume

#### Init Container 2: Docker-in-Docker Setup

```yaml
- name: dind
  image: ghcr.io/saienduri/dind:main
  restartPolicy: Always
  command: ["sh", "-c"]
  args:
    - |
      dockerd --host=unix:///var/run/docker.sock --group=${DOCKER_GROUP_GID} --data-root=/home/runner/docker-data &
      until docker info >/dev/null 2>&1; do sleep 5; done
      tail -f /dev/null
```
- Starts a Docker daemon inside the pod
- Sets up the Docker socket at `/var/run/docker.sock`
- Configures Docker to use a specific group ID (123)
- Waits until Docker is fully initialized
- Keeps the container running with `tail -f /dev/null`

Additional Docker-in-Docker configuration:
- Uses privileged mode (required for Docker-in-Docker)
- Mounts volumes for Docker data, work directory, socket, and externals
- Includes a pre-stop lifecycle hook to clean up Docker data

### Main Runner Container

```yaml
containers:
  - name: runner
    image: ghcr.io/saienduri/ghascale-rocm-dev:main
    imagePullPolicy: Always
```
- Uses a custom ROCm-enabled container image for the GitHub Actions runner
- Ensures the latest version is always pulled

#### GPU Device Setup

```yaml
command:
  - /bin/sh
  - -c
  - |
    devices=$(ls -la /dev/dri/ | grep renderD | awk '{print $10}')
    GHA_RENDER_DEVICES="--group-add 110"
    for device in $devices; do
      GHA_RENDER_DEVICES="${GHA_RENDER_DEVICES} --device /dev/dri/${device}"
    done
    echo "${GHA_RENDER_DEVICES}" > /etc/podinfo/gha-render-devices
    # Wait for Docker to be ready before starting runner
    echo "Waiting for docker..."
    until docker info >/dev/null 2>&1; do sleep 5; done
    /home/runner/run.sh
```
- Detects available AMD GPU render devices (`renderD*`)
- Creates a device mapping configuration for Docker containers that will run inside
- Waits for Docker to be ready before starting the runner
- Launches the runner with `/home/runner/run.sh`

#### Resource Allocation

```yaml
resources:
  requests:
    cpu: 40000m
    memory: 450000Mi
    ephemeral-storage: "100G"
    amd.com/gpu: 2
  limits:
    ephemeral-storage: "100G"
    amd.com/gpu: 2
```
- Requests substantial resources:
  - 40 CPU cores
  - 450GB of memory
  - 100GB of ephemeral storage
  - 2 AMD GPUs
- Sets hard limits on storage and GPUs

#### Environment and Volume Configuration

- Sets `DOCKER_HOST` to use the socket from the DinD container
- Captures pod and node names for reference
- Mounts volumes for:
  - Work directory (for GitHub Actions workflows)
  - Docker socket
  - External dependencies
  - Pod information
  - Docker data

### Volume Definitions

```yaml
volumes:
  - name: work
    emptyDir: {}
  - name: dind-sock
    emptyDir: {}
  - name: dind-externals
    emptyDir: {}
  - name: docker-data
    emptyDir: {}
  - name: podinfo
    emptyDir: {}
```
- Creates ephemeral volumes using Kubernetes `emptyDir`
- These volumes exist for the lifetime of the pod

### GitHub Configuration

```yaml
githubConfigUrl: https://github.com/pytorch
githubConfigSecret: "pytorch-secret"
```
- Links this runner to the PyTorch GitHub organization
- Uses a Kubernetes secret named "pytorch-secret" for authentication

## Execution Flow

1. Pod is scheduled on a node with the `pytorch: "yes"` label
2. Init container copies external dependencies to a shared volume
3. Docker daemon starts in the DinD init container
4. Main runner container:
   - Detects and configures GPU devices
   - Waits for Docker to be available
   - Starts the GitHub Actions runner
5. Runner registers with GitHub (PyTorch organization)
6. Runner begins accepting jobs based on availability

This configuration creates a powerful, GPU-enabled GitHub Actions runner that can execute demanding PyTorch workflows requiring significant compute resources and AMD GPU acceleration.
