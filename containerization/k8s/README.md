# CNDP Pod

The provided configuration creates a CNDP pod with two containers. One that runs
the CNDP cndpfwd example and another which runs the prometheus go agent to
collect metrics from the CNDP cndpfwd container. The CNDP containers are
interconnected via a unix domain socket.

This guide will walk you through the setup of the CNDP pods and the AF_XDP
Device Plugin (DP).

## 1. Setup a Kubernetes Cluster

This section will walk you through how to setup a single node cluster where you
can launch a CNDP container. The example uses `kubeadm` to bootstrap the
cluster.

Start by setting the hostname for your platform and ensuring that there's a
corresponding entry in /etc/hosts

```bash
hostnamectl set-hostname <name>
```

```bash
cat >/etc/hosts <<EOL
127.0.0.1 localhost.localdomain localhost
HOST_IP YOURHOSTNAME.com YOURHOSTNAMEXXXXXXX
```

It's really important to make sure that your no proxy is configured as follows,
especially every time you log in and try to run any kubectl commands

```bash
no_proxy=HOST_IP,YOURHOSTNAME.com,localhost,127.0.0.1,10.96.0.0/12
```

### Prepare system to install containerd

#### Ubuntu 21.04

<!-- markdownlint-disable MD014 MD024 -->

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

#### Fedora 38

```bash
modprobe br_netfilter

echo 1 >/proc/sys/net/ipv4/ip_forward
```

### Install containerd

#### Ubuntu 21.04

```bash
sudo apt-get update

sudo apt-get install -y software-properties-common apt-transport-https ca-certificates curl gnupg
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update
sudo apt-get install apparmor-utils containerd.io
```

#### Fedora 38

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
dnf install containerd
```

### Configure containerd

#### Ubuntu 21.04

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

#### Fedora 38

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

On `Fedora 38` you also need to modify the `/etc/containerd/config.toml` file to
change the value for `SystemCgroup` from `false` to `true`:

```bash
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        BinaryName = ""
        CriuImagePath = ""
        CriuPath = ""
        CriuWorkPath = ""
        IoGid = 0
        IoUid = 0
        NoNewKeyring = false
        NoPivotRoot = false
        Root = ""
        ShimCgroup = ""
        SystemdCgroup = false =====>>> change to true
```

Reload the containerd daemon

```bash
sudo systemctl daemon-reload
sudo systemctl restart containerd
```

### Setup proxies for containerd

```bash
sudo mkdir -p /etc/systemd/system/containerd.service.d

cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/proxy.conf
[Service]

Environment="HTTP_PROXY=http://proxy.example.com:80/"

Environment="HTTPS_PROXY=http://proxy.example.com:80/"

Environment="NO_PROXY=localhost,127.0.0.1,10.96.0.0/16"
EOF
```

### Set Max Locked Memory Limit

#### Ubuntu 21.04

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/limits.conf
[Service]
LimitMEMLOCK=infinity
EOF
```

This step is needed so that the CNDP application launched in the unprivileged
container does not error out with ENOBUFS on socket creation.

### Restart the containerd daemon

```bash
sudo systemctl daemon-reload

sudo systemctl restart containerd
```

Verify that the proxy settings have taken effect

```bash
sudo systemctl show --property=Environment containerd
```

### Install Kubeadm

#### Ubuntu 21.04

Add the kubeadm repo:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

Install kubeadm:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

### Create the cluster

Ensure swap is disabled:

```bash
sudo swapoff -a
```

> **NOTE**: This step is required on both Ubuntu and Fedora On `Fedora 38` you
> also need to disable zram:
>
> ```bash
> sudo touch /etc/systemd/zram-generator.conf
> ```

And setup hugepages (used later by CNDP pod):

```bash
cat <<EOF | sudo tee -a /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
256
EOF
```

Launch a cluster:

```bash
sudo kubeadm init --v 99 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all
```

If the cluster launches successfully you should see a message that looks like:

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Copy the config file to $HOME/.kube

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install flannel and all the default networking plugins

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.15.1/Documentation/kube-flannel.yml
git clone https://github.com/containernetworking/plugins.git
cd plugins
$./build_linux.sh
ls bin/
bandwidth bridge dhcp dummy firewall host-device host-local ipvlan loopback macvlan portmap ptp sbr static tap tuning vlan vrf
cp bin/* /opt/cni/bin/
```

> **NOTE**: Compiling the network plugins requires golang to be installed.

Un-taint the controller node so you can schedule pods there and add a label to
it: Remember to change HOSTNAME to your actual hostname in the commands below:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl label node HOSTNAME cndp="true"
```

### Setup Multus

Setup Multus:

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/v3.8/images/multus-daemonset.yml
```

## 2. Build the CNDP container image

In the top level CNDP directory run:

```bash
make oci-image
```

Push this image to a registry.

### 3. Build and deploy AF_XDP plugin and CNI for K8s

The source code is available
[here](https://github.com/intel/afxdp-plugins-for-kubernetes).

For detailed install instructions please refer to README.md in the device plugin
repo. This section will provide a quick start for deploying the device plugin
and CNI.

Please ensure you have the runtime dependencies installed.

1. Clone the repo

    ```bash
    git clone https://github.com/intel/afxdp-plugins-for-kubernetes
    ```

1. Move to the top level directory of the repo

    ```bash
    cd afxdp-plugins-for-kubernetes
    ```

#### AF_XDP DP UDS based deployment

Edit the `deployments/daemonset.yml` file

```bash
vim deployments/daemonset.yml
```

Modify the ConfigMap to include the relevant drivers, also modify the af_xdp dp
image to reference the registry used to store the image built in the previous
step.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: afxdp-dp-config
  namespace: kube-system
data:
  config.json: |
    {
       "logLevel":"debug",
       "logFile":"afxdp-dp.log",
       "pools":[
          {
             "name":"myPool",
             "mode":"primary",
             "drivers":[
                {
                   "name":"i40e" ###### Modify the drivers you wish to add to the pool
                },
                {
                   "name":"ice" ###### Modify the drivers you wish to add to the pool
                }
             ]
          }
       ]
    }
```

#### AF_XDP DP xskmap pinning based deployment

Edit the `deployments/daemonset-pinning.yml` file

```bash
vim deployments/daemonset-pinning.yml
```

Modify the ConfigMap to include the relevant drivers, also modify the af_xdp dp
image to reference the registry used to store the image built in the previous
step.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: afxdp-dp-config
  namespace: kube-system
data:
  config.json: |
    {
       "logLevel":"debug",
       "logFile":"afxdp-dp.log",
       "pools":[
          {
             "name":"myPool",
             "mode":"primary",
             "drivers":[
                {
                   "name":"i40e" ###### Modify the drivers you wish to add to the pool
                },
                {
                   "name":"ice" ###### Modify the drivers you wish to add to the pool
                }
             ]
          }
       ]
    }
```

#### Build the AF_XDP DP Daemonset and CNI

Build the AF_XDP DP container image:

```bash
sudo make image
```

### 4. Import docker images

Before importing docker images, the container images are built using docker. For
more information on building the CNDP image using docker, please follow the
instructions at containerization/docker/README.md.

If using containerd as the container run-time interface with K8s, the images
built using docker need to be imported so that the K8s run-time can use it:

Import the CNDP docker image

```bash
docker save cndp -o cndp.tar

ctr -n=k8s.io images import cndp.tar
```

Import the AF_XDP device plugin docker image.

```bash
docker save afxdp-device-plugin -o afxdp-device-plugin.tar

ctr -n=k8s.io images import afxdp-device-plugin.tar
```

#### Verify that the image is now available to the container run-time

More information on how to install crictl can be found
[here](https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/).

```bash
sudo crictl images
```

### 5. Deploy the AF_XDP DP Daemonset and CNI

To deploy the UDS based configuration run:

```bash
kubectl create -f ./deployments/daemonset.yml
```

To deploy the map pinning based configuration run:

```bash
kubectl create -f ./deployments/daemonset-pinning.yml
```

### 6. Modify the Network attachment definition for AF_XDP interface

> **_NOTE:_** make sure any interfaces you wish to add to the Pod are in an UP
> state and do not have IP addresses configured.

Ethtool filters are programmed on the interface by the CNI through the NAD.
Currently, the ethtool filter will be programmed on queue 4 and on.

The "queues" field in the network attachment definition decides how many queues
are used for the ethtool filter setup.

If `"queues":"1"` is set in the network attachment definition, the ethtool
filter programmed by the CNI, will be of the form

```bash
"ethtoolCmds" : ["-X -device- equal 1 start 4", # CNI ethtool filters (optional)
"--config-ntuple -device- flow-type udp4 dst-ip -ip- action"
],
```

In order to create the network attachment definition:

```bash
kubectl create -f containerization/k8s/networks/uds-nad.yaml
```

> Note: To deploy the map pinning based configuration run:

```bash
kubectl create -f containerization/k8s/networks/map-pinning-nad.yaml
```

Check the definition was added:

```bash
kubectl get network-attachment-definitions
```

### Create CNDP pod

First make optional changes to the containerization/k8s/cndp-pods/cndp-0-0.yaml
file to define LIST_OF_QIDS or CNDP_COPY_MODE environment variables. These
variables are read by `tools/jsonc_gen.sh` when the container starts to generate
the configuration for the CNDP application.

> **_NOTE:_** the `tools/jsonc_gen.sh` supports BPF map pinning configuration by
> running it with the `-p` flag. If you wish to use it in a kind cluster then
> run the script with a `-k` flag in the pod YAML.

An example to force copy-mode for all AF_XDP sockets:

```yaml
  containers:
    - name: cndp-0
    ...
      env:
      - name: CNDP_COPY_MODE
        value: "true"
```

Use the [cndp-0-0.yaml](../cndp-pods/cndp-0-0.yaml) to create the pod.

```bash
kubectl create -f containerization/k8s/cndp-pods/cndp-0-0.yaml
```

### Check the CNDP pod

```bash
kubectl describe pods
```

### Connecting to the cndp containers container

```bash
kubectl exec -i -t cndp-0-0 --container control-0 -- /bin/bash
kubectl exec -i -t cndp-0-0 --container cndp-0 -- /bin/bash
```

### Checking the container logs

```bash
kubectl logs cndp-0-0 control-0
kubectl logs cndp-0-0 cndp-0
```

### Port forwarding to export metrics to the local host

```bash
kubectl port-forward cndp-0-0 2112:2112 &
```

### Prometheus installation

To install Prometheus locally on the host, please follow the sets
[here](https://computingforgeeks.com/install-prometheus-server-on-debian-ubuntu-linux/)

> **_NOTE:_** There's no need to enable Prometheus as a service that runs all
> the time.

Modify the /etc/prometheus/prometheus.yml file to pull stats from the CNDP pod

```yaml
# my global config
global:
  scrape_interval: 1s     # UPDATE THIS INTERVAL AS PREFERRED. Default is every 1 minute.
  evaluation_interval: 1s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: prometheus

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: [localhost:9090]
      - targets: [localhost:2112] # THIS IS THE CNDP PODs exposed metrics port

```

After restarting the prometheus service to pick up these changes you should be
able to see the CNDP pod metrics in the prometheus frontend through a web
browser, or through curl:

```bash
curl http://localhost:2112/metrics
```

### Logging

To install a reference logging stack based on Fluent Bit, Elasticsearch, and
Kibana follow the instructions in [logging-README.md](logging-README.md).

### Deleting the CNDP pod

```bash
kubectl delete pods cndp-0-0
```

## Resetting the Cluster

```bash
kubeadm reset
unset KUBECONFIG
rm -rf $HOME/.kube/
systemctl daemon-reload
systemctl restart kubelet
systemctl restart containerd
ls /etc/cni/net.d
rm -rf /etc/cni/net.d/*
swapoff -av
free -h
kubeadm init --pod-network-cidr=10.244.0.0/16
```

### References

- [Container-runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Kubeadm installation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubeadm cluster creation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Multus Quickstart](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md)

<!-- markdownlint-enable MD014 -->

<!-- markdownlint-enable MD024 -->
