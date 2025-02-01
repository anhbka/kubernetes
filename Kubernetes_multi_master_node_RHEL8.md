### Install and Configure a Multi-Master HA Kubernetes Cluster with kubeadm, HAProxy and Keepalived

Mount the downloaded RHEL installation ISO to a directory like /mnt:

```
# mount -o loop <downloaded iso name> /mnt

Example:
# mount -o loop rhel-server-8.8-x86_64-dvd.iso /mnt

If you use DVD media, you can mount like below.

mount -o loop /dev/sr0  /mnt

* Note: The Warning mount: /mnt/disc: WARNING: source write-protected, mounted read-only. is expected.

#Create new repo file like below. There are two repositories in RHEL 8, named BaseOS and AppStream.

cat << EOF > /etc/yum.repos.d/my.repo 
[dvd-BaseOS]
name=DVD for RHEL - BaseOS
baseurl=file:///mnt/BaseOS/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[dvd-AppStream]
name=DVD for RHEL - AppStream/
baseurl=file:///mnt/AppStream/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

#Check whether you can get the packages list from the DVD repositories.

dnf clean all && dnf repolist
dnf groupinstall "Development Tools" -y
dnf install yum-utils vim wget conntrack-tools -y
```

| Hostname | RAM | CPU | OS | IP Address |
|----------|-----|-----|----|------------|
|  master01  | 4GB |2    |RHEL 8|192.168.99.101|
|  master02  | 4GB |2    |RHEL 8|192.168.99.102|
|  master03  | 4GB |2    |RHEL 8|192.168.99.103|
|  worker01  | 4GB |2    |RHEL 8|192.168.99.104|
|  worker02  | 4GB |2    |RHEL 8|192.168.99.105|
|  haproxy01  | 4GB |2    |RHEL 8|192.168.99.106|
|  haproxy02  | 4GB |2    |RHEL 8|192.168.99.107|

Next, add the following lines to /etc/hosts file on each instance.

```
cat << EOF >> /etc/hosts
192.168.99.101 master01 
192.168.99.102 master02 
192.168.99.103 master03 
192.168.99.104 worker01 
192.168.99.105 worker02 
192.168.99.106 haproxy01
192.168.99.107 haproxy02
EOF
```


### Run on all node

Stop the firewall on all node:

```
systemctl stop firewalld
systemctl disable firewalld
```

Disable swap on each instance so that Kubernetes cluster works smoothly. Run beneath command on each instance to disable swap space.

```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Disable SELinux on each system using following set of commands

```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

### Install HAProxy and Keepalived on `haproxy01/02`

```
dnf install -y haproxy keepalived
```

### Configure HAProxy

Add the following configuration to file `/etc/haproxy/haproxy.cfg`, keeping in mind that our virtual IP address is `192.168.99.200`

Run on both node `haproxy01/02`

Change file `/etc/haproxy/haproxy.cfg`

```
global
  log /dev/log  local0
  log /dev/log  local1 notice
  stats socket /var/lib/haproxy/stats level admin
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
  log global
  mode  http
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 192.168.99.101:6443 check fall 3 rise 2
    server master02 192.168.99.102:6443 check fall 3 rise 2
    server master03 192.168.99.103:6443 check fall 3 rise 2

listen stats 0.0.0.0:8080
    mode http
    stats enable
    stats uri /
    stats realm HAProxy\ Statistics
    stats auth admin:haproxy
```	

Enable and start the haproxy service:

```
systemctl enable --now haproxy
```

You can access HAProxy stats page by navigating to the following URL: `http://192.168.99.200:8080/` Username is `admin`, and password is `haproxy`.


Change file `/etc/keepalived/keepalived.conf`

* haproxy01

```
global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
}

# Script used to check if HAProxy is running
vrrp_script check_haproxy {
    script "killall -0 haproxy" # check the haproxy process
    interval 2 # every 2 seconds
    weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
    state MASTER # MASTER on haproxy, BACKUP on haproxy2
    interface eth0 
    virtual_router_id 255
    priority 101 # 101 on haproxy, 100 on haproxy2
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.99.200
    }
    
    track_script {
        check_haproxy
    }
}
```

* haproxy02

```
global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
}

# Script used to check if HAProxy is running
vrrp_script check_haproxy {
    script "killall -0 haproxy" # check the haproxy process
    interval 2 # every 2 seconds
    weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
    state BACKUP # MASTER on haproxy, BACKUP on haproxy2
    interface eth0 
    virtual_router_id 255
    priority 100 # 101 on haproxy, 100 on haproxy2
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.99.200
    }
    track_script {
        check_haproxy
    }
}
```

Enable and start the keepalived service:

```
sudo systemctl enable --now keepalived
```

Add the following kernel modules using modprobe command.

```
modprobe overlay
modprobe br_netfilter
```
For the permanent loading, create a file (k8s.conf) with following content.

```
tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Now, add the kernel parameters like IP forwarding. Create a file and load the parameters using sysctl command

```
tee /etc/sysctl.d/k8s.conf <<EOT
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOT

sysctl --system
```

Install Containerd

```
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

Or use curl file:

```
curl -Lo /etc/yum.repos.d/docker-ce.repo https://raw.githubusercontent.com/anhbka/Ansible/master/repo/docker-ce.repo
``` 
Install the containerd package:

```
dnf install -y containerd.io 
```

Add Kubernetes Yum Repository

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Refer: `https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/`

Install Kubeadm, kubelet & kubectl

```
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Enable the Kubelet service on all machines.

```
systemctl enable kubelet --now

#Fix warning crictl

crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock

```

*Don’t worry about any kubelet errors at this point. Once the worker nodes are successfully joined to the Kubernetes cluster using the provided join command, the kubelet service on each worker node will automatically activate and start communicating with the control plane. The kubelet is responsible for managing the containers on the node and ensuring that they run according to the specifications provided by the Kubernetes control plane.*

*NOTE: Up until this point of the installation process, we’ve installed and configured Kubernetes components on all nodes. From this point onward, we will focus on the master node.*


Great! Let’s proceed with initializing the Kubernetes control plane on the master node. Here’s how we can do it:

```
kubeadm config images pull
```

After executing this command, Kubernetes will pull the necessary container images from the default container registry (usually Docker Hub) and store them locally on the machine. This step is typically performed before initializing the Kubernetes cluster to ensure that all required images are available locally and can be used without relying on an external registry during cluster setup.

```
kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.99.200:6443 --upload-certs
```

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.99.200:6443 --token 9ldrwj.7llgwddw8y5h4ewz \
        --discovery-token-ca-cert-hash sha256:8a7360b0f963a3ac37e4ac0a7b4d14c5aa8e8f77943ed3fcf633a052bcc85b2c \
        --control-plane --certificate-key 7fa0460d57fadf02669b2715c17678fd687f4dfc4a1213b9991ee93c6e85ca9f

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.99.200:6443 --token 9ldrwj.7llgwddw8y5h4ewz \
        --discovery-token-ca-cert-hash sha256:8a7360b0f963a3ac37e4ac0a7b4d14c5aa8e8f77943ed3fcf633a052bcc85b2c
```

### Joint node master

```
kubeadm join 192.168.99.200:6443 --token 9ldrwj.7llgwddw8y5h4ewz \
        --discovery-token-ca-cert-hash sha256:8a7360b0f963a3ac37e4ac0a7b4d14c5aa8e8f77943ed3fcf633a052bcc85b2c \
        --control-plane --certificate-key 7fa0460d57fadf02669b2715c17678fd687f4dfc4a1213b9991ee93c6e85ca9f
```		

Deploy the pod network to the cluster run on Master node.

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

```
kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-564985c589-wdskm   1/1     Running   0          10m
calico-node-fcrst                          1/1     Running   0          10m
calico-node-gjgh9                          1/1     Running   0          10m
calico-node-tgkqx                          1/1     Running   0          10m
calico-node-zjbd8                          1/1     Running   0          10m
calico-node-zkfz4                          1/1     Running   0          10m
coredns-55cb58b774-6gq86                   1/1     Running   0          13m
coredns-55cb58b774-r4ld7                   1/1     Running   0          13m
etcd-master01                              1/1     Running   1          13m
etcd-master02                              1/1     Running   0          7m48s
etcd-master03                              1/1     Running   0          9m43s
kube-apiserver-master01                    1/1     Running   1          13m
kube-apiserver-master02                    1/1     Running   1          12m
kube-apiserver-master03                    1/1     Running   3          11m
kube-controller-manager-master01           1/1     Running   1          13m
kube-controller-manager-master02           1/1     Running   1          12m
kube-controller-manager-master03           1/1     Running   1          11m
kube-proxy-8d66z                           1/1     Running   1          12m
kube-proxy-9trp5                           1/1     Running   1          11m
kube-proxy-dlprw                           1/1     Running   0          12m
kube-proxy-j65p8                           1/1     Running   0          13m
kube-proxy-n955d                           1/1     Running   0          12m
kube-scheduler-master01                    1/1     Running   2          13m
kube-scheduler-master02                    1/1     Running   1          12m
kube-scheduler-master03                    1/1     Running   1          11m
```

### Initializing Kubernetes worker node

### Now we need to join worker machine node1/2 to k8 master. (Run on both worker01/02)

```
kubeadm join 192.168.99.200:6443 --token 9ldrwj.7llgwddw8y5h4ewz \
        --discovery-token-ca-cert-hash sha256:8a7360b0f963a3ac37e4ac0a7b4d14c5aa8e8f77943ed3fcf633a052bcc85b2c
```

- Run command line check on node master:

```
k get node
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   43m   v1.30.9
master02   Ready    control-plane   41m   v1.30.9
master03   Ready    control-plane   41m   v1.30.9
worker01   Ready    <none>          42m   v1.30.9
worker02   Ready    <none>          42m   v1.30.9
```

- Note: If you forget to copy the command, or can’t find it anymore, you can regenerate it by using the following command:

```
Get Join Command on Master Node

kubeadm join 192.168.99.200:6443 --token 9ldrwj.7llgwddw8y5h4ewz \
        --discovery-token-ca-cert-hash sha256:8a7360b0f963a3ac37e4ac0a7b4d14c5aa8e8f77943ed3fcf633a052bcc85b2c
```

### NGINX Test Deployment (Run on master node)

To test your Kubernetes cluster, you can deploy a simple application such as a NGINX web server. Here’s a sample YAML manifest to deploy NGINX as a test deployment:

```
vim nginx-deployment.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
```
kubectl apply -f nginx-deployment.yaml
```
```
kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           14m
```
```
k get pod
```
```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-576c6b7b6-9h67f   1/1     Running   0          3m
nginx-deployment-576c6b7b6-tgjcj   1/1     Running   0          3m
nginx-deployment-576c6b7b6-xvrwj   1/1     Running   0          3m
```

Expose NGINX to the external network:

```
vim nginx-service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort

kubectl apply -f nginx-service.yaml
```
```  
kubectl get service nginx-service
```
```
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.110.93.221   <none>        80:31738/TCP   85s
```

That’s it! Our 1-master-2-worker Kubernetes cluster is ready!
To add more nodes, simply repeat this step on other machines.

![haproxy](https://github.com/user-attachments/assets/250fcfc1-bcb3-4d80-9427-b47747001860)