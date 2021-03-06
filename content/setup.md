# Setup a Kubernetes Cluster
This tutorial refers to a cluster of nodes (virtual, physical or a mix of both) running CentOS 7.3 Operating System. We'll set Kubernetes components as system processes managed by systemd.

   * [Requirements](#requirements)
   * [Configure Master](#configure-masters)
   * [Configure Workers](#configure-workers)
   * [Define the Containers Network Routes](#define-the-containers-network-routes)
   * [Configure DNS service](#configure-dns-service)
   
## Requirements
Our initial cluster will be made of 1 Master node and 3 Workers nodes. All machines can be virtual or physical or a mix of both. Minimum hardware requirements are: 2 vCPUs, 2GB of RAM, 16GB HDD for OS. All machines will be installed with Linux CentOS 7.3. Firewall and Selinux will be disabled. An NTP server is installed and running on all machines. On worker nodes, Docker is installed with a Device Mapper on a separate 10GB HDD. Internet access.

Here the hostnames:

   * kubem00 10.10.10.80 (master)
   * kubew03 10.10.10.83 (worker)
   * kubew04 10.10.10.84 (worker)
   * kubew05 10.10.10.85 (worker)

Make sure to enable DNS resolution for the above hostnames or set the ``/etc/hosts`` file on all the machines.

Here the releases we'll use during this tutorial

   * Kubernetes 1.7.0
   * Docker 1.12.6
   * Etcd 3.2.6

## Configure Masters
On the Master, get etcd and kubernetes

    wget https://github.com/coreos/etcd/releases/download/v3.2.6/etcd-v3.2.6-linux-amd64.tar.gz
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kube-apiserver
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kube-controller-manager
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kube-scheduler
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kubectl

Estract and install the binaries

    tar -xvf etcd-v3.2.6-linux-amd64.tar.gz && mv etcd-v3.2.6-linux-amd64/etcd* /usr/bin/
    chmod +x kube-apiserver && mv kube-apiserver /usr/bin/
    chmod +x kube-controller-manager && mv kube-controller-manager /usr/bin/
    chmod +x kube-scheduler && mv kube-scheduler /usr/bin/
    chmod +x kubectl && mv kubectl /usr/bin/

### Configure etcd
Create the etcd data directory

    mkdir -p /var/lib/etcd

Before launching and enabling the etcd service, set options in the ``/etc/systemd/system/etcd.service`` startup file

    [Unit]
    Description=etcd
    Documentation=https://github.com/coreos
    After=network.target
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=notify
    ExecStart=/usr/bin/etcd \
      --name kubem00 \
      --initial-advertise-peer-urls http://10.10.10.80:2380 \
      --listen-peer-urls http://10.10.10.80:2380 \
      --listen-client-urls http://10.10.10.80:2379,http://127.0.0.1:2379 \
      --advertise-client-urls http://10.10.10.80:2379 \
      --initial-cluster-token etcd-cluster-token \
      --initial-cluster kubem00=http://10.10.10.80:2380 \
      --initial-cluster-state new \
      --data-dir=/var/lib/etcd \
      --debug=false

    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start etcd
    systemctl enable etcd
    systemctl status etcd

### Configure API server
To configure the API server, configure the ``/etc/systemd/system/kube-apiserver.service`` startup file

    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target
    After=etcd.service

    [Service]
    Type=notify
    ExecStart=/usr/bin/kube-apiserver \
      --admission-control=NamespaceLifecycle,LimitRanger,DefaultStorageClass,ResourceQuota \
      --anonymous-auth=false \
      --etcd-servers=http://10.10.10.80:2379 \
      --advertise-address=10.10.10.80 \
      --allow-privileged=true \
      --audit-log-maxage=30 \
      --audit-log-maxbackup=3 \
      --audit-log-maxsize=100 \
      --audit-log-path=/var/lib/audit.log \
      --enable-swagger-ui=true \
      --event-ttl=1h \
      --insecure-bind-address=0.0.0.0 \
      --service-cluster-ip-range=10.32.0.0/16 \
      --service-node-port-range=30000-32767 \
      --v=2
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target


Start and enable the service

    systemctl daemon-reload
    systemctl start kube-apiserver
    systemctl enable kube-apiserver
    systemctl status kube-apiserver

### Configure controller manager
To configure the kubernetes controller manager, create the ``/etc/systemd/system/kube-controller-manager.service`` startup file

    [Unit]
    Description=Kubernetes Controller Manager
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target

    [Service]
    ExecStart=/usr/bin/kube-controller-manager \
      --address=0.0.0.0 \
      --cluster-cidr=10.38.0.0/16 \
      --cluster-name=kubernetes \
      --master=http://10.10.10.80:8080 \
      --service-cluster-ip-range=10.32.0.0/16 \
      --v=2

    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

    # Should CIDRs for pods be allocated and set by the controller manager
    # --allocate-node-cidrs=true \

Start and enable the service

    systemctl daemon-reload
    systemctl start kube-controller-manager
    systemctl enable kube-controller-manager
    systemctl status kube-controller-manager
    
### Configure scheduler
To configure the kubernetes scheduler, create the ``/etc/systemd/system/kube-scheduler.service`` startup file

    [Unit]
    Description=Kubernetes Scheduler
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target

    [Service]
    ExecStart=/usr/bin/kube-scheduler \
      --master=http://10.10.10.80:8080 \
      --v=2

    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start kube-scheduler
    systemctl enable kube-scheduler
    systemctl status kube-scheduler

### Configure CLI
On the master node, configure the ``kubectl`` command line. Please not it is not required to enable kubectl on the master node. It can be installed and configured on any machine that is able to reach the API Server.

The contest defines the namespace as well as the cluster and the user accessing the resources.

Create a context for default admin access to the cluster

    kubectl config set-credentials root
    User "root" set.

    kubectl config set-cluster unsecure-kubernetes \
            --server=http://10.10.10.80:8080
    Cluster "unsecure-kubernetes" set.

    kubectl config set-context default/unsecure-kubernetes/root \
            --cluster=unsecure-kubernetes \
            --user=root \
            --namespace=default
    Context "default/unsecure-kubernetes/root" set.

then enable the context

    kubectl config use-context default/unsecure-kubernetes/root
    Switched to context "default/unsecure-kubernetes/root"
    
and check it

    kubectl config current-context
    default/unsecure-kubernetes/root

Now it is possible to query and operate with the cluster

    kubectl get componentstatuses
    NAME                 STATUS    MESSAGE              ERROR
    scheduler            Healthy   ok
    controller-manager   Healthy   ok
    etcd-0               Healthy   {"health": "true"}

It is possible to define more contexts and switch between different contexts with different scopes and priviledges. The contexts are stored in the hidden file ``.kube/conf`` under the user home directory

    apiVersion: v1
    clusters:
    - cluster:
        server: http://10.10.10.90:8080
      name: unsecure-kubernetes
    contexts:
    - context:
        cluster: unsecure-kubernetes
        namespace: default
        user: root
      name: default/unsecure-kubernetes/admin
    current-context: default/unsecure-kubernetes/root
    kind: Config
    preferences: {}
    users:
    - name: root
      user: {}

## Configure Workers
On all the worker nodes, make sure IP forwarding is enabled

    echo "net.ipv4.ip_forward = 1" > /etc/sysctl.conf
    sysctl -p /etc/sysctl.conf

On all the worker nodes, get kubernetes

    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kubelet
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kube-proxy

Estract and install the binaries

    chmod +x kubelet && mv kubelet /usr/bin/
    chmod +x kube-proxy && mv kube-proxy /usr/bin/

On all the worker nodes, install docker

    yum -y install docker

### Configure Docker
There are a number of ways to customize the Docker daemon flags and environment variables. The recommended way from Docker web site is to use the platform-independent ``/etc/docker/daemon.json`` file instead of the systemd unit file. This file is available on all the Linux distributions

```json
{
 "debug": true,
 "storage-driver": "devicemapper",
 "iptables": false,
 "ip-masq": false
}
```

On CentOS systems, the suggested storage mapper is the Device Mapper.

Also, since Kubernetes uses a different network model than Docker, we need to prevent Docker to use NAT/IP Table rewriting, as for its default settings. For this reason, we disable the IP Table and NAT options in Docker daemon.

Start and enable the docker service

    systemctl start docker
    systemctl enable docker
    systemctl status docker

As usual, Docker will create the default ``docker0`` bridge network interface

    ifconfig docker0
    docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
            inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
            ether 02:42:c3:64:b4:7f  txqueuelen 0  (Ethernet)

However, we're not going to use it since Kubernetes networking is based on the **CNI** Container Network Interface.

### Setup the CNI network plugins
In this tutorial we are not going to provision any overlay networks for containers networking. Instead we'll rely on the simpler routing networking between the nodes. That means we need to add some static routes to our hosts.

First, make sure the IP forwarding kernel option is enabled on all hosts

    echo "net.ipv4.ip_forward = 1" > /etc/sysctl.conf
    systemctl restart network

The IP address space for containers will be allocated from the ``10.38.0.0/16`` cluster range assigned to each Kubernetes worker through the node registration process. Based on the above configuration each worker will be set with a 24-bit subnet

    * kubew03 10.38.3.0/24
    * kubew04 10.38.4.0/24
    * kubew05 10.38.5.0/24

On all the worker nodes, download and install the CNI network plugin

    wget https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz 
    mkdir -p /etc/cni/bin
    tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /etc/cni/bin

Then configure the CNI networking according to the subnets above. For example, on the kubew03 worker node

    mkdir -p /etc/cni/config
    
    vi /etc/cni/config/bridge.conf
    {
        "cniVersion": "0.3.1",
        "name": "bridge",
        "type": "bridge",
        "bridge": "cni0",
        "isGateway": true,
        "ipMasq": true,
        "ipam": {
            "type": "host-local",
            "ranges": [
              [{"subnet": "10.38.3.0/24"}]
            ],
            "routes": [{"dst": "0.0.0.0/0"}]
        }
    }
    
    vi /etc/cni/config/loopback.conf
    {
        "cniVersion": "0.3.1",
        "type": "loopback"
    }

Make the same for all worker nodes paying attention to set the correct subnet for each node.

On the first cluster activation, the above configurations will create on the worker node a bridge interface ``cni0`` having the IP address as ``10.38.3.1/24``. All containers running on that worker node, will get an IP in the above subnet having that bridge interface as default gateway.

### Configure kubelet
To configure the kubelet component, create the ``/etc/systemd/system/kubelet.service`` startup file

    [Unit]
    Description=Kubernetes Kubelet Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=docker.service
    Requires=docker.service

    [Service]
    ExecStart=/usr/bin/kubelet \
      --api-servers=http://10.10.10.80:8080 \
      --allow-privileged=true \
      --cluster-dns=10.32.0.10 \
      --cluster-domain=cluster.local \
      --container-runtime=docker \
      --cgroup-driver=systemd \
      --serialize-image-pulls=false \
      --register-node=true \
      --network-plugin=cni \
      --cni-bin-dir=/etc/cni/bin \
      --cni-conf-dir=/etc/cni/config \
      --v=2

    Restart=on-failure

    [Install]
    WantedBy=multi-user.target


The cluster DNS parameter above refers to the IP of the DNS internal service used by Kubernetes for service discovery. It should be in the IP addressese space used for internal services we set in the ``--service-cluster-ip-range`` option of the controller manager.

Start and enable the kubelet service

    systemctl daemon-reload
    systemctl start kubelet
    systemctl enable kubelet
    systemctl status kubelet

### Configure proxy
To configure the proxy component, edit the ``/etc/systemd/system/kube-proxy.service`` startup file

    [Unit]
    Description=Kubernetes Kube-Proxy Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target

    [Service]
    ExecStart=/usr/bin/kube-proxy \
      --master=http://10.10.10.80:8080 \
      --cluster-cidr=10.38.0.0/16 \
      --proxy-mode=iptables \
      --v=2

    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start kube-proxy
    systemctl enable kube-proxy
    systemctl status kube-proxy


## Define the Containers Network Routes
The cluster should be now running. Check to make sure the cluster can see the nodes, by querying the master

    kubectl get nodes
    
    NAME      STATUS    AGE       VERSION
    kubew03   Ready     12m       v1.7.0
    kubew04   Ready     1m        v1.7.0
    kubew05   Ready     1m        v1.7.0

Now that each worker node is online we need to add routes to make sure that containers running on different machines can talk to each other. On the master node, given ``eth0`` the cluster network interface, create the script file ``/etc/sysconfig/network-scripts/route-eth0`` for adding permanent static routes containing the following

    # The pod network 10.38.3.0/24 is reachable through the worker node kubew03
    10.38.3.0/24 via 10.10.10.83
    
    # The pod network 10.38.4.0/24 is reachable through the worker node kubew04
    10.38.4.0/24 via 10.10.10.84
    
    # The pod network 10.38.2.0/24 is reachable through the worker node kubew05
    10.38.5.0/24 via 10.10.10.85

Restart the network service and check the master routing table

    systemctl restart network
    
    netstat -nr
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.10.1      0.0.0.0         UG        0 0          0 eth0
    10.10.10.0      0.0.0.0         255.255.255.0   U         0 0          0 eth0
    10.38.3.0       10.10.10.83     255.255.255.0   UG        0 0          0 eth0
    10.38.4.0       10.10.10.84     255.255.255.0   UG        0 0          0 eth0
    10.38.5.0       10.10.10.85     255.255.255.0   UG        0 0          0 eth0

On all the worker nodes, create a similar script file. For example, on the worker ``kubew03``, create the file ``/etc/sysconfig/network-scripts/route-eth0`` containing the following

    # The pod network 10.38.4.0/24 is reachable through the worker node kubew04
    10.38.4.0/24 via 10.10.10.84
    
    # The pod network 10.38.5.0/24 is reachable through the worker node kubew05
    10.38.5.0/24 via 10.10.10.85

Restart the network service and check the master routing table

    systemctl restart network
    
    netstat -nr
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.10.1      0.0.0.0         UG        0 0          0 eth0
    10.10.10.0      0.0.0.0         255.255.255.0   U         0 0          0 eth0
    10.38.4.0       10.10.10.84     255.255.255.0   UG        0 0          0 eth0
    10.38.5.0       10.10.10.85     255.255.255.0   UG        0 0          0 eth0
    172.17.0.0      0.0.0.0         255.255.0.0     U         0 0          0 docker0

Repeat the steps for all workers paying attention to set the routes correctly.

## Configure DNS service
To enable service name discovery in our Kubernetes cluster, we need to configure an embedded DNS service. To do so, we need to deploy DNS pod and service having configured kubelet to resolve all DNS queries from this local DNS service.

Login to the master node and download the DNS template ``kubedns-template.yaml`` from [here](https://github.com/kalise/Kubernetes-Lab-Tutorial/blob/master/examples/kubedns-template.yaml)

This template defines a Replica Controller and a DNS service. The controller defines four containers running on the same pod: a DNS server, a dnsmasq for caching, a dnsmasq metric collector and healthz for liveness probe:
```yaml
...
    spec:
      containers:
      - name: kubedns
        image: gcr.io/google_containers/kubedns-amd64:1.9
...
      - name: dnsmasq
        image: gcr.io/google_containers/kube-dnsmasq-amd64:1.4
...
      - name: dnsmasq-metrics
        image: gcr.io/google_containers/dnsmasq-metrics-amd64:1.0
...
      - name: healthz
        image: gcr.io/google_containers/exechealthz-amd64:1.2
```

Make sure you have set the master IP address parameter to pass to the kubedns container
```
...
        # This is used to reach kubernetes master
        # It should be set only if service accounts are not used
        - --kube-master-url=http://10.10.10.80:8080
...
```

*Note: the above is required only if service accounts are not used.*

Make sure you have set the correct cluster IP address parameter as we have specified in ``--cluster-dns`` option of the kubelet startup configuration 
```yaml
...
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.32.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

Create the DNS setup from the template

    [root@kube00 ~]# kubectl create -f kubedns-template.yaml
    replicationcontroller "kube-dns-v20" created
    service "kube-dns" created

and check if it works in the dedicated namespace

    [root@kube00 ~]# kubectl get all -n kube-system
    NAME              DESIRED   CURRENT   READY     AGE
    rc/kube-dns       1         1         1         22m
    NAME           CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
    svc/kube-dns   10.32.0.10     <none>        53/UDP,53/TCP   22m
    NAME                    READY     STATUS    RESTARTS   AGE
    po/kube-dns-3xk4v       4/4       Running   0          22m

To test if it works, create a file named ``busybox.yaml`` with the following contents:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

Then create a pod using this file

    kubectl create -f busybox.yaml
    
wait for pod is running and validate that DNS is working by resolving the kubernetes service

    kubectl exec -ti busybox -- nslookup kubernetes    
    Server:    10.254.3.100
    Address 1: 10.254.3.100 kube-dns.kube-system.svc.cluster.local
    Name:      kubernetes
    Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local

Take a look inside the ``resolv.conf file`` of the busybox container
    
    kubectl exec busybox cat /etc/resolv.conf
    search default.svc.cluster.local svc.cluster.local cluster.local
    nameserver 10.254.3.100
    nameserver 8.8.8.8
    options ndots:5

Each time a new service starts on the cluster, it will register with the DNS letting all the pods to reach the new service.
