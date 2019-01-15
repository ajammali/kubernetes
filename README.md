# Kubernetes cluster upgrade to 1.11.3

## Introduction

This project describes the mounting process of a new kubernetes cluster with 1.11 k8s artifacts version.
Each role realizes a step forward to get the k8s cluster up and running.

### Prerequisites

You must have 4 CentOS7 machines with linux base packages installed.
On each server, you need to have an ansible user with your ssh key on its authorized_keys file.

You need to have a crt file in order to use for securing HTTP access to the dashboard UI.
This crt file is to be placed in files/certs directory.

### Steps

* Install python packages
* Install docker packages and make sure it is up and running
* Install kubernetes base packages (kubectl, kebelet, kubeadm,..)
* Network configuration
* Load k8s official docker images to all cluster servers
* Configure the master server and launch kubelet service using a preconfigured token (same token we use later in the join operation).
* Configure nodes servers ans launch the kubelet join command using the preconfigured token. This step should be marked by the join confirmation of these nodes to the cluster.
* Create an admin service account that will grant us admin access rights to the dashboard UI
* Deploy grafana, influxDB and heapster apps
* Deploy dashboard UI along with its RBAC configuration
* Launch kube-proxy that will give external access to our cluster, specially the dashboard UI

## K8S Provisionning

The install process can be launched using the playbook **k8s_provision.yml**

```sh
ansible-playbook -i inventories/ENV.ini -l k8s-cluster k8s_provision.yml --vault-password-file /path/to/vault/password/file
```

As you can see, I am using a vault password to encrypt some configuration values. This vault can be used to secure the uri and access credentials to
the docker registry private server, for example, or/and nexus artifactory server, the preconfigured token that we use to initiate the cluster, etc.

## Access Granted

After intitiating the cluster, join its nodes and deploy its main apps, the dashboard UI will be accessible on the port **30000**.

```ssh
https://master-node-machine-name:30000
```

Of course you can associate a VIP to this uri if you have more than one master node.

In order to log to this UI, you need to get the auto-generated token that k8s privide us after creation the admin service account.
To get this token, you need to access the master server with root rights, and do the following:

* Display the secrets list for the namespace kube-system

```sh
kubectl get secrets -n kube-system
```
This is an example of a secrets list that you can get:

```
[root@POC-kubernetes-master-a-01 ~]# kubectl get secrets -n kube-system
NAME                                             TYPE                                  DATA      AGE
admin-user-token-qj8mc                           kubernetes.io/service-account-token   3         1h
attachdetach-controller-token-dpznp              kubernetes.io/service-account-token   3         1h
bootstrap-signer-token-bh778                     kubernetes.io/service-account-token   3         1h
bootstrap-token-k8sclp                           bootstrap.kubernetes.io/token         7         1h
certificate-controller-token-zbgmp               kubernetes.io/service-account-token   3         1h
clusterrole-aggregation-controller-token-jckvq   kubernetes.io/service-account-token   3         1h
coredns-token-bt6h5                              kubernetes.io/service-account-token   3         1h
cronjob-controller-token-xlkl6                   kubernetes.io/service-account-token   3         1h
daemon-set-controller-token-lbpqq                kubernetes.io/service-account-token   3         1h
default-token-wv7vb                              kubernetes.io/service-account-token   3         1h
deployment-controller-token-5fbzl                kubernetes.io/service-account-token   3         1h
disruption-controller-token-qrvmc                kubernetes.io/service-account-token   3         1h
endpoint-controller-token-bcljp                  kubernetes.io/service-account-token   3         1h
expand-controller-token-gjwlf                    kubernetes.io/service-account-token   3         1h
flannel-token-btdpr                              kubernetes.io/service-account-token   3         1h
generic-garbage-collector-token-69cq2            kubernetes.io/service-account-token   3         1h
horizontal-pod-autoscaler-token-wpcwf            kubernetes.io/service-account-token   3         1h
job-controller-token-cff5q                       kubernetes.io/service-account-token   3         1h
kube-proxy-token-gdwhk                           kubernetes.io/service-account-token   3         1h
kubernetes-dashboard-certs                       Opaque                                1         1h
kubernetes-dashboard-key-holder                  Opaque                                2         1h
kubernetes-dashboard-token-kr2gh                 kubernetes.io/service-account-token   3         1h
namespace-controller-token-j5x6h                 kubernetes.io/service-account-token   3         1h
node-controller-token-dbftb                      kubernetes.io/service-account-token   3         1h
persistent-volume-binder-token-6t4jt             kubernetes.io/service-account-token   3         1h
pod-garbage-collector-token-dbs84                kubernetes.io/service-account-token   3         1h
pv-protection-controller-token-8jthn             kubernetes.io/service-account-token   3         1h
pvc-protection-controller-token-nrqxg            kubernetes.io/service-account-token   3         1h
replicaset-controller-token-ldxbq                kubernetes.io/service-account-token   3         1h
replication-controller-token-f98rd               kubernetes.io/service-account-token   3         1h
resourcequota-controller-token-6dfjj             kubernetes.io/service-account-token   3         1h
service-account-controller-token-hdt2j           kubernetes.io/service-account-token   3         1h
service-controller-token-8nchm                   kubernetes.io/service-account-token   3         1h
statefulset-controller-token-fzc5m               kubernetes.io/service-account-token   3         1h
token-cleaner-token-29dbw                        kubernetes.io/service-account-token   3         1h
ttl-controller-token-9cftl                       kubernetes.io/service-account-token   3         1h
```
The secret we are looking for is the one starting with admin-user-xxxxx

* Get the looging token with admin access rights:

```ssh
kubectl describe secret admin-user-xxxxx -n kube-system
```
```
Name:         admin-user-token-qj8mc
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=cbd9cad0-18c4-11e9-8706-fa163e83431a

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXFqOG1jIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjYmQ5Y2FkMC0xOGM0LTExZTktODcwNi1mYTE2M2U4MzQzMWEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9
            NaXG68Xmpde0Jap7cpRiyfnxp1L9EIHB2yUMIsT7vcT1xPxFjT6O9EWJwhEn7fK0-i6wTVv0gA-cLUnsM2WUIt2JvMT4FuSJlSoqUovMl5iH6JTVaRCfAlEP20XD78z7ZVBfE7nNH7X6rjylpmwjX96KbVEEjY3V8geDdOqs7sRc_63qM-Z9kFQVcnYV-pLCWfirDcvQJy8Ussjs3U_7cg2dyTmQxHYjr6Yr2ak4DCAdT7KsHxhi4WMcWO4wNMJKO0H-hDXSLIrKcLqbQXXWEOaKGOTp-lh0zuSWf3QTtBIlRH7rc1sVwq69t4ccIl-oWGv_gvzKwbeX6joKd3tzWA
```

This token is the one associated to the admin service account that have admin access on the kube-system namespace.

**PS** Any one of the above listed tokens can be used to access the dashboard UI, but not all of them can give you access to each namespace.

If you do not wish to have a secured dashboard UI, accessible for everyone, then you need to use the 1.8.3 version of kubernetes-dashboard docker image without RBAC configuration.