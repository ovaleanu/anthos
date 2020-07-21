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
# kubectl get po -n kube-system
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
