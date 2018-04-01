# kubernetes-via-kubeadm
How to deploy a Kubernetes master and node on a single Fedora host via kubeadm.    
  
By default it is not possible (and not recommanded) to run applications on `master` node(s). In case you have only one host you could deploy a Kubernetes cluster on one host via `kubeadm`. Basicly this means that all the components needs to be installed on the same host and that the `master` should be configured `schedulable`.  
Down here you find the steps to configure a Fedora hosts as a single cluster hosts.  
  
## Prerequisites 
One host with:   
* Fedora 27 (or higher) installed 
* Docker 1.13 (or higher) installed  
  
## Deploy Kubernetes cluster via kubeadm
  
### Install the kubeadm, kubectl and kubelet packages  
```
[root@nuc ~]# dnf install kubernetes-kubeadm kubernetes kubernetes-client
[root@nuc ~]# systemctl enable kubelet && systemctl start kubelet
```
  
### Disable selinux 
`SELINUX` is not yet support with `kubeadm`, so it must be turned off!  
```
[root@nuc ~]# setenforce 0
[root@nuc ~]# vi /etc/sysconfig/selinux  (SELINUX=disabled)
```
  
### Open firewall ports for Kubernetes. 
Since we have only one host that we use as `master` and `node` we apply both commands to our host.  
On the master(s):  
```
[root@nuc ~]# for PORT in 6443 2379 2380 10250 10251 10252 10255; do \
	firewall-cmd --permanent --add-port=${PORT}/tcp --permanent \
done
[root@nuc ~]# firewall-cmd --reload
[root@nuc ~]# firewall-cmd --list-ports
```  
On the node(s): 
```
[root@nuc ~]# for PORT in 53 10250 10255 30000-32767; do \
	firewall-cmd --permanent --add-port=${PORT}/tcp --permanent \
done
[root@nuc ~]# firewall-cmd --reload
[root@nuc ~]# firewall-cmd --list-ports
``` 
   
### Disable swapping: 
```
[root@nuc ~]# swapoff -a
[root@nuc ~]# sed -i 's/\(^.*swap.*$\)/\#\1/g' /etc/fstab
```
Swapping is not supported by `kubelet`.
  
### net.bridge.bridge-nf-call-iptables needs to be set to 1
```
[root@nuc ~]# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
```
  
### Bootstrap the cluster  
I choose to use `flannel` as network layer. This means kubeadm needs to know a network CIDR: `--pod-network-cidr=10.244.0.0/16`
```
[root@nuc ~]# kubeadm init --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.9.6
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
	[WARNING FileExisting-crictl]: crictl not found in system path
[preflight] Starting the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [nuc.bachstraat20 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.11.100]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 29.003147 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node nuc.bachstraat20 as master by adding a label and a taint
[markmaster] Master nuc.bachstraat20 tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: 00b963.5761fb79e8d66f42
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 00b963.5761fb79e8d66f42 192.168.11.100:6443 --discovery-token-ca-cert-hash sha256:8cbed3f0d2e71762d3c52f03bef702700c2e889827e0e13f403fc463ff58c3ef
```
Read on, you're not yet there... 

### Make sure kubectl works  
For regular users:  
```
[some-user@nuc ~]# mkdir -p $HOME/.kube
[some-user@nuc ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
``` 
Alternatively, if you are the root user, you could run this:  
```
[root@nuc ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
```  
Now `kubectl` works, but the node is `NotReady`:  
```
[root@nuc ~]# kubectl get nodes
NAME               STATUS     ROLES     AGE       VERSION
nuc.bachstraat20   NotReady   master    1m        v1.9.1
```
  
### Deploy a pod network  
In order to enable connectivity between pods, we deploy `flannel`:  
```
[root@nuc ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
clusterrole "flannel" created
clusterrolebinding "flannel" created
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
```
Now the node reports `ready`:  
```
[root@nuc ~]# kubectl get nodes
NAME               STATUS    ROLES     AGE       VERSION
nuc.bachstraat20   Ready     master    8m        v1.9.1
```
 
### Check the node
```
[root@nuc ~]# kubectl describe nodes
NAME                                       READY     STATUS    RESTARTS   AGE
etcd-nuc.bachstraat20                      1/1       Running   0          7m
kube-apiserver-nuc.bachstraat20            1/1       Running   0          8m
kube-controller-manager-nuc.bachstraat20   1/1       Running   0          7m
kube-dns-6f4fd4bdf-z6z9k                   3/3       Running   0          8m
kube-flannel-ds-d9j74                      1/1       Running   0          2m
kube-proxy-d945n                           1/1       Running   0          8m
kube-scheduler-nuc.bachstraat20            1/1       Running   0          7m
[root@nuc kubelet.service.d]# kubectl describe nodes
Name:               nuc.bachstraat20
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=nuc.bachstraat20
                    node-role.kubernetes.io/master=
Annotations:        flannel.alpha.coreos.com/backend-data={"VtepMAC":"02:84:03:2e:99:1a"}
                    flannel.alpha.coreos.com/backend-type=vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager=true
                    flannel.alpha.coreos.com/public-ip=192.168.11.100
                    node.alpha.kubernetes.io/ttl=0
                    volumes.kubernetes.io/controller-managed-attach-detach=true
Taints:             node-role.kubernetes.io/master:NoSchedule
CreationTimestamp:  Sun, 01 Apr 2018 15:48:08 +0200
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  OutOfDisk        False   Sun, 01 Apr 2018 15:58:29 +0200   Sun, 01 Apr 2018 15:48:00 +0200   KubeletHasSufficientDisk     kubelet has sufficient disk space available
  MemoryPressure   False   Sun, 01 Apr 2018 15:58:29 +0200   Sun, 01 Apr 2018 15:48:00 +0200   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 01 Apr 2018 15:58:29 +0200   Sun, 01 Apr 2018 15:48:00 +0200   KubeletHasNoDiskPressure     kubelet has no disk pressure
  Ready            True    Sun, 01 Apr 2018 15:58:29 +0200   Sun, 01 Apr 2018 15:54:59 +0200   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.11.100
  Hostname:    nuc.bachstraat20
Capacity:
 cpu:     4
 memory:  16351108Ki
 pods:    110
Allocatable:
 cpu:     4
 memory:  16248708Ki
 pods:    110
System Info:
 Machine ID:                 ccb68e77d17e47e6b23fca3120ef55f4
 System UUID:                E820CB80-34D4-11E1-B35F-C03FD562BB77
 Boot ID:                    f74102a4-1a1f-476b-a9d5-16c36ea8476f
 Kernel Version:             4.15.13-300.fc27.x86_64
 OS Image:                   Fedora 27 (Workstation Edition)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://1.13.1
 Kubelet Version:            v1.9.1
 Kube-Proxy Version:         v1.9.1
PodCIDR:                     10.244.0.0/24
ExternalID:                  nuc.bachstraat20
Non-terminated Pods:         (7 in total)
  Namespace                  Name                                        CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                                        ------------  ----------  ---------------  -------------
  kube-system                etcd-nuc.bachstraat20                       0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-apiserver-nuc.bachstraat20             250m (6%)     0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-controller-manager-nuc.bachstraat20    200m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-dns-6f4fd4bdf-z6z9k                    260m (6%)     0 (0%)      110Mi (0%)       170Mi (1%)
  kube-system                kube-flannel-ds-d9j74                       0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-proxy-d945n                            0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-scheduler-nuc.bachstraat20             100m (2%)     0 (0%)      0 (0%)           0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  810m (20%)    0 (0%)      110Mi (0%)       170Mi (1%)
Events:
  Type    Reason                   Age                From                          Message
  ----    ------                   ----               ----                          -------
  Normal  NodeAllocatableEnforced  10m                kubelet, nuc.bachstraat20     Updated Node Allocatable limit across pods
  Normal  NodeHasSufficientDisk    10m (x8 over 10m)  kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeHasSufficientDisk
  Normal  NodeHasSufficientMemory  10m (x8 over 10m)  kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    10m (x7 over 10m)  kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeHasNoDiskPressure
  Normal  Starting                 10m                kube-proxy, nuc.bachstraat20  Starting kube-proxy.
  Normal  NodeReady                3m                 kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeReady
```
The node looks `okay`, but since it is a master, it reports `Taints: node-role.kubernetes.io/master:NoSchedule`. So it can't schedule any pods.  
   
### Make master node being able to schedule pods
By default it is not possible (and not very wise) to schedule pods on the master hosts, but if you just have only one host, you can make it `schedulable`.  
Remove `taints`, `effect:NoSchedule` and `key: node-role.kubernetes.io/master` from master node config.
```
[root@nuc ~]# kubectl edit node nuc.bachstraat20
spec:
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
```
Now the host is ready to schedule pods:  
```
[root@nuc ~]# kubectl get nodes -o wide
NAME               STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE                          KERNEL-VERSION            CONTAINER-RUNTIME
nuc.bachstraat20   Ready     master    17m       v1.9.1    <none>        Fedora 27 (Workstation Edition)   4.15.10-300.fc27.x86_64   docker://1.13.1
```
The cluster is ready to schedule pods. Continue with deploying the Kubernetes Dashboard...  
  
## Install Kubernetes Dashboard  
The `dashboard` itself is a single deployment, but in order to view pod metrics `heapster`, `influxdb` and `grafana` needs to be deployed.  
  
### InfluxDB
```
[root@nuc ~]# kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml                     
deployment "monitoring-influxdb" created 
service "monitoring-influxdb" created  

[root@nuc ~]# kubectl -n kube-system logs monitoring-influxdb-66946c9f58-rw695 
 8888888           .d888 888                   8888888b.  888888b.
   888            d88P"  888                   888  "Y88b 888  "88b
   888            888    888                   888    888 888  .88P
   888   88888b.  888888 888 888  888 888  888 888    888 8888888K.
   888   888 "88b 888    888 888  888  Y8bd8P' 888    888 888  "Y88b
   888   888  888 888    888 888  888   X88K   888    888 888    888
   888   888  888 888    888 Y88b 888 .d8""8b. 888  .d88P 888   d88P
 8888888 888  888 888    888  "Y88888 888  888 8888888P"  8888888P"
[I] 2018-04-01T14:00:28Z InfluxDB starting, version unknown, branch unknown, commit unknown
[I] 2018-04-01T14:00:28Z Go version go1.8.3, GOMAXPROCS set to 4
[I] 2018-04-01T14:00:28Z Using configuration at: /etc/config.toml
[I] 2018-04-01T14:00:38Z Using data dir: /data/data service=store
[I] 2018-04-01T14:00:38Z opened service service=subscriber
[I] 2018-04-01T14:00:38Z Starting monitor system service=monitor
[I] 2018-04-01T14:00:38Z 'build' registered for diagnostics monitoring service=monitor
[I] 2018-04-01T14:00:38Z 'runtime' registered for diagnostics monitoring service=monitor
[I] 2018-04-01T14:00:38Z 'network' registered for diagnostics monitoring service=monitor
[I] 2018-04-01T14:00:38Z 'system' registered for diagnostics monitoring service=monitor
[I] 2018-04-01T14:00:38Z Starting precreation service with check interval of 10m0s, advance period of 30m0s service=shard-precreation
[I] 2018-04-01T14:00:38Z Starting snapshot service service=snapshot
[I] 2018-04-01T14:00:38Z Starting continuous query service service=continuous_querier
[I] 2018-04-01T14:00:38Z Starting HTTP service service=httpd
[I] 2018-04-01T14:00:38Z Authentication enabled:false service=httpd
[I] 2018-04-01T14:00:38Z Listening on HTTP:[::]:8086 service=httpd
[I] 2018-04-01T14:00:38Z Starting retention policy enforcement service with check interval of 30m0s service=retention
[I] 2018-04-01T14:00:38Z Listening for signals
[I] 2018-04-01T14:00:38Z Storing statistics in database '_internal' retention policy 'monitor', at interval 10s service=monitor
```

### Grafana
```
[root@nuc ~]# kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
deployment "monitoring-grafana" created
service "monitoring-grafana" created

[root@nuc ~]# kubectl -n kube-system logs monitoring-grafana-65757b9656-c2b6r 
Starting a utility program that will configure Grafana
Starting Grafana in foreground mode
t=2018-04-01T14:02:35+0000 lvl=info msg="Starting Grafana" logger=main version=v4.4.3 commit=unknown-dev compiled=2018-04-01T14:02:35+0000
t=2018-04-01T14:02:35+0000 lvl=info msg="Config loaded from" logger=settings file=/usr/share/grafana/conf/defaults.ini
t=2018-04-01T14:02:35+0000 lvl=info msg="Config loaded from" logger=settings file=/etc/grafana/grafana.ini
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from command line" logger=settings arg="default.paths.data=/var/lib/grafana"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from command line" logger=settings arg="default.paths.logs=/var/log/grafana"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from command line" logger=settings arg="default.log.mode=console"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from Environment variable" logger=settings var="GF_SERVER_PROTOCOL=http"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from Environment variable" logger=settings var="GF_SERVER_HTTP_PORT=3000"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from Environment variable" logger=settings var="GF_SERVER_ROOT_URL=/"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from Environment variable" logger=settings var="GF_AUTH_ANONYMOUS_ENABLED=true"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from Environment variable" logger=settings var="GF_AUTH_ANONYMOUS_ORG_ROLE=Admin"
t=2018-04-01T14:02:35+0000 lvl=info msg="Config overridden from Environment variable" logger=settings var="GF_AUTH_BASIC_ENABLED=false"
t=2018-04-01T14:02:35+0000 lvl=info msg="Path Home" logger=settings path=/usr/share/grafana
t=2018-04-01T14:02:35+0000 lvl=info msg="Path Data" logger=settings path=/var/lib/grafana
t=2018-04-01T14:02:35+0000 lvl=info msg="Path Logs" logger=settings path=/var/log/grafana
t=2018-04-01T14:02:35+0000 lvl=info msg="Path Plugins" logger=settings path=/usr/share/grafana/data/plugins
t=2018-04-01T14:02:35+0000 lvl=info msg="Initializing DB" logger=sqlstore dbtype=sqlite3
t=2018-04-01T14:02:35+0000 lvl=info msg="Starting DB migration" logger=migrator
t=2018-04-01T14:02:35+0000 lvl=info msg="Executing migration" logger=migrator id="create migration_log table"
t=2018-04-01T14:02:35+0000 lvl=info msg="Executing migration" logger=migrator id="create user table"
t=2018-04-01T14:02:35+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index user.login"
t=2018-04-01T14:02:35+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index user.email"
t=2018-04-01T14:02:35+0000 lvl=info msg="Executing migration" logger=migrator id="drop index UQE_user_login - v1"
t=2018-04-01T14:02:35+0000 lvl=info msg="Executing migration" logger=migrator id="drop index UQE_user_email - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Rename table user to user_v1 - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create user table v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_user_login - v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_user_email - v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="copy data_source v1 to v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table user_v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Add column help_flags1 to user table"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Update user table charset"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create temp user table v1-7"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_temp_user_email - v1-7"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_temp_user_org_id - v1-7"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_temp_user_code - v1-7"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_temp_user_status - v1-7"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Update temp_user table charset"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create star table"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index star.user_id_dashboard_id"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create org table v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_org_name - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create org_user table v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_org_user_org_id - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_org_user_org_id_user_id - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="copy data account to org"
t=2018-04-01T14:02:36+0000 lvl=info msg="Skipping migration condition not fulfilled" logger=migrator id="copy data account to org"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="copy data account_user to org_user"
t=2018-04-01T14:02:36+0000 lvl=info msg="Skipping migration condition not fulfilled" logger=migrator id="copy data account_user to org_user"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table account"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table account_user"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Update org table charset"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Update org_user table charset"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create dashboard table"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="add index dashboard.account_id"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index dashboard_account_id_slug"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create dashboard_tag table"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index dashboard_tag.dasboard_id_term"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="drop index UQE_dashboard_tag_dashboard_id_term - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Rename table dashboard to dashboard_v1 - v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create dashboard v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_dashboard_org_id - v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_dashboard_org_id_slug - v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="copy dashboard v1 to v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="drop table dashboard_v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="alter dashboard.data to mediumtext v1"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Add column updated_by in dashboard - v2"
t=2018-04-01T14:02:36+0000 lvl=info msg="Executing migration" logger=migrator id="Add column created_by in dashboard - v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add column gnetId in dashboard"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add index for gnetId in dashboard"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add column plugin_id in dashboard"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add index for plugin_id in dashboard"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add index for dashboard_id in dashboard_tag"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Update dashboard table charset"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Update dashboard_tag table charset"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create data_source table"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="add index data_source.account_id"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index data_source.account_id_name"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="drop index IDX_data_source_account_id - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="drop index UQE_data_source_account_id_name - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Rename table data_source to data_source_v1 - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create data_source table v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_data_source_org_id - v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_data_source_org_id_name - v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="copy data_source v1 to v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table data_source_v1 #2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add column with_credentials"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Add secure json data column"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Update data_source table charset"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create api_key table"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="add index api_key.account_id"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="add index api_key.key"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="add index api_key.account_id_name"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="drop index IDX_api_key_account_id - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="drop index UQE_api_key_key - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="drop index UQE_api_key_account_id_name - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Rename table api_key to api_key_v1 - v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create api_key table v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_api_key_org_id - v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_api_key_key - v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_api_key_org_id_name - v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="copy api_key v1 to v2"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table api_key_v1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="Update api_key table charset"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create dashboard_snapshot table v4"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="drop table dashboard_snapshot_v4 #1"
t=2018-04-01T14:02:37+0000 lvl=info msg="Executing migration" logger=migrator id="create dashboard_snapshot table v5 #2"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_dashboard_snapshot_key - v5"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_dashboard_snapshot_delete_key - v5"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create index IDX_dashboard_snapshot_user_id - v5"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="alter dashboard_snapshot to mediumtext v2"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update dashboard_snapshot table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create quota table v1"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_quota_org_id_user_id_target - v1"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update quota table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create plugin_setting table"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create index UQE_plugin_setting_org_id_plugin_id - v1"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Add column plugin_version to plugin_settings"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update plugin_setting table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create session table"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table playlist table"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old table playlist_item table"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create playlist table v2"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create playlist item table v2"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update playlist table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update playlist_item table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="drop preferences table v2"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="drop preferences table v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create preferences table v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update preferences table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create alert table v1"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index alert org_id & id "
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index alert state"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index alert dashboard_id"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create alert_notification table v1"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Add column is_default"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index alert_notification org_id & name"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update alert table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update alert_notification table charset"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Drop old annotation table v4"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="create annotation table v5"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index annotation 0 v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index annotation 1 v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index annotation 2 v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index annotation 3 v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="add index annotation 4 v3"
t=2018-04-01T14:02:38+0000 lvl=info msg="Executing migration" logger=migrator id="Update annotation table charset"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="Add column region_id to annotation table"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="create test_data table"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="create dashboard_version table v1"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="add index dashboard_version.dashboard_id"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="add unique index dashboard_version.dashboard_id and dashboard_version.version"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="Set dashboard version to 1 where 0"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="save existing dashboard data in dashboard_version table v1"
t=2018-04-01T14:02:39+0000 lvl=info msg="Executing migration" logger=migrator id="alter dashboard_version.data to mediumtext v1"
t=2018-04-01T14:02:39+0000 lvl=info msg="Created default admin user: admin"
t=2018-04-01T14:02:39+0000 lvl=info msg="Starting plugin search" logger=plugins
t=2018-04-01T14:02:39+0000 lvl=warn msg="Plugin dir does not exist" logger=plugins dir=/usr/share/grafana/data/plugins
t=2018-04-01T14:02:39+0000 lvl=info msg="Plugin dir created" logger=plugins dir=/usr/share/grafana/data/plugins
t=2018-04-01T14:02:39+0000 lvl=info msg="Initializing Alerting" logger=alerting.engine
t=2018-04-01T14:02:39+0000 lvl=info msg="Initializing CleanUpService" logger=cleanup
t=2018-04-01T14:02:39+0000 lvl=info msg="Initializing Stream Manager"
t=2018-04-01T14:02:39+0000 lvl=info msg="Initializing HTTP Server" logger=http.server address=0.0.0.0:3000 protocol=http subUrl= socket=
Connected to the Grafana dashboard.
The datasource for the Grafana dashboard is now set.
```
  
### Heapster
```
[root@nuc ~]# kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created

[root@nuc ~]# kubectl -n kube-system logs heapster-5c448886d-ssw4f 
I0401 19:22:48.659393       1 heapster.go:72] /heapster --source=kubernetes:https://kubernetes.default --sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
I0401 19:22:48.659440       1 heapster.go:73] Heapster version v1.4.2
I0401 19:22:48.659642       1 configs.go:61] Using Kubernetes client with master "https://kubernetes.default" and version v1
I0401 19:22:48.659664       1 configs.go:62] Using kubelet port 10255
I0401 19:22:48.672479       1 influxdb.go:278] created influxdb sink with options: host:monitoring-influxdb.kube-system.svc:8086 user:root db:k8s
I0401 19:22:48.672511       1 heapster.go:196] Starting with InfluxDB Sink
I0401 19:22:48.672515       1 heapster.go:196] Starting with Metric Sink
I0401 19:22:48.684241       1 heapster.go:106] Starting heapster on port 8082
I0401 19:23:05.112135       1 influxdb.go:241] Created database "k8s" on influxDB server at "monitoring-influxdb.kube-system.svc:8086"
```
### Heapster RBAC
```
[root@nuc ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
clusterrolebinding "heapster" created
```
### Kubernetes dashboard
```
[root@nuc ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created

[root@nuc ~]# kubectl -n kube-system logs kubernetes-dashboard-5bd6f767c7-xzcq4 
2018/04/01 14:07:26 Starting overwatch
2018/04/01 14:07:26 Using in-cluster config to connect to apiserver
2018/04/01 14:07:26 Using service account token for csrf signing
2018/04/01 14:07:26 No request provided. Skipping authorization
2018/04/01 14:07:26 Successful initial request to the apiserver, version: v1.9.6
2018/04/01 14:07:26 Generating JWE encryption key
2018/04/01 14:07:26 New synchronizer has been registered: kubernetes-dashboard-key-holder-kube-system. Starting
2018/04/01 14:07:26 Starting secret synchronizer for kubernetes-dashboard-key-holder in namespace kube-system
2018/04/01 14:07:26 Storing encryption key in a secret
2018/04/01 14:07:26 Creating in-cluster Heapster client
2018/04/01 14:07:26 Successful request to heapster
2018/04/01 14:07:26 Auto-generating certificates
2018/04/01 14:07:26 Successfully created certificates
2018/04/01 14:07:26 Serving securely on HTTPS port: 8443
```
To make the dashboard accessible via a `node-port` change the `type` in the `kubernetes-dashboard` service. Change `type: ClusterIP` to `type: NodePort` and save it: 
```
[root@nuc ~]#  kubectl -n kube-system edit service kubernetes-dashboard
service "kubernetes-dashboard" edited

[root@nuc ~]# kubectl get service -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
heapster               ClusterIP   10.101.132.149   <none>        80/TCP          8m
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   25m
kubernetes-dashboard   NodePort    10.100.227.126   <none>        443:30120/TCP   5m
monitoring-grafana     ClusterIP   10.102.178.122   <none>        80/TCP          10m
monitoring-influxdb    ClusterIP   10.103.66.221    <none>        8086/TCP        12m
```
The Dashboard has been exposed on port 30120 (HTTPS). It is accessible from a web browser at: https://nuc.bachstraat20:30120  
Continue below to create credentieels to login to the kubernetes-dashboard.
  
### Create an admin user  
To create a cluster-admin user, use these files: [admin-user.yaml](https://github.com/tedsluis/kubernetes-via-kubeadm/blob/master/admin-user.yaml) and [admin-user-clusterrolebinding.yaml](https://github.com/tedsluis/kubernetes-via-kubeadm/blob/master/admin-user-clusterrolebinding.yaml):  
```
[root@nuc ~]# kubectl create -f admin-user.yaml -n kube-system
serviceaccount "admin-user" created
[root@nuc ~]# kubectl create -f admin-user-clusterrolebinding.yaml
clusterrolebinding "admin-user" created
```
  
To get the token for this `admin-user`:  
```
[root@nuc kubernetes-via-kubeadm]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') | grep ^token: | sed 's/token:[ ]*//'
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW1oNzIyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwNWM0ZDZmZC0yZjYyLTExZTgtYTMxNi1jMDNmZDU2MmJiNzciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.butKxegADx3JQvKpn9Prf7RL_SoxaEyi_scYOvXurm4BAwEj8zfC9a7djqQ9mBtd5cQHlljvMb-3qFc6UPOzAwR8fc5khk-nAkH-5XeahpT8WsyxMcKxqLuyAg8gh4ZtMKvBPk9kOWDtyRBzAeGkisbLxr43ecKO71F5G8D7HR2UGSm-x4Pvhq0uqj8GyIcHw902Ti92BPuBRf-SyTl8uDCQJSDkS5Tru5w0p82borNuVXd1mmDwuI87ApQrqXTY9rbJ61m8iTr0kKJBqw5bHAUAhxwAVtVEKQNNKT6cxWp1FlhHbNkM9bhcj1qj8bN1QCMjPWlWKj7NkPbbBAJthQ
``` 
You can use the token to login to the kubernetes-dashboard.  
  
[![Kubernetes Dashboard](https://raw.githubusercontent.com/tedsluis/kubernetes-via-kubeadm/master/img/kubernetes-dashboard.gif)](https://raw.githubusercontent.com/tedsluis/kubernetes-via-kubeadm/master/img/kubernetes-dashboard.gif)
  

## Deploying Prometheus 

```
[root@nuc ~]# kubectl create namespace prometheus
namespace "prometheus" created

[root@nuc ~]# kubectl -n prometheus create -f prometheus-sa.yaml 
serviceaccount "prometheus" created

[root@nuc ~]# kubectl -n prometheus create -f prometheus-sa-clusterrolebinding.yaml
clusterrolebinding "prometheus" created

[root@nuc ~]# kubectl create -f prometheus-config.yaml -n prometheus
configmap "prometheus-config" created

[root@nuc ~]# kubectl create -f prometheus-deployment.yaml -n prometheus
deployment "prometheus" created

[root@nuc ~]# kubectl -n prometheus logs prometheus-7c7b9585f9-5lczv
level=info ts=2018-03-25T15:25:47.849051693Z caller=main.go:220 msg="Starting Prometheus" version="(version=2.2.1, branch=HEAD, revision=bc6058c81272a8d938c05e75607371284236aadc)"
level=info ts=2018-03-25T15:25:47.849106663Z caller=main.go:221 build_context="(go=go1.10, user=root@149e5b3f0829, date=20180314-14:15:45)"
level=info ts=2018-03-25T15:25:47.84913756Z caller=main.go:222 host_details="(Linux 4.15.10-300.fc27.x86_64 #1 SMP Thu Mar 15 17:13:04 UTC 2018 x86_64 prometheus-7c7b9585f9-5lczv (none))"
level=info ts=2018-03-25T15:25:47.849165208Z caller=main.go:223 fd_limits="(soft=1048576, hard=1048576)"
level=info ts=2018-03-25T15:25:47.852738361Z caller=main.go:504 msg="Starting TSDB ..."
level=info ts=2018-03-25T15:25:47.852769276Z caller=web.go:382 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2018-03-25T15:25:47.859354423Z caller=main.go:514 msg="TSDB started"
level=info ts=2018-03-25T15:25:47.859412609Z caller=main.go:588 msg="Loading configuration file" filename=/var/lib/prometheus/prometheus-config.yaml
level=info ts=2018-03-25T15:25:47.861036136Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.861868278Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.862872662Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.863871661Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.865287076Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.86634838Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.867490787Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.868484361Z caller=main.go:491 msg="Server is ready to receive web requests."

[root@nuc ~]# kubectl -n prometheus create -f prometheus-service.yaml 
service "prometheus" created

[root@nuc ~]# kubectl -n prometheus create -f node-exporter-deployment.yaml

[root@nuc kubernetes-via-kubeadm]# kubectl -n prometheus logs node-exporter-j6k7m                                                                                               
time="2018-03-30T18:41:12Z" level=info msg="Starting node_exporter (version=0.15.2, branch=HEAD, revision=98bc64930d34878b84a0f87dfe6e1a6da61e532d)" source="node_exporter.go:4$
"                                                                                                                                                                               
time="2018-03-30T18:41:12Z" level=info msg="Build context (go=go1.9.2, user=root@d5c4792c921f, date=20171205-14:50:53)" source="node_exporter.go:44"                            
time="2018-03-30T18:41:12Z" level=info msg="No directory specified, see --collector.textfile.directory" source="textfile.go:57"                                                 
time="2018-03-30T18:41:12Z" level=info msg="Enabled collectors:" source="node_exporter.go:50"                                                                                   
time="2018-03-30T18:41:12Z" level=info msg=" - time" source="node_exporter.go:52"                                                                                               
time="2018-03-30T18:41:12Z" level=info msg=" - filefd" source="node_exporter.go:52"                                                                                             
time="2018-03-30T18:41:12Z" level=info msg=" - cpu" source="node_exporter.go:52"                                                                                                
time="2018-03-30T18:41:12Z" level=info msg=" - hwmon" source="node_exporter.go:52"                                                                                              
time="2018-03-30T18:41:12Z" level=info msg=" - vmstat" source="node_exporter.go:52"                                                                                             
time="2018-03-30T18:41:12Z" level=info msg=" - zfs" source="node_exporter.go:52"                                                                                                
time="2018-03-30T18:41:12Z" level=info msg=" - infiniband" source="node_exporter.go:52"                                                                                         
time="2018-03-30T18:41:12Z" level=info msg=" - arp" source="node_exporter.go:52"                                                                                                
time="2018-03-30T18:41:12Z" level=info msg=" - textfile" source="node_exporter.go:52"                                                                                           
time="2018-03-30T18:41:12Z" level=info msg=" - stat" source="node_exporter.go:52"                                                                                               
time="2018-03-30T18:41:12Z" level=info msg=" - uname" source="node_exporter.go:52"                                                                                              
time="2018-03-30T18:41:12Z" level=info msg=" - xfs" source="node_exporter.go:52"                                                                                                
time="2018-03-30T18:41:12Z" level=info msg=" - conntrack" source="node_exporter.go:52"                                                                                          
time="2018-03-30T18:41:12Z" level=info msg=" - diskstats" source="node_exporter.go:52"                                                                                          
time="2018-03-30T18:41:12Z" level=info msg=" - bcache" source="node_exporter.go:52"                                                                                             
time="2018-03-30T18:41:12Z" level=info msg=" - netstat" source="node_exporter.go:52"                                                                                            
time="2018-03-30T18:41:12Z" level=info msg=" - loadavg" source="node_exporter.go:52"                                                                                            
time="2018-03-30T18:41:12Z" level=info msg=" - entropy" source="node_exporter.go:52"                                                                                            
time="2018-03-30T18:41:12Z" level=info msg=" - sockstat" source="node_exporter.go:52"                                                                                           
time="2018-03-30T18:41:12Z" level=info msg=" - edac" source="node_exporter.go:52"                                                                                               
time="2018-03-30T18:41:12Z" level=info msg=" - filesystem" source="node_exporter.go:52"                                                                                         
time="2018-03-30T18:41:12Z" level=info msg=" - meminfo" source="node_exporter.go:52"                                                                                            
time="2018-03-30T18:41:12Z" level=info msg=" - ipvs" source="node_exporter.go:52"                                                                                               
time="2018-03-30T18:41:12Z" level=info msg=" - mdadm" source="node_exporter.go:52"                                                                                              
time="2018-03-30T18:41:12Z" level=info msg=" - netdev" source="node_exporter.go:52"                                                                                             
time="2018-03-30T18:41:12Z" level=info msg=" - wifi" source="node_exporter.go:52"                                                                                               
time="2018-03-30T18:41:12Z" level=info msg=" - timex" source="node_exporter.go:52"                                                                                              
time="2018-03-30T18:41:12Z" level=info msg="Listening on :9100" source="node_exporter.go:76"      

[root@nuc ]# kubectl -n prometheus create -f node-exporter-service.yaml 
service "node-exporter" created
```


  
## Tear down the cluster 
Perform these steps to desolve the cluster completly.  
```
[root@nuc ~]# kubectl drain nuc.bachstraat20 --delete-local-data --force --ignore-daemonsets
node "nuc.bachstraat20" cordoned
WARNING: Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: etcd-nuc.bachstraat20, kube-apiserver-nuc.bachstraat20, kube-controller-manager-nuc.bachstraat20, kube-scheduler-nuc.bachstraat20; Ignoring DaemonSet-managed pods: kube-proxy-5ksfm
pod "kube-dns-6f4fd4bdf-kcl6k" evicted
node "nuc.bachstraat20" drained
[root@nuc ~]# kubectl delete node nuc.bachstraat20
node "nuc.bachstraat20" deleted
[root@nuc ~]# kubeadm reset
[preflight] Running pre-flight checks.
[reset] Stopping the kubelet service.
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Removing kubernetes-managed containers.
[reset] Deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes /var/lib/etcd]
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
```
  
## Documentation  
* [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)  
* [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)  
* [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init)  
* [Flannel - kubeadm](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md)  
