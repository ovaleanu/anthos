## Integrate a Kubernetes cluster with Contrail in Google Anthos

Anthos is a portfolio of products and services for hybrid cloud and workload management that runs on the Google Kubernetes Engine (GKE) and users can manage workloads running also on third-party clouds like AWS, Azure and on-premises (private) clusters.
The scope of this document is integrate an on-prem K8s cluster with Contrail with Anthos.

Fun fact Anthos is flower in Greek. The reason they chose that is because flowers grow on premise, but they need rain from the cloud to flourish.

Google Cloud Platform console provides a single control plane for managing Kubernetes clusters deployed in multiple locations.

Deploy a GKE cluster and then register it to Anthos

[GKE quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)

[Explore Anthos](https://cloud.google.com/anthos/docs/tutorials/explore-anthos)


![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image1.png)


Install a Kubernetes cluster with Contrail on any VM/BMS on-prem. Check this [Wiki](https://github.com/ovaleanujnpr/Kubernetes/wiki/Installing-Kubernetes-with-Contrail) for deployment.

In this case I installed Kubernetes 1.16.11 on BMS with Ubuntu 18.04 OS.

```
# kubectl get nodes -o wide
NAME                      STATUS   ROLES    AGE   VERSION    INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
r9-ru24.csa.juniper.net   Ready    <none>   16h   v1.16.11   192.168.213.114   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://18.9.9
r9-ru25.csa.juniper.net   Ready    <none>   16h   v1.16.11   192.168.213.113   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://18.9.9
r9-ru26.csa.juniper.net   Ready    master   16h   v1.16.11   192.168.213.112   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://18.9.9
```

After the Kubernetes cluster is deployed I install Contrail using single yaml file.
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

Now, it is time to register our on-prem Kubernetes cluster to Anthos.

### Register an External Kubernetes Cluster to GKE Connect

Connect allows users to connect the on-prem Kubernetes clusters as well as Kubernetes clusters running on other public clouds with the Google cloud platform. Connect uses an encrypted connection between the Kubernetes clusters and the Google cloud platform project and enables authorized users to login to clusters, access details about their resources, projects, and clusters, and to manage cluster infrastructure and workloads whether they are running on Google’s hardware or elsewhere.
When remote clusters are registered, a GKE Connect Agent is deployed to the cluster which manages connectivity to various API endpoints on GCP. The cluster doesn’t require a public IP and just needs reachability to a set of googleapis. Connect agent uses an authenticated and encrypted connection from the Kubernetes cluster to GCP. Connect agent uses VPC Service Controls to ensure that GCP is an extension of users private cloud and can traverse NATs and firewalls. All user interactions with clusters are visible in Kubernetes audit logs.

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image2.png)

To register a remote cluster we need kubeconfig of the cluster, `gcloud` cli tool and some RBAC settings.

Cluster registration can be made using GKE console as shown below or cli. I will use cli to register a remote Kubernetes cluster with Contrail.

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image3.png)

Before we begin I will set permissive RBAC permissions

```
$ kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```

Then I need to install Cloud SDK. This will install `gcloud` cli tool. See [here](https://cloud.google.com/sdk/install) for installation procedure on Mac, Linux and Windows.
We will continue with Ubuntu installation.

Add the Cloud SDK distribution URI as a package source
```
$ echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
```

Make sure apt-transport is installed

```
$ sudo apt-get install apt-transport-https ca-certificates gnupg
```

Import the Google Cloud public key

```
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
```

Update and install Cloud SDK

```
$ sudo apt-get update && sudo apt-get install google-cloud-sdk
```

Optional if needed, install any of these optional components:

```
google-cloud-sdk-app-engine-python
google-cloud-sdk-app-engine-python-extras
google-cloud-sdk-app-engine-java
google-cloud-sdk-app-engine-go
google-cloud-sdk-bigtable-emulator
google-cloud-sdk-cbt
google-cloud-sdk-cloud-build-local
google-cloud-sdk-datalab
google-cloud-sdk-datastore-emulator
google-cloud-sdk-firestore-emulator
google-cloud-sdk-pubsub-emulator
kubectl
```

_Note: if `kubectl` version is lower than the [minimum supported Kubernetes version](https://cloud.google.com/kubernetes-engine/docs/release-notes) of Google Kubernetes Engine (GKE), then you need to update it_

```
yum -y update kubectl
```

I am running Kubernetes 1.16.11 so I don't need to update it.

I need to authorize `gcloud` to access Google Cloud

```
gcloud auth login
```

Being a dev enviroment I preffer to install gcloud beta as well to try alpha or beta Connect features

```
 gcloud components install beta
```

List the projects

```
$ gcloud projects list
PROJECT_ID            NAME              PROJECT_NUMBER
dotted-ranger-283911  My First Project  482168856288
trusty-wares-283912   Contrail          234378606342
```

Next, I need to grant the required IAM roles to the user registering the cluster

```
PROJECT_ID=trusty-wares-283912
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member user:[GCP_EMAIL_ADDRESS] \
 --role=roles/gkehub.admin \
 --role=roles/iam.serviceAccountAdmin \
 --role=roles/iam.serviceAccountKeyAdmin \
 --role=roles/resourcemanager.projectIamAdmin
```

I enabled the APIs required for my project

```
gcloud services enable \
 --project=${PROJECT_ID} \
 container.googleapis.com \
 gkeconnect.googleapis.com \
 gkehub.googleapis.com \
 cloudresourcemanager.googleapis.com \
 anthos.googleapis.com
```

A JSON file containing Google Cloud Service Account credentials is required to manually register a cluster. Create a service account by running the following command:

```
SERVICE_ACCOUNT_NAME=contrail-cluster-1
$ gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME} --project=${PROJECT_ID}
```

Bind the gkehub.connect IAM role to the service account:

```
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/gkehub.connect"
```

Download the service account's private key JSON file. You use this file in the next section:
```
$ gcloud iam service-accounts keys create /tmp/creds/contrail-cluster-1-trusty-wares-283912.json \
  --iam-account=${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
  --project=${PROJECT_ID}
```

Grant the cluster-admin RBAC role to the user registering the cluster

```
kubectl auth can-i '*' '*' --all-namespaces
```

To register a non-GKE cluster we need to run the following command

```
MEMBERSHIP_NAME=contrail-cluster-1

gcloud container hub memberships register ${MEMBERSHIP_NAME} \
   --project=${PROJECT_ID} \
   --context=$(kubectl config current-context) \
   --kubeconfig=$HOME/.kube/config \
   --service-account-key-file=/tmp/creds/contrail-cluster-1-trusty-wares-283912.json
```


When the command finishes a new pod called gke-connect-agent will run in the cluster. This is responsabile to communication with GKE Hub as I decribed above.
```
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

I can view cluster registration status and all the clusters within my Google project

```
$ gcloud container hub memberships list
NAME                EXTERNAL_ID
cluster-1           1aac8a94-9f25-4559-bdc6-a663f25417c2
contrail-cluster-1  da221221-0f05-491c-8fe2-2eb4452a593d
```

To login to the cluster from the Cloud Console I will use a bearer token. For this I create a Kubernetes service account (KSA) in the cluster.

I am creating and applying first a  node-reader role-based access control (RBAC) role

```
cat <<EOF > node-reader.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
EOF
kubectl apply -f node-reader.yaml
```

Create and authorise a KSA.

```
KSA_NAME=contrail-cluster-1-sa
kubectl create serviceaccount ${KSA_NAME}
kubectl create clusterrolebinding ksa-view --clusterrole view --serviceaccount default:${KSA_NAME}
kubectl create clusterrolebinding ksa-node-reader --clusterrole node-reader --serviceaccount default:${KSA_NAME}
kubectl create clusterrolebinding binding-account --clusterrole cluster-admin --serviceaccount default:${KSA_NAME}
```

To acquire the KSA's bearer token, run the following command:

```
SECRET_NAME=$(kubectl get serviceaccount ${KSA_NAME} -o jsonpath='{$.secrets[0].name}')
kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
```

The output token use it in Cloud Console to Login to the cluster

The clusters should be visibile in Anthos

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image4.png)

I can view details about it in Kubernetes Engine tab

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image5.png)

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image6.png)

### Deploy Anthos Apps from GCP Marketplace into Kubernetes on-prem cluster

Anthos expects a namespace called `application-system` which will run the agent to install the apps from the GCP Marketplace.

We need to create at least two namespaces and enable them to pull the container images from the Google Container Registry (GCR) associated with the Marketplace.

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

Now we need to create a secret with the contents of the token

```
$ kubectl create secret docker-registry gcr-json-key \
--docker-server=https://marketplace.gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ./gcr-sa.json)" \
--docker-email=user@email.com
```

We need to patch the default service account within the namespace to use the secret to pull images from GCR instead of Docker Hub

```
$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'
```

Finally, we'll annotate the `application-system` namespace to enable the deployment of Kubernetes Apps from GCP Marketplace

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


Next, we need to create and configure a namespace for the app we will deploy it from GCP Marketplace

```
$ kubectl create ns pgsql

$ kubens pgsql

$ kubectl create secret docker-registry gcr-json-key \
 --docker-server=https://gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ./gcr-sa.json)" \
--docker-email=user@email.com
```

Docker-server key is pointing to https://gcr.io which holds the container images for the GCP Marketplace Apps

Similar to the other namespace, we need to patch the default service account within the pgsql namespace to use the secret to pull images from GCR instead of Docker Hub

```
$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'
```

Also annotate the pgsql namespace to enable the deployment of Kubernetes Apps from GCP Marketplace

```
$ kubectl annotate namespace pgsql marketplace.cloud.google.com/imagePullSecret=gcr-json-key
```

All these steps were a preparation to deploy an app from GCP Marketplace on the on-prem K8s with contrail ClusterRole

Choose PostgresSQl Server from GCP Marketplace

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image7.png)

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
