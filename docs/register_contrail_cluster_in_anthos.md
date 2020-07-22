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

In this case I installed Kubernetes 1.18.5 on BMS with Ubuntu 18.04 OS.

```
# kubectl get nodes -o wide
NAME                      STATUS   ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
r9-ru24.csa.juniper.net   Ready    worker   19d   v1.18.5   192.168.213.114   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://19.3.11
r9-ru25.csa.juniper.net   Ready    worker   20d   v1.18.5   192.168.213.113   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://19.3.11
r9-ru26.csa.juniper.net   Ready    master   20d   v1.18.5   192.168.213.112   <none>        Ubuntu 18.04.4 LTS   4.15.0-108-generic   docker://19.3.11
```

After the Kubernetes cluster is deployed I install Contrail using single yaml file.
```
$ kubectl get po -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
config-zookeeper-st8gj                            1/1     Running   0          19d
contrail-agent-n2kqz                              3/3     Running   0          19d
contrail-agent-pj25z                              3/3     Running   0          19d
contrail-analytics-alarm-w9m4l                    4/4     Running   0          19d
contrail-analytics-cwkgk                          4/4     Running   0          19d
contrail-analytics-snmp-mf6kq                     4/4     Running   0          19d
contrail-analyticsdb-zlp4w                        4/4     Running   0          19d
contrail-configdb-6m5ht                           3/3     Running   0          19d
contrail-controller-config-2t948                  6/6     Running   0          19d
contrail-controller-control-vwczl                 5/5     Running   0          19d
contrail-controller-webui-8d6mz                   2/2     Running   0          19d
contrail-kube-manager-bqtrw                       1/1     Running   0          19d
coredns-66bff467f8-dh8n2                          1/1     Running   0          19d
coredns-66bff467f8-fff52                          1/1     Running   0          20d
etcd-r9-ru26.csa.juniper.net                      1/1     Running   0          20d
kube-apiserver-r9-ru26.csa.juniper.net            1/1     Running   0          20d
kube-controller-manager-r9-ru26.csa.juniper.net   1/1     Running   0          20d
kube-proxy-5q6xd                                  1/1     Running   0          20d
kube-proxy-l4fnj                                  1/1     Running   0          19d
kube-proxy-n7l6p                                  1/1     Running   0          20d
kube-scheduler-r9-ru26.csa.juniper.net            1/1     Running   0          20d
rabbitmq-x7cnk                                    1/1     Running   0          19d
redis-22bb7                                       1/1     Running   0          19d
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

I am running Kubernetes 1.18.5 so I don't need to update it.

I need to authorize `gcloud` to access Google Cloud

```
gcloud auth login
```

Being a dev enviroment I preffer to install gcloud beta as well to try alpha or beta Connect features

```
 gcloud components install beta
```

Next, I need to grant the required IAM roles to the user registering the cluster

```
$ gcloud projects add-iam-policy-binding [PROJECT_ID] \
 --member user:[GCP_EMAIL_ADDRESS] \
 --role=roles/gkehub.admin \
 --role=roles/iam.serviceAccountAdmin \
 --role=roles/iam.serviceAccountKeyAdmin \
 --role=roles/resourcemanager.projectIamAdmin
```

`[PROJECT_ID]` I can find the value from GKE Console or using `gcloud` cli command

```
$ gcloud projects list
PROJECT_ID            NAME              PROJECT_NUMBER
dotted-ranger-283911  My First Project  482168856288
trusty-wares-283912   Contrail          234378606342
```
In my case is `trusty-wares-283912`.

I enabled the APIs required for my project

```
gcloud services enable \
 --project=[PROJECT_ID] \
 container.googleapis.com \
 gkeconnect.googleapis.com \
 gkehub.googleapis.com \
 cloudresourcemanager.googleapis.com \
 anthos.googleapis.com
```

A JSON file containing Google Cloud Service Account credentials is required to manually register a cluster. Create a service account by running the following command:

```
$ gcloud iam service-accounts create [SERVICE_ACCOUNT_NAME] --project=[PROJECT_ID]
```

In my case `[SERVICE_ACCOUNT_NAME]` is contrail. You can choose any name.

Bind the gkehub.connect IAM role to the service account:

```
$ gcloud projects add-iam-policy-binding [PROJECT_ID] \
 --member="serviceAccount:[SERVICE_ACCOUNT_NAME]@[PROJECT_ID].iam.gserviceaccount.com" \
 --role="roles/gkehub.connect"
```

Download the service account's private key JSON file. You use this file in the next section:
```
$ gcloud iam service-accounts keys create [LOCAL_KEY_PATH] \
  --iam-account=[SERVICE_ACCOUNT_NAME]@[PROJECT_ID].iam.gserviceaccount.com \
  --project=[PROJECT_ID]
```
The `[LOCAL_KEY_PATH]` is saved in `/tmp/creds/contrail-trusty-wares-283912.json`

Grant the cluster-admin RBAC role to the user registering the cluster

```
kubectl auth can-i '*' '*' --all-namespaces
```

To register a non-GKE cluster I nned to run the following command

```
gcloud container hub memberships register [MEMBERSHIP_NAME] \
   --project=[PROJECT_ID] \
   --context=[KUBECONFIG_CONTEXT] \
   --kubeconfig=[KUBECONFIG_PATH] \
   --service-account-key-file=SERVICE_ACCOUNT_KEY_PATH
```

In my case, `[MEMBERSHIP_NAME]` is `contrail-cluster` (chosen by me), '[KUBECONFIG_CONTEXT]' is the output for `kubectl config current-context` and `[SERVICE_ACCOUNT_KEY_PATH]` is `/tmp/creds/contrail-trusty-wares-283912.json`.

When the command finishes a new pod called gke-connect-agent will run in the cluster. This is responsabile to communication with GKE Hub as I decribed above.
```
$ kubectl get pods -A
NAMESPACE     NAME                                                READY   STATUS    RESTARTS   AGE
gke-connect   gke-connect-agent-20200710-02-00-59477469f8-lhngc   1/1     Running   0          30h
kube-system   config-zookeeper-st8gj                              1/1     Running   0          21d
kube-system   contrail-agent-n2kqz                                3/3     Running   0          21d
kube-system   contrail-agent-pj25z                                3/3     Running   0          21d
kube-system   contrail-analytics-alarm-w9m4l                      4/4     Running   0          21d
kube-system   contrail-analytics-cwkgk                            4/4     Running   0          21d
kube-system   contrail-analytics-snmp-mf6kq                       4/4     Running   0          21d
kube-system   contrail-analyticsdb-zlp4w                          4/4     Running   0          21d
kube-system   contrail-configdb-6m5ht                             3/3     Running   0          21d
kube-system   contrail-controller-config-2t948                    6/6     Running   0          21d
kube-system   contrail-controller-control-vwczl                   5/5     Running   0          21d
kube-system   contrail-controller-webui-8d6mz                     2/2     Running   0          21d
kube-system   contrail-kube-manager-bqtrw                         1/1     Running   0          21d
kube-system   coredns-66bff467f8-dh8n2                            1/1     Running   0          21d
kube-system   coredns-66bff467f8-fff52                            1/1     Running   0          21d
kube-system   etcd-r9-ru26.csa.juniper.net                        1/1     Running   0          21d
kube-system   kube-apiserver-r9-ru26.csa.juniper.net              1/1     Running   0          21d
kube-system   kube-controller-manager-r9-ru26.csa.juniper.net     1/1     Running   0          21d
kube-system   kube-proxy-5q6xd                                    1/1     Running   0          21d
kube-system   kube-proxy-l4fnj                                    1/1     Running   0          21d
kube-system   kube-proxy-n7l6p                                    1/1     Running   0          21d
kube-system   kube-scheduler-r9-ru26.csa.juniper.net              1/1     Running   0          21d
kube-system   rabbitmq-x7cnk                                      1/1     Running   0          21d
kube-system   redis-22bb7                                         1/1     Running   0          21d
```

_Note: I need SNAT enabled in Contrail to allow gke-connect-agent communication to internet_

I can view cluster registration status and all the clusters within my Google project

```
$ gcloud container hub memberships list
NAME                EXTERNAL_ID
cluster-1           1aac8a94-9f25-4559-bdc6-a663f25417c2
contrail-cluster    2f7d6521-cf8f-4870-a379-50207ecd579b
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

Create and authorise a KSA. I will use `cluster-admin` role for now.

```
KSA_NAME=[KSA_NAME]
kubectl create serviceaccount ${KSA_NAME}
kubectl create clusterrolebinding [BINDING_NAME] \
--clusterrole cluster-admin --serviceaccount default:[KSA_NAME]
```

In my case, `[KSA_NAME]` is `contrail-sa` and `[BINDING_NAME]` is `contrail-admin`.
Both name chosen by me.

To acquire the KSA's bearer token, run the following command:

```
SECRET_NAME=$(kubectl get serviceaccount [KSA_NAME] -o jsonpath='{$.secrets[0].name}')
kubectl get secret ${SECRET_NAME} -o jsonpath='{$.data.token}' | base64 --decode
```

The output token use it in Cloud Console to Login to the cluster

The clusters should be visibile in anthos

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image4.png)

I can view details about it in Kubernetes Engine tab

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image5.png)

![](https://github.com/ovaleanujnpr/anthos/blob/master/images/image6.png)
