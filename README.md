Building a Kubernetes cluster with Vagrant
===

This Vagrantfile allows building a Kubernetes cluster with a configurable number of workers. The cluster is configured using `kubeadm`.

If you are building behind proxies and require proxy configurations inside the VMs, install the **vagrant-proxyconf** plugin with `vagrant plugin install vagrant-proxyconf`.

The following environment variables can be configured to control the disposition of the cluster:

Proxy variables
---
* `http_proxy`: configures `http_proxy` inside cluster VMs
* `https_proxy`: configures `https_proxy` inside cluster VMs
* `no_proxy`: configures `no_proxy` inside cluster VMs. If proxies are being configured, the master IP will be appended to `no_proxy` so that `kubeadm` functions as expected.
* `cacerts_dir`: if your SSL proxies perform MITM on SSL connections using custom root CA certificates, you may provide a path to a directory on the build host that contains additional CA certificates to add to the VMs. All certificates should end in `.crt` and each file should only contain one certificate.

Cluster sizing variables
---
* `worker_count`: configures the number of worker VMs that will be built (default **3**).
* `cpu_count`: configures the number of CPUs each VM will have (default **1**).
* `memory_mb`: configures the amount of memory in MB each VM will have (default **1024**).
