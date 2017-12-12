# Slate-K8s alpha deployment

## Prerequisites

* The user provisioning the cluster must have passwordless sudo access to the targets
* Nodes (both Master and Worker) must have swap disabled
* Any node that is intended to run keepalived must have the sysctl setting `net.ipv4.ip_nonlocal_bind=1` configured.


## Configuring Manager and Kubespray
Additional notes and instructions can be found within Kubespray's own docs:
https://github.com/kubernetes-incubator/kubespray/blob/master/docs/ansible.md

Install pip as appropriate to the host OS. e.g. For CentOS

```
yum install epel-release
yum update
yum install python-pip
```

Install Kubespray dependencies

```
yum install git
pip install ansible netaddr
```

Clone the fork of kubespray and checkout the branch with `keepalived-cloud-provider` support

```
git clone https://github.com/kubernetes-incubator/kubespray.git
cd kubespray
```

Prepare the Ansible host Inventory in `inventory/inventory.ini` using the example file `inventory/inventory.example` as a reference. When complete, it should look similar to this:

```
[kube-master]
master-01
master-02
master-03

[etcd]
master-01
master-02
master-03

[kube-node]
node-01
node-02
node-03

[k8s-cluster:children]
kube-node
kube-master
```

Update the Kubernetes Cluster configs located in `inventory/group_vars/`.

##### all.yml

|      Variable Name     |   Setting   |
|:----------------------:|:-----------:|
|     `bootstrap_os`     | `<host os>` |
| `kubelet_load_modules` |    `true`   |
|    `cloud_provider`    |  `external` |


##### k8-cluster.yml
**Note:** The cidr ranges should be configured appropriately to your own network needs

|       Variable Name      |       Setting      |
|:------------------------:|:------------------:|
|  `enable_network_policy` |       `true`       |
| `kube_service_addresses` | `192.168.128.0/18` |
|    `kube_pods_subnet`    | `192.168.192.0/18` |
|  `local_volumes_enabled` |       `true`       |
|    `kubectl_localhost`   |       `true`       |


Kubespray should now be configured.


## Provisioning the Cluster

from within the kubespray directory, execute the following:

`ansible-playbook -i inventory/inventory.ini cluster.yml -b -v`

It will likely take 15 minutes or longer depending on the size of the cluster.


## Configuring kubectl

With the cluster provisioned, ssh to one of the master nodes and configure `kubectl`.

```
kubectl config set-cluster <cluster_name> --server=http://localhost:8080
kubectl config set-context <context_name> --cluster=<cluster_name> --namespace=kube-system
kubectl config use-context <context_name>
```

The default deployment of kubernetes with kubespray configures the apiserver with unencrypted/unauthenticated access from the localhost of the masters. External connectivity would require provisioning of additional service accounts and certificates with associated certificates.

Verify your connectivity to kubernetes with the following: `kubectl get pods`

It should return a list similar to the below
```
slate@master-01:~$ kubectl get pods
NAME                                         READY     STATUS    RESTARTS   AGE
calico-node-4cjxt                            1/1       Running   0          5m
calico-node-8kxqn                            1/1       Running   0          5m
calico-node-kfq6h                            1/1       Running   0          5m
calico-node-n4nhv                            1/1       Running   0          5m
calico-node-qk47c                            1/1       Running   0          5m
calico-node-vwj2b                            1/1       Running   0          5m
calico-policy-controller-wlg7z               1/1       Running   0          5m
kube-apiserver-master-01                     1/1       Running   0          5m
kube-apiserver-master-02                     1/1       Running   0          5m
kube-apiserver-master-03                     1/1       Running   0          5m
kube-controller-manager-master-01            1/1       Running   0          5m
kube-controller-manager-master-02            1/1       Running   0          5m
kube-controller-manager-master-03            1/1       Running   0          5m
kube-dns-cf9d8c47-h92dm                      3/3       Running   0          5m
kube-dns-cf9d8c47-lf5sc                      3/3       Running   0          5m
kube-proxy-master-01                         1/1       Running   0          5m
kube-proxy-master-02                         1/1       Running   0          5m
kube-proxy-master-03                         1/1       Running   0          5m
kube-proxy-node-01                           1/1       Running   0          5m
kube-proxy-node-02                           1/1       Running   0          5m
kube-proxy-node-03                           1/1       Running   0          5m
kube-scheduler-master-01                     1/1       Running   0          5m
kube-scheduler-master-02                     1/1       Running   0          5m
kube-scheduler-master-03                     1/1       Running   0          5m
kubedns-autoscaler-86c47697df-4bqcf          1/1       Running   0          5m
kubernetes-dashboard-7fd45476f8-7585d        1/1       Running   0          5m
local-volume-provisioner-57nmv               1/1       Running   0          5m
local-volume-provisioner-78mb7               1/1       Running   0          5m
local-volume-provisioner-drdbx               1/1       Running   0          5m
nginx-proxy-node-01                          1/1       Running   0          5m
nginx-proxy-node-02                          1/1       Running   0          5m
nginx-proxy-node-03                          1/1       Running   0          5m
```


## Deploying keepalived-cloud-provider

With the cluster provisioned, keepalived-cloud-provider can now be deployed. This serves as a method of dynamically allocating external IPs to desired exposed services within the cluster.

1. Begin by copying both the `kube-keepalived-vip` and `keepalived-cloud-provider` directories to the master where `kubectl` was configured

2. Update the `KEEPALIVED_SERVICE_CIDR` environment variable within the `keepalived-cloud-provider/003-dep.yaml` file with an appropriate range for your network.
```
        - name: KEEPALIVED_SERVICE_CIDR
          value: "10.255.33.200/30" # pick a CIDR that is explicitly reserved for keepalived
```
3. Once complete, deploy the kube-keepalived-vip DaemonSet by creating everything within the keepalived directory. `kubectl create -f kube-keepalived-vip/*` **Note:** As no service is consuming a shared IP, the DaemonSet pods will quickly go into `CrashLoopBackoff`, this is expected and should be ignored at this stage.

4. Now deploy the keepalived-cloud-provider services by creating everything within the keepalived-cloud-provider directory. `kubectl create -f keepalived-cloud-provider/*`.

5. The keepalived-vip pods will continue to crash until a service is configured that consumes an IP from the pool. This can be done by updating a service such as `kubernetes-dashboard` or deploying some other small service with `spec.type: LoadBalanacer` configured.
