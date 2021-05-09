# Configuring SAML with a hot-worm-cold architecture using ECK

This article will guide to setup a SAML with auth0 as idp & Kibana as sp using a hot/warm/cold artechicture with ECK, the purpose of this project is to use production features like: **dedicated nodes, zone awareness, pod & node affinity, podDisruption, dedicated storage class**. If you interested in know more about it in deep you should check the [ECK](https://github.com/framsouza/eck) article. Also you should be familiar with how [SAML](https://www.elastic.co/guide/en/elasticsearch/reference/7.12/saml-guide-stack.html) works.

### Scenario
On this scenario let's assume you want to deploy ECK to handle application logging and you want to use a centralized way to authenticate into Kibana using auth0. As logging data (time-series) has a predictable time of life, we can use hot, warm & cold architecture to have a better way to deal with your data over time.

### Architecture 

We are using GKE to setup this environment, with the following configurations:

*3 Nodes pools
1 Hot Node Pool wtih 6 Kubernetes instances running spread across 3 availabilty zones;;
1 Warm Node Pool with Kubernetes instances running spread across 3 availabilty zones;;
1 Cold Node Pool with Kubernetes instances running spread across 3 availabilty zones;;

*Instances configuration
- hot nodes: **c2-standard-4    (4 vCPUs, 16 GB memory, 100GB disk)**
- warm nodes: **e2-standard-2   (2 vCPUs, 8 GB memory, 100GB disk)**
- cold nodes: **e2-standard-2   (2 vCPUs, 8 GB memory, 100GB disk)**

*9 Elasticsearch instances (Split by hot & warm, cold and dedicated master)* plust 1 Kibana instance;
- 3 Elasticsearch hot data node (50Gi SSD disk, 4Gi JVM, 8Gi memory)
- 3 Elasticsearch warm data node (100Gi disk, 2Gi JVM, 4Gi memory)
- 3 Elasticsearch cold data node (100Gi disk, 2Gi JVM, 4Gi memory)
- 3 Elasticsearch master node (10Gi disk, 1Gi JVM, 2Gi memory)

_GKE zones_: europe-west1-b, europe-west1-c, europe-west1-d 

For each node pool we are using an especific Kubernetes label to be able to attach the Elasticsearch instance into the right hard configuration. The label name is called "type" and the values are hot, warm or cold. Remember, if you want to update the Kubernetes node pool label, you must use the (command line)[https://cloud.google.com/kubernetes-engine/docs/how-to/update-existing-nodepools#updating_node_labels], this is not possible via Console UI.

### Overview

Hot-Warm-Cold architecture is the powerful way Elasticsearch uses to separe hot nodes (the ones that handle hot data/events) and warm & cold nodes (the ones that handle "no frenquency" data). These architecture are common for time series data like logging and metrics. For instance, logs from today are actively being indexed and this week's logs are most searched. Last week you may want to search but not as much as the current current week's logs. Last month's logs may or may not be searched, then you keep them around just in case stored in the cold nodes.

We're using three node pools, each one to handle the specific Elasticsearch role (hot, warm & code) with differents hardware settings.

Using separed node pools means we can isolate the Kubernetes nodes to run only Elasticsearch workloads and it can be done through Kubernetes labels. You don't need to spin up one Kubernetes cluster only to run Elasticsearch, you can use node pools to run & isolate Elasticsearch from your application, it's helpful to segregate resource utilisation. 

As the purpose of this guide is configure a production environment, we are going to configure also a SAML authentication with auth0 as idP.

Remember, to use SAML you should use the enterprise license.

### First things first / Operator setup

# Install ECK

First you need to setup google cloud RBAC running the following command:

```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
```

Now, you're able to deploy the operator:

```
kubectl apply -f https://download.elastic.co/downloads/eck/1.5.0/all-in-one.yaml
```

### Configuring StorageClass
With StorageClass you can describe the "classes" of storage you want to use. Differents classes might map to quality of service levels, backups purpose or any arbitrary policy determined by the administrator. 
As we are using differents zones and differents hard configuration due to the fact we are using hot/warm/cold architecture, we need to specify a StorageClass for each node and associate it with the proper PersistentVolume.

Here an exemple of StorageClass that will be attached to the PersistentVolume in the hot nodes:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hot-europe-west1-b
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - europe-west1-b
```

Some key points:

*provisioner: kubernetes.io/gce-pd* : it means we are using gce as provider;
*type: pd-ssd*: - it will provide ssd disks for the volumes attached to this StorageClass;
*reclaimPolicy: Delete* - it will deletes PersistentVolumeClaim resources if the owning Elasticsearch nodes are scaled down or a deletion;
*volumeBindingMode: WaitForFirstConsumer* - It will prevent the pod be scheduled, because of affinity settings, on a host where the bound PersistentVolume is not available, *this is a very crucial configuration*;
*allowedTopologies*: - it will bind your StorageClass with the zone labels (you can use any kind of label)


##Create SAML configmap
Before create the Elasticsearch manifest, we need to create the confimap which content the metadata file of the idp. Remeber that, some idP offer the metadata via URL, on this scenario i will a configmap example to demonstrate how mount it in your deployment. First, create the confimap:

```kubectl create configmap saml-metadata --from-file=../framsouza_eu_auth0_com-metadata.xml```

## Create Elasticsearch & Kibana resource
Once you have the saml configmap, you can apply the Elasticsearch & Kibana manifest

```
kubectl create -f eck-saml-hot-warm-cold.yml && kubectl create -f kibana.yml
```

It may take a while untill all the resources get ready, but after some minutes you should see something like this:
```
sh-3.2# kubectl get pods
NAME                             READY   STATUS     RESTARTS   AGE
elastic-prd-es-cold-zone-b-0     1/1     Running    0          35m
elastic-prd-es-cold-zone-c-0     1/1     Running    0          35m
elastic-prd-es-cold-zone-d-0     1/1     Running    0          35m
elastic-prd-es-hot-zone-b-0      1/1     Running    0          35m
elastic-prd-es-hot-zone-c-0      1/1     Running    0          35m
elastic-prd-es-hot-zone-d-0      1/1     Running    0          35m
elastic-prd-es-master-zone-b-0   1/1     Running    0          35m
elastic-prd-es-master-zone-c-0   1/1     Running    0          35m
elastic-prd-es-master-zone-d-0   1/1     Running    0          35m
elastic-prd-es-warm-zone-b-0     1/1     Running   0          16m
elastic-prd-es-warm-zone-c-0     1/1     Running    0          16m
elastic-prd-es-warm-zone-d-0     1/1     Running    0          15m
kibana-prd-kb-7467b79f54-btzhq   1/1     Running    0          4m8s
```

## Test connection
At this point you can test the connection via API or exposing Kibana service,
Grab the elastic password
```
kubectl get secret elastic-prd-es-elastic-user -o yaml
```

and the ES service name and run the following:

```
curl -k https://elastic:es-password@es-service-name8:9200/_cluster/health?pretty
```

You can also expose the Kibana service and try to access it via browser:
```
kubectl port-forward svc/kibana-prd-kb-http 5601
```

### Ingress controller

Deploy the ingress resource:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/cloud/deploy.yaml
```

Before deploy the controller, we need create the ssl certificate, 
```
kubectl create secret tls framsouza-cert --key framsouza_co_key.txt --cert framsouza_co.crt
```

Once the pods are running, you can deploy the ingress controller manifest:
```
kubectl create -f ingress.yml
```
