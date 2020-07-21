## Integrate a Kubernetes cluster with Contrail in Google Anthos

Anthos is a portfolio of products and services for hybrid cloud and workload management that runs on the Google Kubernetes Engine (GKE) and users can manage workloads running also on third-party clouds like AWS, Azure and on-premises (private) clusters.
The scope of this document is integrate an on-prem K8s cluster with Contrail with Anthos.

Fun fact Anthos is flower in Greek. The reason they chose that os because flowers grow on premise, but they need rain from the cloud to flourish.

Google Cloud Platform console provides a single control plane for managing Kubernetes clusters deployed in multiple locations.

Deploy a GKE cluster and then register it to Anthos

[GKE quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)

[Explore Anthos](https://cloud.google.com/anthos/docs/tutorials/explore-anthos)


![](/images/2020/07/image1.png)


Install a Kubernetes cluster with Contrail on any VM/BMS on-prem. Check this [Wiki](https://github.com/ovaleanujnpr/Kubernetes/wiki/Installing-Kubernetes-with-Contrail) for deployment.
