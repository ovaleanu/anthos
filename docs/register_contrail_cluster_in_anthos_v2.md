# Integrate a Kubernetes and EKS cluster with Contrail in Google Anthos


Anthos is a portfolio of products and services for hybrid cloud and workload management that runs on the Google Kubernetes Engine (GKE) and users can manage workloads running also on third-party clouds like AWS, Azure and on-premise (private) clusters.
The following diagram shows Anthos components and features and how they provide Anthos's functionality across your environments, from infrastructure management to facilitating application development.

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/anthos-14-components.png)
Reference https://cloud.google.com/anthos/docs/concepts/overview

The scope of this document is integrate a on-prem K8s runing with Contrail together with an EKS with Contrail cluster and a native GKE cluster in GCP Anthos.
So, in my case I will have one GKE on Google Cloud and two attached clusters, Kubernetes with Contrail running on-prem and EKS with Contrail running in AWS.
Acording with the diagram for an attached cluster, Anthos has the following components:

1. Multi-cluster Management: Connect, Environ
2. Configuration Management: Anthos Config Management, Policy Controller
3. Service Management: Anthos Service Mesh

I will test each component from Anthos with my Contrail attached clusters.

Fun fact Anthos is flower in Greek. The reason they chose that is because flowers grow on premise, but they need rain from the cloud to flourish.


As prerequisites, I will need to install GCP cli tolls from Cloud SDK package, [Install Cloud SDK on macOS](https://cloud.google.com/sdk/docs/quickstart-macos), `kubectl` and because we will have three different clusters [`kubectx` and `kubens`](https://github.com/ahmetb/kubectx) are useful tools to managae them easily.

_Note: if `kubectl` version is lower than the [minimum supported Kubernetes version](https://cloud.google.com/kubernetes-engine/docs/release-notes) of Google Kubernetes Engine (GKE), then you need to update it_


## Creating Kubernetes clusters

### Creating on-prem Kubernetes cluster using Contrail SDN


I installed a Kubernetes cluster using Contrail SDN on BMS following the procedure from this [Wiki](https://github.com/ovaleanujnpr/Kubernetes/wiki/Installing-Kubernetes-with-Contrail).

In this case, I installed Kubernetes 1.18.9 on Ubuntu 18.04 OS.

```
$ kubectl get nodes -o wide
NAME          STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
k8s-master1   Ready    master   19h   v1.18.9   172.16.125.115   <none>        Ubuntu 18.04.5 LTS   4.15.0-118-generic   docker://18.9.9
k8s-master2   Ready    master   19h   v1.18.9   172.16.125.116   <none>        Ubuntu 18.04.5 LTS   4.15.0-118-generic   docker://18.9.9
k8s-master3   Ready    master   19h   v1.18.9   172.16.125.117   <none>        Ubuntu 18.04.5 LTS   4.15.0-118-generic   docker://18.9.9
k8s-node1     Ready    <none>   19h   v1.18.9   172.16.125.118   <none>        Ubuntu 18.04.5 LTS   4.15.0-112-generic   docker://18.9.9
k8s-node2     Ready    <none>   19h   v1.18.9   172.16.125.119   <none>        Ubuntu 18.04.5 LTS   4.15.0-112-generic   docker://18.9.9
```

After the Kubernetes cluster is deployed, I installed Contrail using single yaml file.
```
$ kubectl get po -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
config-zookeeper-4klts                1/1     Running   0          19h
config-zookeeper-cs2fk                1/1     Running   0          19h
config-zookeeper-wgrtb                1/1     Running   0          19h
contrail-agent-ch8kv                  3/3     Running   2          19h
contrail-agent-kh9cf                  3/3     Running   1          19h
contrail-agent-kqtmz                  3/3     Running   0          19h
contrail-agent-m6nrz                  3/3     Running   1          19h
contrail-agent-qgzxt                  3/3     Running   3          19h
contrail-analytics-6666s              4/4     Running   1          19h
contrail-analytics-jrl5x              4/4     Running   4          19h
contrail-analytics-x756g              4/4     Running   4          19h
contrail-configdb-2h7kd               3/3     Running   4          19h
contrail-configdb-d57tb               3/3     Running   4          19h
contrail-configdb-zpmsq               3/3     Running   4          19h
contrail-controller-config-c2226      6/6     Running   9          19h
contrail-controller-config-pbbmz      6/6     Running   5          19h
contrail-controller-config-zqkm6      6/6     Running   4          19h
contrail-controller-control-2kz4c     5/5     Running   2          19h
contrail-controller-control-k522d     5/5     Running   0          19h
contrail-controller-control-nr54m     5/5     Running   2          19h
contrail-controller-webui-5vxl7       2/2     Running   0          19h
contrail-controller-webui-mzpdv       2/2     Running   1          19h
contrail-controller-webui-p8rc2       2/2     Running   1          19h
contrail-kube-manager-88c4f           1/1     Running   0          19h
contrail-kube-manager-fsz2z           1/1     Running   0          19h
contrail-kube-manager-qc27b           1/1     Running   0          19h
coredns-684f7f6cb4-4mmgc              1/1     Running   0          93m
coredns-684f7f6cb4-dvpjk              1/1     Running   0          107m
coredns-684f7f6cb4-m6sj7              1/1     Running   0          84m
coredns-684f7f6cb4-nfkfh              1/1     Running   0          84m
coredns-684f7f6cb4-tk48d              1/1     Running   0          86m
etcd-k8s-master1                      1/1     Running   0          94m
etcd-k8s-master2                      1/1     Running   0          95m
etcd-k8s-master3                      1/1     Running   0          92m
kube-apiserver-k8s-master1            1/1     Running   0          94m
kube-apiserver-k8s-master2            1/1     Running   0          95m
kube-apiserver-k8s-master3            1/1     Running   0          92m
kube-controller-manager-k8s-master1   1/1     Running   0          94m
kube-controller-manager-k8s-master2   1/1     Running   0          95m
kube-controller-manager-k8s-master3   1/1     Running   0          92m
kube-proxy-975tn                      1/1     Running   0          108m
kube-proxy-9qzc9                      1/1     Running   0          108m
kube-proxy-fgwqt                      1/1     Running   0          109m
kube-proxy-n6nnq                      1/1     Running   0          109m
kube-proxy-wf289                      1/1     Running   0          108m
kube-scheduler-k8s-master1            1/1     Running   0          94m
kube-scheduler-k8s-master2            1/1     Running   0          95m
kube-scheduler-k8s-master3            1/1     Running   0          90m
rabbitmq-82lmk                        1/1     Running   0          19h
rabbitmq-b2lz8                        1/1     Running   0          19h
rabbitmq-f2nfc                        1/1     Running   0          19h
redis-42tkr                           1/1     Running   0          19h
redis-bj76v                           1/1     Running   0          19h
redis-ctzhg                           1/1     Running   0          19h
```
```
$ kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```

Grant the cluster-admin RBAC role
```
kubectl auth can-i '*' '*' --all-namespaces
```

### Creating a EKS cluster with Contrail in AWS

For installing an EKS cluster with Contrail in AWS, follow the installation procedure [here](https://ssd-git.juniper.net/sre/contrail-as-the-cni-for-aws-eks)

```
$ kubectl get nodes -o wide
NAME                                            STATUS   ROLES    AGE   VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
ip-100-72-0-119.eu-central-1.compute.internal   Ready    infra    30h   v1.16.15   100.72.0.119   <none>        Ubuntu 18.04.3 LTS   4.15.0-1054-aws   docker://17.3.2
ip-100-72-0-220.eu-central-1.compute.internal   Ready    <none>   30h   v1.16.15   100.72.0.220   <none>        Ubuntu 18.04.3 LTS   4.15.0-1054-aws   docker://17.3.2
ip-100-72-0-245.eu-central-1.compute.internal   Ready    infra    30h   v1.16.15   100.72.0.245   <none>        Ubuntu 18.04.3 LTS   4.15.0-1054-aws   docker://17.3.2
ip-100-72-1-116.eu-central-1.compute.internal   Ready    infra    30h   v1.16.15   100.72.1.116   <none>        Ubuntu 18.04.3 LTS   4.15.0-1054-aws   docker://17.3.2
ip-100-72-1-67.eu-central-1.compute.internal    Ready    <none>   30h   v1.16.15   100.72.1.67    <none>        Ubuntu 18.04.3 LTS   4.15.0-1054-aws   docker://17.3.2

$ kubectl get pods --all-namespaces
NAME                                          READY   STATUS    RESTARTS   AGE
cni-patches-2jm8n                             1/1     Running   0          4d21h
cni-patches-2svt6                             1/1     Running   0          4d21h
cni-patches-9mpss                             1/1     Running   0          4d21h
cni-patches-fdbws                             1/1     Running   0          4d21h
cni-patches-ggdph                             1/1     Running   0          4d21h
config-management-operator-5994858fbb-9xvmx   1/1     Running   0          2d20h
config-zookeeper-fz5zv                        1/1     Running   0          4d21h
config-zookeeper-n7wgk                        1/1     Running   0          4d21h
config-zookeeper-pjffv                        1/1     Running   0          4d21h
contrail-agent-69zpn                          3/3     Running   0          4d21h
contrail-agent-gqtfv                          3/3     Running   0          4d21h
contrail-agent-lb8tj                          3/3     Running   0          4d21h
contrail-agent-lrrp8                          3/3     Running   0          4d21h
contrail-agent-z4qjc                          3/3     Running   0          4d21h
contrail-analytics-2bv7c                      4/4     Running   0          4d21h
contrail-analytics-4jgq6                      4/4     Running   0          4d21h
contrail-analytics-sn6cj                      4/4     Running   0          4d21h
contrail-configdb-bhvlw                       3/3     Running   0          4d21h
contrail-configdb-kvvk4                       3/3     Running   0          4d21h
contrail-configdb-vbczf                       3/3     Running   0          4d21h
contrail-controller-config-8vrrm              6/6     Running   1          4d21h
contrail-controller-config-lxsms              6/6     Running   3          4d21h
contrail-controller-config-r7ncm              6/6     Running   4          4d21h
contrail-controller-control-5795l             5/5     Running   0          4d21h
contrail-controller-control-dz6pl             5/5     Running   0          4d21h
contrail-controller-control-qznf9             5/5     Running   0          4d21h
contrail-controller-webui-2g5jx               2/2     Running   0          4d21h
contrail-controller-webui-7kg48               2/2     Running   0          4d21h
contrail-controller-webui-ww5z9               2/2     Running   0          4d21h
contrail-kube-manager-2jhzc                   1/1     Running   2          4d21h
contrail-kube-manager-8psh9                   1/1     Running   0          4d21h
contrail-kube-manager-m8zg7                   1/1     Running   1          4d21h
coredns-5fdf64ff8-bf2fc                       1/1     Running   0          4d21h
coredns-5fdf64ff8-fg6fg                       1/1     Running   2          4d21h
coredns-5fdf64ff8-g4j95                       1/1     Running   7          4d21h
coredns-5fdf64ff8-s2sx8                       1/1     Running   0          4d21h
coredns-5fdf64ff8-wfltn                       1/1     Running   0          4d21h
kube-proxy-4zw9q                              1/1     Running   1          4d22h
kube-proxy-8d49f                              1/1     Running   1          4d22h
kube-proxy-p6xkw                              1/1     Running   0          4d22h
kube-proxy-s5qhd                              1/1     Running   1          4d22h
kube-proxy-z6wnp                              1/1     Running   1          4d22h
rabbitmq-fpdbv                                1/1     Running   0          4d21h
rabbitmq-hqwqm                                1/1     Running   0          4d21h
rabbitmq-wwlv8                                1/1     Running   0          4d21h
redis-6jxsf                                   1/1     Running   0          4d21h
redis-m8j6c                                   1/1     Running   0          4d21h
redis-wpbq2                                   1/1     Running   0          4d21h
upgrade-nodes-6g2l9                           1/1     Running   0          4d21h
upgrade-nodes-fcrsq                           1/1     Running   1          4d21h
upgrade-nodes-lfjll                           1/1     Running   1          4d21h
upgrade-nodes-qd64n                           1/1     Running   1          4d21h
upgrade-nodes-t9lc6                           1/1     Running   1          4d21h
```

```
$ kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```

Grant the cluster-admin RBAC role
```
kubectl auth can-i '*' '*' --all-namespaces
```

### Creating a GKE cluster in GCP

Before creating the GKE cluster, I need to create a project in GCP.
I will use `gcloud init` command to intialise the SDK:

```
$ gcloud init
```

Follow the process and create a project. Then check the project creation and project id.

```
$ gcloud projects list
```

Choose the project

```
$ gcloud config set project contrail-k8s-289615
```

I need to grant the required IAM roles to the user registering the cluster

```
PROJECT_ID=contrail-k8s-289615
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member user:[GCP_EMAIL_ADDRESS] \
 --role=roles/gkehub.admin \
 --role=roles/iam.serviceAccountAdmin \
 --role=roles/iam.serviceAccountKeyAdmin \
 --role=roles/resourcemanager.projectIamAdmin
```

I enable the APIs required for my project

```
gcloud services enable \
 --project=${PROJECT_ID} \
 container.googleapis.com \
 compute.googleapis.com \
 gkeconnect.googleapis.com \
 gkehub.googleapis.com \
 cloudresourcemanager.googleapis.com \
 cloudtrace.googleapis.com \
 anthos.googleapis.com \
 iamcredentials.googleapis.com \
 meshca.googleapis.com \
 meshconfig.googleapis.com \
 meshtelemetry.googleapis.com \
 monitoring.googleapis.com \
 logging.googleapis.com \
 runtimeconfig.googleapis.com
```

Now, I will create the GKE cluster

```
$ export KUBECONFIG=gke-config
$ gcloud container clusters create gke-cluster-1 \
--zone "europe-west2-b" \
--disk-type "pd-ssd" \
--disk-size "150GB" \
--machine-type "n2-standard-4" \
--num-nodes=3 \
--image-type "COS" \
--enable-stackdriver-kubernetes \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio,CloudRun \
--istio-config auth=MTLS_PERMISSIVE \
--cluster-version "1.17.9-gke.1504"
```

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

Because I will have to change the context often from one cluster to another, I will merge all the contexs into one configuration

Copy the $HOME/.kube/config from on-prem cluster on my mac as `contrail-config` in `~/.kube`.

Copy the ocp4 and gke configs in the same directory

```
$ cp *-config ~/.kube
$ KUBECONFIG=$HOME/.kube/eks-config:$HOME/.kube/contrail-config:$HOME/.kube/gke-config kubectl config view --merge --flatten > $HOME/.kube/config

$ kubectx gke_contrail-k8s-289615_europe-west2-b_gke-cluster-1
$ kubectx gke=.

$ kubectx arn:aws:eks:eu-central-1:927874460243:cluster/EKS-YC0U0TU5
$ kubectx eks-contrail=.

$ kubectx kubernetes-admin@kubernetes
$ kubectx onprem-k8s-contrail=.
```

Now we have three context representing the clusters

```
$ kubectx
eks-contrail
gke
onprem-k8s-contrail
```

### Configure the GCP account for Anthos

Before registering the clusters we need to create a service account and JSON file containing Google Cloud Service Account credentials for external clusters (On-premise and EKS) to connect to Anthos

```
$ PROJECT_ID=contrail-k8s-289615
$ SERVICE_ACCOUNT_NAME=anthos-connect

$ gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME} --project=${PROJECT_ID}
```

Bind the gkehub.connect IAM role to the service account:

```
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/gkehub.connect"
```

Create the service account's private key JSON file in current directory. I will need this file for registering the clusters:
```
$ gcloud iam service-accounts keys create ./${SERVICE_ACCOUNT_NAME}-svc.json \
  --iam-account=${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
  --project=${PROJECT_ID}
```

## Multi-cluster Management

### Register an External Kubernetes Cluster to GKE Connect

Connect allows you to connect any of your Kubernetes clusters to Google Cloud. This enables access to cluster and to workload management features, including a unified user interface, Cloud Console, to interact with your cluster. More details [here](https://cloud.google.com/anthos/multicluster-management/connect/overview).

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image2.png)


To register a non-GKE clusters we need to run the following command

For on-prem Contrail cluster:

```
gcloud container hub memberships register onpremk8s-contrail-cluster-1 \
   --project=${PROJECT_ID} \
   --context=onprem-k8s-contrail \
   --kubeconfig=$HOME/.kube/config \
   --service-account-key-file=./anthos-connect-svc.json
```

When the command finishes a new pod called gke-connect-agent will run in the cluster. This is responsabile to communication with GKE Hub as I decribed above.
```
$ kubectx onprem-k8s-contrail
Switched to context "onprem-k8s-contrail".

$ kubectl get pods -n gke-connect
NAMESPACE      NAME                                               READY   STATUS      RESTARTS   AGE
gke-connect    gke-connect-agent-20200918-01-00-7bc77884d-st4r2   1/1     Running     0          45m
```

_Note: I need SNAT enabled in Contrail to allow gke-connect-agent communication to internet_

The same for the Contrail EKS cluster

```
gcloud container hub memberships register eks-contrail-cluster-1 \
   --project=${PROJECT_ID} \
   --context=eks-contrail \
   --kubeconfig=$HOME/.kube/config \
   --service-account-key-file=./anthos-connect-svc.json
```
The same `gke-connect-agent` will be installed like on on-prem cluster

```
$ kubectx eks-contrail
Switched to context "eks-contrail".

$ kubectl get pods -n gke-connect
NAME                                                READY   STATUS    RESTARTS   AGE
gke-connect-agent-20201002-01-00-5749bfc847-qhvft   1/1     Running   0          43m
```

And the GKE cluster

```
gcloud container hub memberships register gke-cluster-1 \
--project=${PROJECT_ID} \
--gke-cluster=europe-west2-b/gke-cluster-1 \
--service-account-key-file=./anthos-connect-svc.json
```

I can view cluster registration status and all the clusters within my Google project

```
$ gcloud container hub memberships list
NAME                          EXTERNAL_ID
onpremk8s-contrail-cluster-1  78f7890b-3a43-4bc7-8fd9-44c76953781b
eks-contrail-cluster-1        42e532ba-a0d9-4087-baed-647be8bca7e9
gke-cluster-1                 6671599e-87af-461b-aff9-7105ebda5c66
```

To login to the external clusters from the Google Anthos Console I will use a bearer token. For this I will create a Kubernetes service account (KSA) in the cluster.

I am creating and applying first a  node-reader role-based access control (RBAC) role

```
$ cat <<EOF > node-reader.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
EOF

$ kubectx onpremk8s-contrail-cluster-1

$ kubectl apply -f node-reader.yaml
```

Create and authorise a KSA.

```
$ KSA_NAME=anthos-sa
$ kubectl create serviceaccount ${KSA_NAME}
$ kubectl create clusterrolebinding anthos-view --clusterrole view --serviceaccount default:${KSA_NAME}
$ kubectl create clusterrolebinding anthos-node-reader --clusterrole node-reader --serviceaccount default:${KSA_NAME}
$ kubectl create clusterrolebinding anthos-cluster-admin --clusterrole cluster-admin --serviceaccount default:${KSA_NAME}
```

To acquire the KSA's bearer token, run the following command:

```
$ SECRET_NAME=$(kubectl get serviceaccount ${KSA_NAME} -o jsonpath='{$.secrets[0].name}')
$ kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
```

The output token use it in Cloud Console to Login to the cluster

I will do the same for the EKS cluster

```
$ kubectx eks-contrail
$ $ kubectl apply -f node-reader.yaml

$ kubectl create serviceaccount ${KSA_NAME}
$ kubectl create clusterrolebinding anthos-view --clusterrole view --serviceaccount default:${KSA_NAME}
$ kubectl create clusterrolebinding anthos-node-reader --clusterrole node-reader --serviceaccount default:${KSA_NAME}
$ kubectl create clusterrolebinding anthos-cluster-admin --clusterrole cluster-admin --serviceaccount default:${KSA_NAME}

$ SECRET_NAME=$(kubectl get serviceaccount ${KSA_NAME} -o jsonpath='{$.secrets[0].name}')
$ kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
```

The clusters should be visibile in Anthos

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image23.png)

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image24.png)

I can view details about it in Kubernetes Engine tab

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image25.png)

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image15.png)

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image26.png)

### Deploy Anthos Apps from GCP Marketplace into Kubernetes On-premise and EKS clusters

### Deploy PostgreSQL on Contrail Kubernetes On-premise cluster

The first time you deploy an application to a Anthos attached cluster, you must also create a namespace called `application-system` for Cloud Marketplace components, and apply an imagePullSecret to the default service account for the namespace.

```
$ kubectl create ns application-system

$ kubens application-system
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "application-system".
```

To pull the images from GCR, we need to create a service account and download the associated JSON token

```
$ PROJECT_ID=contrail-k8s-289615

$ gcloud iam service-accounts create gcr-sa \
	--project=${PROJECT_ID}

$ gcloud iam service-accounts list \
	--project=${PROJECT_ID}

$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member="serviceAccount:gcr-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/storage.objectViewer"

$ gcloud iam service-accounts keys create ./gcr-sa.json \
  --iam-account="gcr-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project=${PROJECT_ID}
```

Create a secret with the contents of the token

```
$ kubectl create secret docker-registry gcr-json-key \
--docker-server=https://marketplace.gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ./gcr-sa.json)" \
--docker-email=[GCP_EMAIL_ADDRESS]
```

Patch the default service account within the namespace to use the secret to pull images from GCR instead of Docker Hub

```
$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'
```

Annotate the `application-system` namespace to enable the deployment of Kubernetes Apps from GCP Marketplace

```
$ kubectl annotate namespace application-system marketplace.cloud.google.com/imagePullSecret=gcr-json-key
```

GCP Marketplace expects a storage class by name `standard` as the default storage class.

Rename you storage class if it has a different name or create it. [Here](https://github.com/ovaleanujnpr/kubernetes/blob/master/docs/add_local_pv_k8s.md) is how to create a storage class using local volumes.

```
$ cat sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```
$ kubectl get sc
NAME                 PROVISIONER                    AGE
standard (default)   kubernetes.io/no-provisioner   6m14s
```

This will be utilized by the GCP Marketplace Apps to dynamically provision Persistent Volume (PV) and Persistent Volume Claim (PVC).


Next, we need to create and configure a namespace for the app we will deploy it from GCP Marketplace. We will deploy PostgreSQL.

```
$ kubectl create ns pgsql

$ kubens pgsql

$ kubectl create secret docker-registry gcr-json-key \
 --docker-server=https://gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ./gcr-sa.json)" \
--docker-email=[GCP_EMAIL_ADDRESS]
```

Docker-server key is pointing to https://gcr.io which holds the container images for the GCP Marketplace Apps

Like we did for `application-system` namespace, we need to patch the default service account within the pgsql namespace to use the secret to pull images from GCR instead of Docker Hub

```
$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'
```

Also annotate the pgsql namespace to enable the deployment of Kubernetes Apps from GCP Marketplace

```
$ kubectl annotate namespace pgsql marketplace.cloud.google.com/imagePullSecret=gcr-json-key
```

All these steps were a preparation to deploy an app from GCP Marketplace to the on-prem K8s cluster with Contrail

Choose PostgresSQL Server from GCP Marketplace and then click on Configure de start the deployment procedure

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image17.png)

Choose the external cluster `onpremk8s-contrail-cluster-1`

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image27.png)

Select the namespace created, storage class and click Deploy

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image19.png)

After a minute PostgresSQl is deployed

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image20.png)

```
$ kubectl get po -n pgsql
NAME                          READY   STATUS      RESTARTS   AGE
postgresql-1-deployer-nzpfn   0/1     Completed   0          91s
postgresql-1-postgresql-0     2/2     Running     0          46s

$ kubectl get pvc
NAME                                                    STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgresql-1-postgresql-pvc-postgresql-1-postgresql-0   Bound    local-pv-e00b14f6   62Gi       RWO            standard       91s
```

In GKE Console we can filter to see the applications deployed on on-prem cluster

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image21.png)

Access PostgreSQL

Forward PostgreSQL port locally:

```
$ export NAMESPACE=pgsql
$ export APP_INSTANCE_NAME="postgresql-1"
$ kubectl port-forward --namespace "${NAMESPACE}" "${APP_INSTANCE_NAME}-postgresql-0" 5432
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

Connect to the database:

```
$ apt -y install postgresql-client-10 postgresql-client-common
$ export PGPASSWORD=$(kubectl get secret "postgresql-1-secret" --output=jsonpath='{.data.password}' | base64 -d)
$ psql (10.12 (Ubuntu 10.12-0ubuntu0.18.04.1), server 9.6.18)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=#
```

### Deploy Prometheus & Grafana on Contrail EKS cluster

Let's deploy an application from Google Marketplace also on the EKS with Contrail cluster.
You will repeat some steps from above.

```
$ kubectx eks-contrail
Switched to context "eks-contrail"

$ kubectl create ns application-system

$ kubens application-system
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "application-system".

$ kubectl create secret docker-registry gcr-json-key \
--docker-server=https://marketplace.gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ./gcr-sa.json)" \
--docker-email=[GCP_EMAIL_ADDRESS]

$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'

$ kubectl annotate namespace application-system marketplace.cloud.google.com/imagePullSecret=gcr-json-key
```

As I mentioned above GCP Marketplace extects a storage class named `standard`. Thde default storage class name in EKS is `gp2`.

First remove the default flag from the gp2 storage class by patching it.

```
$  kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
storageclass.storage.k8s.io/gp2 patched
```

Now you will create a new storage class based Amazon EBS and mark it as the default one.

```
$ cat <<EOF > eks-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
EOF

$ kubectl create -f eks-sc.yaml
storageclass.storage.k8s.io/standard created

$ kubectl get sc
NAME                 PROVISIONER             AGE
gp2                  kubernetes.io/aws-ebs   2d
standard (default)   kubernetes.io/aws-ebs   5s
```

You can choose to deploy any application Anthos compatible from GCP Marketplace. I choose to deploy Prometheus & Grafana.

You will create a namespace for the applications

```
$ kubectl create ns monitoring

$ kubens monitoring

kubectl create secret docker-registry gcr-json-key \
 --docker-server=https://gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ./gcr-sa.json)" \
--docker-email=[GCP_EMAIL_ADDRESS]

$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'

$ kubectl annotate namespace monitoring marketplace.cloud.google.com/imagePullSecret=gcr-json-key
```

Choose Prometheus & Grafana from GCP Marketplace and then click on Configure de start the deployment procedure

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image28.png)

Choose the external cluster `eks-contrail-cluster-1`

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image29.png)

Select the namespace created, storage class and click Deploy

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image30.png)

After several minutes Prometheus & Grafana are deployed

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image31.png)

```
$ kubectl get pods -n monitoring
NAME                                               READY   STATUS      RESTARTS   AGE
prometheus-1-alertmanager-0                        1/1     Running     0          2m36s
prometheus-1-alertmanager-1                        1/1     Running     0          88s
prometheus-1-deployer-blm5f                        0/1     Completed   0          3m20s
prometheus-1-grafana-0                             1/1     Running     0          2m36s
prometheus-1-kube-state-metrics-6f64b67684-shtdg   2/2     Running     0          2m37s
prometheus-1-node-exporter-5scf4                   1/1     Running     0          2m36s
prometheus-1-node-exporter-gdp77                   1/1     Running     0          2m36s
prometheus-1-node-exporter-k8vfn                   1/1     Running     0          2m36s
prometheus-1-node-exporter-v6w7g                   1/1     Running     0          2m36s
prometheus-1-node-exporter-zffs9                   1/1     Running     0          2m36s
prometheus-1-prometheus-0                          1/1     Running     0          2m36s
prometheus-1-prometheus-1                          1/1     Running     0          2m36s
```

Grafana is exposed in a ClusterP-only service `prometheus-1-grafana`. To connect to Grafana UI, you can either expose a public service endpoint or keep it private, but connect from you local environment with `kubectl port-forward`.

```
kubectl get svc -n monitoring
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
prometheus-1-alertmanager            ClusterIP   10.100.92.6      <none>        9093/TCP            10m
prometheus-1-alertmanager-operated   ClusterIP   None             <none>        6783/TCP,9093/TCP   10m
prometheus-1-grafana                 ClusterIP   10.100.126.78    <none>        80/TCP              10m
prometheus-1-kube-state-metrics      ClusterIP   10.100.46.18     <none>        8080/TCP,8081/TCP   10m
prometheus-1-prometheus              ClusterIP   10.100.214.104   <none>        9090/TCP            10m
```

You can use port forwarding feature of kubectl to forward Grafana's port to your local machine. Run the following command in background:

```
$ kubectl port-forward --namespace monitoring prometheus-1-grafana-0 3000
Now you can access Grafana UI with http://localhost:3000/.
````


## Configuration Management

### Anthos Config Management

In this part will I use GitOps-style deployment to push workloads across all registered clusters through Anthos Config Management.
GitOps is a way to do Kubernetes cluster management and application delivery.  It works by using Git as a single source of truth for declarative infrastructure and applications. Since Kubernetes uses YAML or JSON files for specifications, it becomes easy to combine these artifacts with code.
Google built a tool called [Config Sync](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/overview) which acts as the bridge between an external source code repository and the Kubernetes API server. Anthos Config Management is based on Config Sync to extend it to multicluster scenarios.
I will use a GitHub repository that acts as a single source of truth for deployments and configuration. A component of ACM is installed into each of the registered Anthos clusters to monitor the external repository for any changes and synchronizing them with the cluster.

### Installing the Configuration Management Operator

The Configuration Management Operator like any other operator is a controller that manages installation of the Anthos Config Management.

I will install this operator on all three clusters, GKE, Contrail K8s on-premis and Contrail EKS.

Download the operator and apply it to each cluster

```
gsutil cp gs://config-management-release/released/latest/config-management-operator.yaml config-management-operator.yaml
```

```
$ kubectl create -f config-management-operator.yaml
customresourcedefinition.apiextensions.k8s.io/configmanagements.configmanagement.gke.io configured
clusterrolebinding.rbac.authorization.k8s.io/config-management-operator configured
clusterrole.rbac.authorization.k8s.io/config-management-operator configured
serviceaccount/config-management-operator configured
deployment.apps/config-management-operator configured
namespace/config-management-system configured
```

Run this command on all the clusters

Check if the operator was CREATED

```
$ kubectl describe crds configmanagements.configmanagement.gke.io
Name:         configmanagements.configmanagement.gke.io
Namespace:
Labels:       controller-tools.k8s.io=1.0
Annotations:  <none>
API Version:  apiextensions.k8s.io/v1
Kind:         CustomResourceDefinition
Metadata:
  Creation Timestamp:  2020-10-09T13:13:17Z
  Generation:          1
  Resource Version:    363244
  Self Link:           /apis/apiextensions.k8s.io/v1/customresourcedefinitions/configmanagements.configmanagement.gke.io
  UID:                 a088edbc-8232-419f-8f42-365fa36de110
Spec:
  Conversion:
    Strategy:  None
  Group:       configmanagement.gke.io
  Names:
    Kind:                   ConfigManagement
    List Kind:              ConfigManagementList
    Plural:                 configmanagements
    Singular:               configmanagement
....
```

### Configuring the clusters for ACM

Create an SSH keypair to allow the Operator to authenticate to your Git repository

```
$ ssh-keygen -t rsa -b 4096 -C "ovaleanujnpr" -N '' -f "~/.ssh/gke-github"
```

In my case is `ovaleanujnpr` my git repository name.

Configure your repo to recognize the newly-created public key. Details [here](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/adding-a-new-ssh-key-to-your-github-account) for Github.
Add the private key to a new Secret in the cluster. Do this step separately on every cluster.

```
$ kubectl create secret generic git-creds \
  --namespace=config-management-system \
  --from-file=ssh="/Users/ovaleanu/.ssh/gke-github"
```

I will use a sample repo from Google Cloud documentation, https://github.com/GoogleCloudPlatform/csp-config-management/. Fork this repo into your Github repo.

You will need the name for each register cluster. To find it run this command:

```
$ gcloud container hub memberships list
NAME                          EXTERNAL_ID
onpremk8s-contrail-cluster-1  78f7890b-3a43-4bc7-8fd9-44c76953781b
eks-contrail-cluster-1        42e532ba-a0d9-4087-baed-647be8bca7e9
gke-cluster-1                 6671599e-87af-461b-aff9-7105ebda5c66
```

Create the below YAML file for each cluster by replacing the clusterName with the registered clustered name in Anthos.

```
$ cat config-management.yaml
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  # clusterName is required and must be unique among all managed clusters
  clusterName:
  git:
    syncRepo: git@github.com:ovaleanujnpr/csp-config-management.git
    syncBranch: 1.0.0
    secretType: ssh
    policyDir: foo-corp
    proxy: {}
```

```
$ kubectx eks-contrail
$ kubectl apply -f config-management.yaml

$ kubectx onprem-k8s-contrail
$ kubectl apply -f config-management.yaml

$ kubectx gke
$ kubectl apply -f config-management.yaml
```

You should see the pods Running on every cluster

```
$ kubectl get pods -n config-management-system
NAME                            READY   STATUS    RESTARTS   AGE
git-importer-584bd49676-46bjq   3/3     Running   0          4m23s
monitor-c8c68d5ff-bdhzl         1/1     Running   0          4m25s
syncer-7dbbc8868c-gtp8d         1/1     Running   0          4m25s
```

and all the clusters synced on Anthos Dashboard

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image32.png)

Google offers also an utility called **nomos** which can be used to manage the ACM.

```
$ gsutil cp gs://config-management-release/released/latest/darwin_amd64/nomos nomos

$ cp ./nomos /usr/local/bin
$ chmod +x /usr/local/bin/nomos

$ nomos status
Connecting to clusters...
Current   Context               Sync Status      Last Synced Token   Sync Branch   Resource Status
-------   -------               -----------      -----------------   -----------   ---------------
*         eks-contrail          SYNCED           7da177ce            1.0.0         Healthy
          gke                   SYNCED           7da177ce            1.0.0         Healthy
          onprem-k8s-contrail   SYNCED           7da177ce            1.0.0         Healthy
```

The sameple foo-corp repo includes configs in the cluster/ and namespace/ directories. All objects managed by Anthos Config Management have the app.kubernetes.io/managed-by label set to configmanagement.gke.io.
You can list all the namespaces managed by Anthos Config management. You can find more details on [here](https://cloud.google.com/anthos-config-management/docs/how-to/configs)

```
$ kubectl get ns -l app.kubernetes.io/managed-by=configmanagement.gke.io
NAME               STATUS   AGE
audit              Active   13m
shipping-dev       Active   13m
shipping-prod      Active   13m
shipping-staging   Active   13m
```

Let's see what happens, if you will delete a namespace from Contrail EKS cluster

```
$ kubectx eks-contrail

$ kubectl delete ns audit
namespace "audit" deleted
```

Check immediately the status of the `namespace audit`

```
$ kubectl get ns audit
NAME    STATUS   AGE
audit   Active   5s
```

You can see was just created 5s ago.

With ACM, Anthos can ensure that each registered cluster has the desired state of the configuration by constantly syncing it with the ACM repo.
