# Integrate a Kubernetes cluster with Contrail in Google Anthos

Anthos is a portfolio of products and services for hybrid cloud and workload management that runs on the Google Kubernetes Engine (GKE) and users can manage workloads running also on third-party clouds like AWS, Azure and on-premises (private) clusters.
The scope of this document is integrate an on-prem K8s runing with Contrail toghther with a GKE and EKS cluster in GCP Anthos.

Fun fact Anthos is flower in Greek. The reason they chose that is because flowers grow on premise, but they need rain from the cloud to flourish.

Google Cloud Platform console provides a single control plane for managing Kubernetes clusters deployed in multiple locations.

As prerequisites, I will need to install GCP cli tolls from Cloud SDK package, [Install Cloud SDK on macOS](https://cloud.google.com/sdk/docs/quickstart-macos), [`eksctl` cli tool](https://eksctl.io/introduction/#getting-started) for creating EKS cluster on AWS, `kubectl` and because we will have three different clusters [`kubectx` and `kubens`](https://github.com/ahmetb/kubectx) are useful tools to managae them easily.

_Note: if `kubectl` version is lower than the [minimum supported Kubernetes version](https://cloud.google.com/kubernetes-engine/docs/release-notes) of Google Kubernetes Engine (GKE), then you need to update it_

## Creating Kubernetes clusters

### Creating on-prem Kubernetes cluster using Contrail SDN


I installed a Kubernetes cluster using Contrail SDN on BMS following the procedure from this [Wiki](https://github.com/ovaleanujnpr/Kubernetes/wiki/Installing-Kubernetes-with-Contrail).

In this case, I installed Kubernetes 1.16.11 on Ubuntu 18.04 OS.

```
$ kubectl get nodes -o wide
NAME                      STATUS   ROLES    AGE   VERSION    INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
r9-ru24.csa.juniper.net   Ready    <none>   16h   v1.16.11   192.168.213.114   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://18.9.9
r9-ru25.csa.juniper.net   Ready    <none>   16h   v1.16.11   192.168.213.113   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://18.9.9
r9-ru26.csa.juniper.net   Ready    master   16h   v1.16.11   192.168.213.112   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://18.9.9
```

After the Kubernetes cluster is deployed, I installed Contrail using single yaml file.
```
$ kubectl get po -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
config-zookeeper-6kqsf                            1/1     Running   0          16h
contrail-agent-qpxqd                              3/3     Running   0          16h
contrail-agent-vrcsl                              3/3     Running   0          16h
contrail-agent-xvbhq                              3/3     Running   0          16h
contrail-analytics-alarm-m7rhg                    4/4     Running   0          16h
contrail-analytics-mnx2q                          4/4     Running   0          16h
contrail-analytics-snmp-v5dzj                     4/4     Running   0          16h
contrail-analyticsdb-z6cfm                        4/4     Running   0          16h
contrail-configdb-jp9sg                           3/3     Running   0          16h
contrail-controller-config-7zn25                  6/6     Running   0          16h
contrail-controller-control-7nf97                 5/5     Running   0          16h
contrail-controller-webui-7xgp7                   2/2     Running   0          16h
contrail-kube-manager-qqh52                       1/1     Running   0          16h
coredns-5644d7b6d9-njzsm                          1/1     Running   0          16h
coredns-5644d7b6d9-rfn8t                          1/1     Running   0          16h
etcd-r9-ru26.csa.juniper.net                      1/1     Running   0          16h
kube-apiserver-r9-ru26.csa.juniper.net            1/1     Running   0          16h
kube-controller-manager-r9-ru26.csa.juniper.net   1/1     Running   0          16h
kube-proxy-nkh65                                  1/1     Running   0          16h
kube-proxy-nvmwd                                  1/1     Running   0          16h
kube-proxy-xqrsd                                  1/1     Running   0          16h
kube-scheduler-r9-ru26.csa.juniper.net            1/1     Running   0          16h
rabbitmq-f4tws                                    1/1     Running   0          16h
redis-xlzvb                                       1/1     Running   0          16h
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

### Creating a EKS cluster in AWS

I installed an EKS cluster using `eksctl` cli tool. Make sure [aws credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) are configured on your station.

Create a working directory

```
$ mkdir ~/anthos ; cd ~/anthos
```

```
$ export KUBECONFIG=eks-config
$ eksctl create cluster \
--name eks-cluster-1 \
--version 1.16 \
--nodegroup-name eks-workers \
--node-type t3.medium \
--node-volume-size=150 \
--nodes 3 \
--nodes-min 3 \
--nodes-max 6 \
--node-ami auto \
--node-ami-family Ubuntu1804 \
--ssh-access \
--region=eu-west-3 \
--set-kubeconfig-context=true
```

```
$ kubectl get nodes
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-59-157.eu-west-3.compute.internal   Ready    <none>   18h   v1.16.9
ip-192-168-9-2.eu-west-3.compute.internal      Ready    <none>   18h   v1.16.9
ip-192-168-90-91.eu-west-3.compute.internal    Ready    <none>   18h   v1.16.9

$ kubectl get pods -A
NAME                       READY   STATUS    RESTARTS   AGE
aws-node-8rn5b             1/1     Running   0          18h
aws-node-dnl8x             1/1     Running   0          18h
aws-node-nwj6n             1/1     Running   0          18h
coredns-84744d8475-6njk7   1/1     Running   0          18h
coredns-84744d8475-pt8g6   1/1     Running   0          18h
kube-proxy-6jsd7           1/1     Running   0          18h
kube-proxy-9kxgq           1/1     Running   0          18h
kube-proxy-jb49b           1/1     Running   0          18h
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
PROJECT_ID            NAME              PROJECT_NUMBER
dotted-ranger-283911  My First Project  482168856288
trusty-wares-283912   Contrail          234378606342
```

Choose the projects

```
$ gcloud config set project trusty-wares-283912
```

I need to grant the required IAM roles to the user registering the cluster

```
PROJECT_ID=trusty-wares-283912
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
 gkeconnect.googleapis.com \
 gkehub.googleapis.com \
 cloudresourcemanager.googleapis.com \
 anthos.googleapis.com \
 iamcredentials.googleapis.com \
 meshca.googleapis.com \
 meshconfig.googleapis.com \
 meshtelemetry.googleapis.com \
 monitoring.googleapis.com \
 runtimeconfig.googleapis.com
```

Now, I will create the GKE cluster

```
$ export KUBECONFIG=gke-config
$ gcloud container clusters create gke-cluster-1 \
--zone europe-west3-a \
--disk-type=pd-ssd \
--disk-size=80GB \
--machine-type=n1-standard-1 \
--num-nodes=3 \
--image-type ubuntu \
--cluster-version=1.16.11-gke.5
```

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

Because I will have to change the context often from one cluster to another, I will merge all the contexs into one configuration

Copy the $HOME/.kube/config from on-prem cluster on my mac as `contrail-config` in `~/.kube`.

Copy the eks and gke configs in the same directory

```
$ cp *-config ~/.kube
$ KUBECONFIG=$HOME/.kube/eks-config:$HOME/.kube/contrail-config:$HOME/.kube/gke-config
$ kubectl config view --merge --flatten &gt; $HOME/.kube/config
$ export KUBECONFIG=

$ kubectx gke_trusty-wares-283912_europe-west3-a_gke-cluster-1
$ kubectx gke=.

$ kubectx iam-root-account@eks-cluster-1.eu-west-3.eksctl.io
$ kubectx eks=.

$ kubectx kubernetes-admin@kubernetes
$ kubectx contrail=.
```

Now we have three context representing the clusters

```
$ kubectx
gke
eks
contrail
```

### Configure the GCP account for Anthos

Before registering the clusters we need to create a service account and JSON file containing Google Cloud Service Account credentials for external clusters (on-prem and EKS) to connect to Anthos

```
$ PROJECT_ID=trusty-wares-283912
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

### Register an External Kubernetes Cluster to GKE Connect

Connect allows you to connect any of your Kubernetes clusters to Google Cloud. This enables access to cluster and to workload management features, including a unified user interface, Cloud Console, to interact with your cluster. More details [here](https://cloud.google.com/anthos/multicluster-management/connect/overview).

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image2.png)


To register a non-GKE clusters we need to run the following command

For on-prem Contrail cluster:

```
gcloud container hub memberships register contrail-cluster-1 \
   --project=${PROJECT_ID} \
   --context=contrail \
   --kubeconfig=$HOME/.kube/config \
   --service-account-key-file=./anthos-connect-svc.json
```


When the command finishes a new pod called gke-connect-agent will run in the cluster. This is responsabile to communication with GKE Hub as I decribed above.
```
$ kubectx contrail
Switched to context "contrail".

$ kubectl get pods -A
NAMESPACE     NAME                                                READY   STATUS    RESTARTS   AGE
gke-connect   gke-connect-agent-20200717-00-00-58c749c9d7-l9v66   1/1     Running   0          23m
kube-system   config-zookeeper-6kqsf                              1/1     Running   0          16h
kube-system   contrail-agent-qpxqd                                3/3     Running   0          16h
kube-system   contrail-agent-vrcsl                                3/3     Running   0          16h
kube-system   contrail-agent-xvbhq                                3/3     Running   0          16h
kube-system   contrail-analytics-alarm-m7rhg                      4/4     Running   0          16h
kube-system   contrail-analytics-mnx2q                            4/4     Running   0          16h
kube-system   contrail-analytics-snmp-v5dzj                       4/4     Running   0          16h
kube-system   contrail-analyticsdb-z6cfm                          4/4     Running   0          16h
kube-system   contrail-configdb-jp9sg                             3/3     Running   0          16h
kube-system   contrail-controller-config-7zn25                    6/6     Running   0          16h
kube-system   contrail-controller-control-7nf97                   5/5     Running   0          16h
kube-system   contrail-controller-webui-7xgp7                     2/2     Running   0          16h
kube-system   contrail-kube-manager-qqh52                         1/1     Running   0          16h
kube-system   coredns-5644d7b6d9-njzsm                            1/1     Running   0          16h
kube-system   coredns-5644d7b6d9-rfn8t                            1/1     Running   0          16h
kube-system   etcd-r9-ru26.csa.juniper.net                        1/1     Running   0          16h
kube-system   kube-apiserver-r9-ru26.csa.juniper.net              1/1     Running   0          16h
kube-system   kube-controller-manager-r9-ru26.csa.juniper.net     1/1     Running   0          16h
kube-system   kube-proxy-nkh65                                    1/1     Running   0          16h
kube-system   kube-proxy-nvmwd                                    1/1     Running   0          16h
kube-system   kube-proxy-xqrsd                                    1/1     Running   0          16h
kube-system   kube-scheduler-r9-ru26.csa.juniper.net              1/1     Running   0          16h
kube-system   rabbitmq-f4tws                                      1/1     Running   0          16h
kube-system   redis-xlzvb                                         1/1     Running   0          16h
```

_Note: I need SNAT enabled in Contrail to allow gke-connect-agent communication to internet_

The same for the EKS cluster

```
gcloud container hub memberships register eks-cluster-1 \
   --project=${PROJECT_ID} \
   --context=eks \
   --kubeconfig=$HOME/.kube/config \
   --service-account-key-file=./anthos-connect-svc.json
```
The same `gke-connect-agent` will be installed like on on-prem cluster

```
$ kubectx eks
Switched to context "eks".

$ kubectl get pods -A
NAMESPACE     NAME                                                READY   STATUS    RESTARTS   AGE
gke-connect   gke-connect-agent-20200724-01-00-57895588b7-c6flv   1/1     Running   0          125m
kube-system   aws-node-8rn5b                                      1/1     Running   0          19h
kube-system   aws-node-dnl8x                                      1/1     Running   0          19h
kube-system   aws-node-nwj6n                                      1/1     Running   0          19h
kube-system   coredns-84744d8475-6njk7                            1/1     Running   0          19h
kube-system   coredns-84744d8475-pt8g6                            1/1     Running   0          19h
kube-system   kube-proxy-6jsd7                                    1/1     Running   0          19h
kube-system   kube-proxy-9kxgq                                    1/1     Running   0          19h
kube-system   kube-proxy-jb49b                                    1/1     Running   0          19h
```

And the GKE cluster

```
gcloud container hub memberships register gke-cluster-1 \
--project=${PROJECT_ID} \
--gke-cluster=europe-west3-a/gke-cluster-1 \
--service-account-key-file=./anthos-connect-svc.json
```

I can view cluster registration status and all the clusters within my Google project

```
$ gcloud container hub memberships list
NAME                EXTERNAL_ID
NAME                EXTERNAL_ID
contrail-cluster-1  da221221-0f05-491c-8fe2-2eb4452a593d
gke-cluster-1       193af6eb-790b-444b-9eb0-62f03ace7c76
eks-cluster-1       6418ecdc-cf0e-448b-ade7-1d32a892309c
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

$ kubectx contrail

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

I will do the same for EKS cluster

```
$ kubectx eks
$ $ kubectl apply -f node-reader.yaml

$ kubectl create serviceaccount ${KSA_NAME}
$ kubectl create clusterrolebinding anthos-view --clusterrole view --serviceaccount default:${KSA_NAME}
$ kubectl create clusterrolebinding anthos-node-reader --clusterrole node-reader --serviceaccount default:${KSA_NAME}
$ kubectl create clusterrolebinding anthos-cluster-admin --clusterrole cluster-admin --serviceaccount default:${KSA_NAME}
```


The clusters should be visibile in Anthos

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image4.png)

I can view details about it in Kubernetes Engine tab

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image5.png)

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image6.png)

### Deploy Anthos Apps from GCP Marketplace into Kubernetes on-prem cluster

The first time you deploy an application to a Anthos GKE on-prem cluster, you must also create a namespace called `application-system` for Cloud Marketplace components, and apply an imagePullSecret to the default service account for the namespace.

```
$ kubectl create ns application-system

$ kubens application-system
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "application-system".
```

To pull the images from GCR, we need to create a service account and download the associated JSON token

```
$ PROJECT="trusty-wares-283912"

$ gcloud iam service-accounts create gcr-sa \
	--project=${PROJECT}

$ gcloud iam service-accounts list \
	--project=${PROJECT}

$ gcloud projects add-iam-policy-binding ${PROJECT} \
 --member="serviceAccount:gcr-sa@${PROJECT}.iam.gserviceaccount.com" \
 --role="roles/storage.objectViewer"

$ gcloud iam service-accounts keys create ./gcr-sa.json \
  --iam-account="gcr-sa@${PROJECT}.iam.gserviceaccount.com" \
  --project=${PROJECT}
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

Choose PostgresSQl Server from GCP Marketplace

Click on Configure de start the deployment procedure

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image8.png)

Choose the external cluster, `contrail-cluster-1`

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image9.png)

Select the namespace created, storage class and click Deploy

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image10.png)

After a minute PostgresSQl is deployed

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image11.png)

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

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image12.png)

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
