---
title: "Upgrading My Kubernetes Cluster"
date: 2021-04-24T19:35:18-07:00
tags: ["k8s", "kubadm"]
draft: false
---

# Rough guide to upgrading k8s cluster with kubeadm

Disclaimer: This is not the best way, just _a_ way that works for me given the cluster topography I have (which was installed using kubeadm on ubuntu, and includes a non-HA etcd running in-cluster).

## On the control plane / master node

1. Backup etcd (manually)
   You might need the info from the etcd pod (`kubectl -n kube-system describe po etcd-master`) to find the various certs/keys/etc, but they're probably at `/etc/kubernetes/pki/etcd/`
  
    ```bash
    kubectl exec -n kube-system etcd-kmaster -- etcdctl \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \ 
        --cert=/etc/kubernetes/pki/etcd/server.crt  \
        snapshot save /var/lib/etcd/snapshot.db --ignore-daemonsets
    ```

1. Backup important files locally (_really, these should also be backed-up on a different server_)

    ```sh
    mkdir $HOME/backup
    sudo cp -r /etc/kubernetes/pki/etcd $HOME/backup/
    sudo cp /var/lib/etcd/snapshot.db $HOME/backup/$(date +%Y-%m-%d--%H-%M)-snapshot.db
    sudo cp /$HOME/kubeadm-init.yaml $HOME/backup
    ```

1. Figure out to which version we're going to upgrade.

    **Do NOT attempt to skip minor versions** (i.e. go from 1.19 -> 1.20 -> 1.21, not 1.19 directly to 1.21)

    ```sh
    sudo apt update
    sudo apt-cache madison kubeadm
    ```

    This will get you the full list of `kubeadm` versions available for your system/arch

    ```text
    kubeadm |  1.21.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.20.6-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.20.5-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.20.4-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.20.2-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.20.1-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.20.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm | 1.19.11-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm | 1.19.10-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.19.9-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.19.8-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.19.7-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    kubeadm |  1.19.6-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    ...
    ```

    ``` sh
    sudo kubeadm version
    kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.19.6", GitCommit:"8a62859e515889f07e3e3be6a1080413f17cf2c3", GitTreeState:"clean", BuildDate:"2021-04-15T03:26:21Z", GoVersion:"go1.15.10", Compiler:"gc", Platform:"linux/amd64"}
    ```

    We're going to go from 1.19.6-00 to 1.20.6-00, because that's what's currently available (and then we'll repeat this whole process to go from 1.20.6-00 to 1.21.0-00)

1. Remove the hold on kubeadm, update it, then freeze it again.

    ```bash
    sudo apt-mark unhold kubeadm
    sudo apt-get install -y kubeadm=1.20.6-00
    sudo apt-mark hold kubeadm
    ```

    Make sure it worked

    ```bash
    sudo kubeadm version
    kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.6", GitCommit:"8a62859e515889f07e3e3be6a1080413f17cf2c3", GitTreeState:"clean", BuildDate:"2021-04-15T03:26:21Z", GoVersion:"go1.15.10", Compiler:"gc", Platform:"linux/amd64"}
    ```

1. Cordon and drain the master node (There's a pod using local storage, so that extra `--delete-local-data` flag is necessary)

    ```bash
    kubectl cordon kmaster
    kubectl drain kmaster --ignore-daemonsets --delete-local-data
    ```

    Check out the upgrade plan.  We get two options, upgrade to latest in the v1.19 series (1.19.11) or upgrade to latest stable version (1.20.6)

    ```bash
    sudo kubeadm upgrade plan
    sudo kubeadm upgrade apply v1.20.6
    ```

    Nothing else needed to be upgraded, so we see:

    ```text
    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.20.6". Enjoy!
    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

    If we look at the nodes at this point, it will still show 1.19.6, which is expected.

    ```sh
    kubectl get no
    NAME        STATUS                     ROLES                  AGE    VERSION
    kmaster     Ready,SchedulingDisabled   control-plane,master   128d   v1.19.6
    kworker01   Ready                      none                   125d   v1.19.6
    ```

1. Now upgrade `kubelet` and `kubectl` to the SAME version as `kubeadm`

    ```bash
    sudo apt-mark unhold kubelet kubectl
    sudo apt-get install -y  kubelet=1.20.6-00 kubectl=1.20.6-00
    sudo apt-mark hold kubelet kubectl
    ```

    *Do not forget to freeze the kubelet/kubectl version again!*

    Restart the kubelet

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet.service
    ```

    If we look at the nodes now, we should see the master node running the updated version

    ```sh
    kubectl get no
    NAME        STATUS                     ROLES                  AGE    VERSION
    kmaster     Ready,SchedulingDisabled   control-plane,master   128d   v1.20.6
    kworker01   Ready                      none                   125d   v1.19.6
    ```

1. Uncordon it, and make sure it shows 'Ready'

1. Now drain the worker(s) and then repeat roughly the same process on the worker nodes (and yes, `--force` is necessary because I'm running something that either isn't set up correctly, or isn't playing nicely...)

    ```bash
    kubectl drain kworker01 --ignore-daemonsets --delete-local-data --force
    ```

## On the worker node(s)

1. Repeat the process from the master node

    ```bash
    sudo apt-mark unhold kubeadm
    sudo apt-get install -y kubeadm=1.20.6-00
    sudo apt-mark hold kubeadm

    sudo kubeadm upgrade node

    sudo apt-mark unhold kubelet kubectl
    sudo apt-get install -y  kubelet=1.20.6-00 kubectl=1.20.6-00
    sudo apt-mark hold kubelet kubectl

    sudo systemctl daemon-reload
    sudo systemctl restart kubelet.service
    ```

    Back on the master node, we should be able to get the nodes and see that the worker is upgraded.  

    Since it is, we can uncordon it, and it should switch to 'Ready'

    ```bash
    kubectl get no
    NAME        STATUS                     ROLES                  AGE    VERSION
    kmaster     Ready                      control-plane,master   128d   v1.20.6
    kworker01   Ready                      none                   125d   v1.20.6
    ```

That's it!  Rinse and repeat for 1.21 once the entire cluster is on 1.20
