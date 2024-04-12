---
title: Setting Up a Kubernetes Cluster on Raspberry Pi with k0s and Deploying Actions Runner Controller
date: 2024-04-04 09:57:27+0800
categories:
  - GitHub Actions
tags:
  - k0s
  - kubernetes
  - k8s
  - raspi
  - container
  - github-actions
---

I own a [repository](https://github.com/ukewea/python-talib) which builds Docker (container) image for amd64, arm64, arm/v7 architectures.

Building amd64 variant is fast, but building arm64 and arm/v7 images on GitHub-hosted Actions Runner is a disaster. Although I don't care about build time (image build happens while I'm asleep), but it's a good chance for me to investigate any way to reduce the build time.

![](2024-04-04-08-27-22.png)

Since the image build of ARM variants ran on AMD64 machines with QEMU, I think it's possible to speed up by using GitHub Actions Runners running in ARM architecture environments.

Since GitHub doesn't provide such runner to public (as of April 2024), we have to setup our own.

So in this guide, I will walk through the process of setting up a single node Kubernetes cluster on a Raspberry Pi using k0s, and then deploying an Actions Runner Controller to manage our GitHub Actions runners.

## Hardware & OS

I'll be setup a Raspberry Pi 4 running Ubuntu 23.10 (arm64). Only one machine is required as I am setting the cluster with only one single node.

I don't need HA for my hobby project.

## Pre-requisites

I followed some steps on the [official k0s documentation](https://docs.k0sproject.io/v1.23.6+k0s.2/raspberry-pi4/).

Ensure the following packages are installed on your Raspberry Pi :

```bash
sudo apt-get update && sudo apt-get install cgroup-lite cgroup-tools cgroupfs-mount
```

## Deploy k0s on the Pi 4

Before installing GitHub Actions Runner Controller (ARC), we need a kubernetes cluster. For quick and dirty setup on a resource constrained machine, I chose k0s and deployed in single mode configuration.

### Kernel Configuration

Enable memory cgroup in the kernel by modifying the kernel command line:

```bash
echo "$(cat /boot/firmware/cmdline.txt) cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1" | sudo tee /boot/firmware/cmdline.txt
```

Load necessary kernel modules:

```bash
echo "overlay
nf_conntrack
br_netfilter" | sudo tee /etc/modules-load.d/modules.conf
```

Reboot the Raspberry Pi to apply the changes:

```bash
sudo reboot
```

### Installing k0s

Download and install k0s on the Raspberry Pi:

```bash
curl -sSLf https://get.k0s.sh | sudo sh
```

Create initial k0s configuration file:

```bash
sudo mkdir -p /etc/k0s
k0s config create | sudo tee /etc/k0s/k0s.yaml
```

Modify the `extensions.helm` section in `/etc/k0s/k0s.yaml` configuration to include OpenEBS helm charts:

```yaml
  extensions:
    helm:
      concurrencyLevel: 5
      repositories:
      - name: openebs-internal
        url: https://openebs.github.io/charts
      charts:
      - name: openebs
        chartname: openebs-internal/openebs
        version: "3.9.0"
        namespace: openebs
        order: 1
        values: |
          localprovisioner:
            hostpathClass:
              enabled: true
              isDefaultClass: true
```

Deploy k0s as a single node cluster (control plane & worker on the same machine):

```bash
sudo k0s install controller --single -c /etc/k0s/k0s.yaml
sudo systemctl start k0scontroller.service
sudo systemctl status k0scontroller.service
```

The service should be running now, as shown below:

![](2024-04-04-08-50-01.png)

### Configuring kubectl

For easier access to kubectl:

```bash
mkdir -p ~/.kube
chmod 700 ~/.kube
sudo cp /var/lib/k0s/pki/admin.conf ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

export KUBECONFIG=$HOME/.kube/config
alias kubectl='k0s kubectl'
echo "export KUBECONFIG=$HOME/.kube/config" >> ~/.bashrc
echo "alias kubectl='k0s kubectl'" >> ~/.bashrc
```

Let's pause and test the configuration, type `kubectl get pods -A` in your terminal, you should see some pods running:

![](2024-04-04-08-50-48.png)

### Helm Installation

Install Helm 3 to manage Kubernetes applications:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Install GitHub Actions Runner Controller And Runner Scale Set

### Deploying Actions Runner Controller

Set up the Actions Runner Controller using Helm:

```bash
NAMESPACE="arc-systems"
helm install arc \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

### Create a GitHub App for Runner Scale Set

Runner Scale Set needs to interactive with your organization/repo when register/deregister runner, so we have either provide a Personal Access Token or create a GitHub App for it.

I go for GitHub Apps this time, it is more complicated but offers some degress of security comaring to provide a PAT.

In your organization settings, navigate to "Developer Settings" > "GitHub Apps".

![](2024-04-11-23-48-04.png)

Click "New GitHub App"

![](2024-04-11-23-49-03.png)

Select required permissions as per official documentation [here](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api#authenticating-arc-with-a-github-app).

After created the App, click Generate a private key and save the .pem file.

![](2024-04-11-23-57-48.png)

In the menu at the top-left corner of the page, click Install app, and next to your organization, click Install to install the app on your organization.

![](2024-04-12-00-02-33.png)

After confirming the installation permissions on your organization, note the app installation ID. You will use it later. You can find the app installation ID on the app installation page, which has the following URL format:

https://github.com/organizations/ORGANIZATION/settings/installations/INSTALLATION_ID

Remember `App ID`, `Installation ID`, and private key, you'll need them later.

### Configure Runner Scale Set

Create a configuration file called `runner-scale-set-values.yaml` for runner scale set and define your preferences:

* `githubConfigUrl`: the URL of your repository/organization/enterprise where you are going to register your runners to.
* `runnerScaleSetName`: the label (or `tag` if you are coming from GitLab CI/CD) you will write in your workflow to assign job to these runners.
* `maxRunners`, `minRunners`: you guessed it.
* `containerMode`: you could use `kubernetes` type of `dind` type. Please tell me their difference if you know it, thanks. Uncomment the text below for your preferred mode. I chose to able to switch between 2 modes quickly during development so I ended up installed 2 Runner scale sets, one for kubernetes and one for DinD.

```yaml
githubConfigUrl: "https://github.com/your-organization"
githubConfigSecret: pre-defined-secret
maxRunners: 5
minRunners: 2
runnerScaleSetName: "set-linux-arm64"

# for DinD mode, uncomment the containerMode section below
# containerMode:
#   type: "dind"

# for kubernetes mode, uncomment the containerMode and template section below
# containerMode:
#   type: "kubernetes"
#   kubernetesModeWorkVolumeClaim:
#     accessModes: ["ReadWriteOnce"]
#     storageClassName: "openebs-hostpath"
#     resources:
#       requests:
#         storage: 5Gi
#
# template:
#   spec:
#     initContainers:
#     - name: kube-init
#       image: ghcr.io/actions/actions-runner:latest
#       command: ["sudo", "chown", "-R", "1001:1001", "/home/runner/_work"]
#       volumeMounts:
#       - name: work
#         mountPath: /home/runner/_work
#     containers:
#     - name: runner
#       image: ghcr.io/actions/actions-runner:latest
#       command: ["/home/runner/run.sh"]
#       env:
#         - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
#           value: "false"
```

### Add Personal Access Token to k8s namespace (mutual exclusive to next part)

(This part is only needed if you choose not to create a GitHub App but rather **provide a PAT to runner scale set**)

Securely add your Personal Access Token (PAT) for GitHub:

For how to generate PAT, please read the [documentation](https://github.com/actions/actions-runner-controller/blob/master/docs/authenticating-to-the-github-api.md#deploying-using-pat-authentication).


```bash
kubectl create namespace arc-runners
kubectl -n arc-runners create secret generic pre-defined-secret \
   --from-literal=github_token='__PAT_YOU_GENERATED_ON_GITHUB__'
```

### Add GitHub App Credential to k8s namespace (mutual exclusive to previous part)

(This part is only needed if you choose not to provide a PAT but rather **create a GitHub App for runner scale set**)

```bash
kubectl create namespace arc-runners
kubectl create secret generic pre-defined-secret \
   --namespace=arc-runners \
   --from-literal=github_app_id=123456 \
   --from-literal=github_app_installation_id=654321 \
   --from-literal=github_app_private_key='-----BEGIN RSA PRIVATE KEY-----********'
```

### Install the runner set using the prepared Helm values

```bash
INSTALLATION_NAME="arc-runner-set"
NAMESPACE="arc-runners"
helm install "${INSTALLATION_NAME}" \
    --namespace "${NAMESPACE}" \
    -f runner-scale-set-values.yaml \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## Verifying the Installation

Check the status of your Kubernetes pods and ensure everything is running smoothly:

```bash
➜  ~ kubectl get pods -A | grep '^arc'
arc-runners-dind   set-linux-arm64-dind-ukewea-runner-8gf8f        2/2     Running   0          3m17s
arc-runners-dind   set-linux-arm64-dind-ukewea-runner-vw87t        2/2     Running   0          3m17s
arc-runners        set-linux-arm64-ukewea-runner-5kl8p             1/1     Running   0          3m53s
arc-runners        set-linux-arm64-ukewea-runner-xx9k9             1/1     Running   0          3m54s
arc-systems        arc-gha-rs-controller-88454c84d-crdft          1/1     Running   0          21m
arc-systems        set-linux-arm64-754b578d-listener              1/1     Running   0          3m58s
arc-systems        set-linux-arm64-dind-6f85f55d-listener         1/1     Running   0          3m21s
```

And we are done!

![](2024-04-04-09-01-48.png)

---

## Proxy setting

If you need proxy setup, read following articles to do it properly:
- k0s: [Environment variables](https://docs.k0sproject.io/stable/environment-variables/?h=proxy)
- ARC: [values.yaml](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set/values.yaml) (jump to `proxy` part of the file)
