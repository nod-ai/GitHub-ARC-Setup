# Linux ROCm ARC Config Files

## Changing the AWS credentials for `azure-linux-scale-rocm` runners

To change the AWS credentials, run `kubectl edit configmap -n arc-rocm-linux-runners aws-cred-config -o yaml`.

This will open a default text editor. After editing the yaml file, simply save it and the credentials will be updated. Please allow up to 2 minutes for changes to propogate.
