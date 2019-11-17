
Slide: https://docs.google.com/presentation/d/12CCdLm_Sr79VOOclIMtgBPWOZJ2DV3tx7JDc0bLfxag/edit?usp=sharing

This guide provides information to
* Install vagrant and vagrant-libvirt on Leap
* Use kubespray to bring up a Kubernetes cluster with VMs
* Play rook

# Install Vagrant & Vagrant-libvirt

```
git clone https://github.com/bk201/play-rook.git
cd play-rook
sudo ./install_vagrant.sh
```

# Bring up VMs and install kubernetes

```
sudo -i
cd <play-rook-folder>

export PATH=/opt/vagrant/bin/:$PATH

python3 -m venv venv
. venv/bin/activate

git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
pip install -r requirements.txt


# Depends on your env, the firewall bited me
# systemctl stop firewalld
# systemctl restart libvirtd

# If you don't have a nice machine
# echo 1 >/sys/kernel/mm/ksm/run

vagrant up
```

# Test Kubernetes env

```
# still in kubespray folder
export PATH="$(pwd)/.vagrant/provisioners/ansible/inventory/artifacts/:$PATH"
export KUBECONFIG="$(pwd)/.vagrant/provisioners/ansible/inventory/artifacts/admin.conf"
kubectl get nodes
```

Suppose to see results like:
```
NAME    STATUS   ROLES    AGE   VERSION
k8s-1   Ready    master   10h   v1.16.2
k8s-2   Ready    master   10h   v1.16.2
k8s-3   Ready    <none>   10h   v1.16.2
```

# Deploy Rook

Ref: https://rook.io/docs/rook/master/ceph-quickstart.html

```
cd <play-rook-folder>
git clone https://github.com/rook/rook.git
cd rook
git checkout v1.1.6
cd cluster/examples/kubernetes/ceph/
```

## Create Rook CRDs and Operator

```
kubectl create -f common.yaml 
kubectl create -f operator.yaml
kubectl get pods -n rook-ceph
```

Suppose to see results like:

```
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-6648559c6c-2rxrn   1/1     Running   0          34s
rook-discover-9bhvd                   1/1     Running   0          23s
rook-discover-g24nh                   1/1     Running   0          23s
rook-discover-zm4rr                   1/1     Running   0          23s
```

## Create Ceph cluster

```
kubectl create -f cluster-test.yaml
kubectl get pods -n rook-ceph
```

Suppose to see results like (after some minutes):
```
NAME                                            READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-f6m7r                          3/3     Running     0          3m59s
csi-cephfsplugin-jkfdm                          3/3     Running     0          3m59s
csi-cephfsplugin-provisioner-75c965db4f-jsrqq   4/4     Running     0          3m59s
csi-cephfsplugin-provisioner-75c965db4f-qchs4   4/4     Running     0          3m59s
csi-cephfsplugin-tgwnm                          3/3     Running     0          3m59s
csi-rbdplugin-ck7xk                             3/3     Running     0          3m59s
csi-rbdplugin-d2xr7                             3/3     Running     0          3m59s
csi-rbdplugin-ldklp                             3/3     Running     0          3m59s
csi-rbdplugin-provisioner-56cbc4d585-92cwk      5/5     Running     0          3m59s
csi-rbdplugin-provisioner-56cbc4d585-gck9j      5/5     Running     0          3m59s
rook-ceph-mgr-a-7f87cbdbb5-n6lsw                1/1     Running     0          2m54s
rook-ceph-mon-a-7ff44884d6-bp9b7                1/1     Running     0          3m32s
rook-ceph-mon-b-66c667cdc9-bt48m                1/1     Running     0          3m25s
rook-ceph-mon-c-5c49967f55-d8x76                1/1     Running     0          3m12s
rook-ceph-operator-6648559c6c-2rxrn             1/1     Running     0          7m40s
rook-ceph-osd-prepare-k8s-1-b5pxf               0/1     Completed   0          2m31s
rook-ceph-osd-prepare-k8s-2-j92cb               0/1     Completed   0          2m31s
rook-ceph-osd-prepare-k8s-3-98kds               0/1     Completed   0          2m31s
rook-discover-9bhvd                             1/1     Running     0          7m29s
rook-discover-g24nh                             1/1     Running     0          7m29s
rook-discover-zm4rr                             1/1     Running     0          7m29s

```

## Create a toolbox pod to use Ceph CLI

Ref: https://rook.io/docs/rook/master/ceph-toolbox.html

```
kubectl create -f toolbox.yaml

# wait for toolbox pod
kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"


# Exec bash in toolbox container 
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

# Play Ceph commands inside the toolbox container

ceph -s 
ceph osd dump

# orchestrator commands
ceph orchestrator host ls
ceph orchestrator device ls
ceph orchestrator service ls
```

## Connect to Ceph Dashboard

```
kubectl create -f dashboard-external-https.yaml
kubectl -n rook-ceph get service
```

Suppose to see:
```
...
rook-ceph-mgr-dashboard-external-https   NodePort    10.233.61.254   <none>        8443:32201/TCP      12s
...
```

`32201` is the port you need to connect to. Connect to one of your VMs:

```
https://<vm_ip>:32201
```

TIP: Get IPs of vagrant VMs:
```
cd <kubespray folder>
vagrant ssh-config
```

Dashboard user is `admin`, and password can be retrieved with:
```
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode &&
 echo
```

## Create storage class for RBD

```
kubectl create -f csi/rbd/storageclass-test.yaml

# your should see a new pool on Dashboard -> Pools
```

## Claim a RBD volume in App

```
# in play-rook
cd cluster/examples/kubernetes/ceph/
kubectl create -f mysql.yaml
kubectl create -f wordpress.yaml

kubectl get pvc
```

Suppose to see:
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-pv-claim   Bound    pvc-50fb8333-219d-4e12-b45e-3fb9a3b2cf04   20Gi       RWO            rook-ceph-block   8s
wp-pv-claim      Bound    pvc-4a956fa3-1c8f-46fe-81c2-1cc734d20d45   20Gi       RWO            rook-ceph-block   5s
```

And two RBD images in Dashboard (Block --> Images).

# Cleanup

See: https://rook.io/docs/rook/master/ceph-teardown.html

Remember to clean rook folder on VMs:
```
# in kubespray dir
vagrant ssh k8s-1 -c "sudo rm -r /var/lib/rook"
vagrant ssh k8s-2 -c "sudo rm -r /var/lib/rook"
vagrant ssh k8s-3 -c "sudo rm -r /var/lib/rook"
```


