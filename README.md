# Docker_and_Kubernetes_Installation


Tasks to do in every node:

1. Turn off swap memory
vi /etc/fstab		
	#/dev/mapper/centos-swap swap                    swap    defaults        0 0
	 <use hash for count this line is comment line>
swapoff -a

free -m

2. Turn off firewall 
systemctl stop firewalld && systemctl disable firewalld

systemctl status firewalld



Hostname Change:
hostnamectl set-hostname master

3. Update hosts
cat <<EOF >>  /etc/hosts
192.168.0.120 master
EOF
4. Load the br_netfilter module


modprobe br_netfilter

5. Turn off SELINUX 
vi /etc/selinux/config
	SELINUX=disabled

reboot

getenforce

4. Remove existing version of docker (if exists)
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine -y

5. Install docker on linux machine.

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

yum update -y && yum install -y \
  containerd.io-1.2.10 \
  docker-ce-19.03.4 \
  docker-ce-cli-19.03.4

mkdir /etc/docker

cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload

systemctl restart docker && systemctl enable docker









6. Install kubernetes on linux machine.
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF


sysctl --system

Task to do in only Master node

1. Initiate cluster:
kubeadm init --control-plane-endpoint "master:6443" --pod-network-cidr=10.244.0.0/16 

Copy token (Example below)
Example: kubeadm join 192.168.10.123:6443 --token d612m1.9l18chn3o19iigmf \
    --discovery-token-ca-cert-hash sha256:c6c2a354670ef9d0c7145a497f8d67892d0c4965304d2264b81e705c2f746f11

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
or
export KUBECONFIG=/etc/kubernetes/admin.conf
##
vi webnet.yml
2. Configure flannel:
kubectl apply -f webnet.yml

or kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

3.Untainted master:
kubectl taint nodes --all node-role.kubernetes.io/master-

Task to do in Worker node

1. worker node connect with cluster using token.

Example: kubeadm join 192.168.10.123:6443 --token d612m1.9l18chn3o19iigmf \
    --discovery-token-ca-cert-hash sha256:c6c2a354670ef9d0c7145a497f8d67892d0c4965304d2264b81e705c2f746f11

INSTALL DASHBOARD (Task to do in only master node)

1. Install dashboard: 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml

2. Change type ClusterIP to NodePort
kubectl -n kubernetes-dashboard  edit service kubernetes-dashboard
 

kubectl get services --all-namespaces

3. Create admin user for kubernetes
cat <<EOF >  dashboard-adminuser.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF

kubectl apply -f dashboard-adminuser.yml

cat <<EOF >  admin-role-binding.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF

kubectl apply -f admin-role-binding.yml

4. View admin user token
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

kubectl get service --all-namespaces			 <for port number > 
 

kubectl get pod -A

 


5. open your PC browser.
Example LIKE this <https://LinuxMachineIP: portnumber"
  <https://172.16.0.23:31969/#/login>

6. Create readonly user
# cat <<EOF >  readonly-user.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dashboard-viewonly
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - nodes
  - persistentvolumeclaims
  - persistentvolumes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - clusterroles
  - roles
  - rolebindings
  verbs:
  - get
  - list
  - watch
EOF
# kubectl apply -f readonly-user.yaml
# Create the service account in the current namespace 
# (we assume default)
kubectl create serviceaccount dashboard-view-only
# Give that service account root on the cluster
kubectl create clusterrolebinding dashboard-view-only \
  --clusterrole=dashboard-viewonly \
  --serviceaccount=default:dashboard-view-only
# Find the secret that was created to hold the token for the SA
kubectl get secrets
# Show the contents of the secret to extract the token
kubectl describe secret dashboard-view-only-token-XXXXXX

7. Install metrics Server:
# wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6
# tar -xzf v0.3.6.tar.gz
# kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
# kubectl get deployment metrics-server -n kube-system
>CHANGE:
Matrics server deployment:
Add below after image block...
          args:
            - '/metrics-server'
            - '--logtostderr'
            - '--kubelet-insecure-tls=true'
            - '--kubelet-preferred-address-types=InternalIP'
            - '--v=2'
Now system is ready to deploy
