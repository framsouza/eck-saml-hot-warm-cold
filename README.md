# eck-hot-warm-architecture

This article will guide to setup a hot-warm artechicture on top of Kubernetes using production features like: **dedicated nodes, zone awareness, pod & node affinity** and so an. If you interested in know more about it in deep you should check the [ECK](https://github.com/framsouza/eck) article.

### Scenario
On this scenario we're running ECK to store Kubernetes metrics collected by metricbeats as a DaemonSet.

### Before Start
If you wanna run this example without make any change on the manifest, please make sure you're using the following configuration:

-   2 GKE node pools
	- 1. Using a Kubernetes label called  **type**  :  **hot**,
	- 2. Using a Kubernetes label called  **type**  :  **warm**
-   GKE Cluster running on  **europe-west1**  region

### Install ECK

First you need to setup google cloud RBAC running the following command:

```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
```

Now, you're able to deploy the Opereator:

```
kubectl apply -f https://download.elastic.co/downloads/eck/1.3.0/all-in-one.yaml
```

### Architecture 

*6 Kubernetes Nodes (Separeted by node pools & availability three zones)*
- 3 GKE Instance type (hot): **e2-standard-4	 (4 vCPUs, 16 GB memory)**
- 3 GKE Instance type (warm): **e2-standard-2   (2 vCPUs, 8 GB memory)**

*9 Elasticsearch instances (Slip by hot & warm and dedicated master)*
- 3 Elasticsearch hot data node (50Gi SSD disk, 4Gi JVM, 8Gi memory)
- 3 Elasticsearch warm data node (100Gi disk, 2Gi JVM, 4Gi memory)
- 3 Elasticsearch master node (10Gi disk, 2Gi JVM, 4Gi memory)

![ECK Architecture](img/architecture.png)

### Overview

Hot-Warm architecture is the powerful way Elasticsearch use to separe hot nodes (the ones that handle hot data/events) and warm nodes (the ones that handle "no frenquency" data) and it's ideal for storing longer retention periods.

On this scenario, we're using Metricbeat as a [Daemonset](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) to collect metrics of our Kubernetes nodes.

Metricbeat sends events collected to ECK and it's indexed on the hot nodes, after a period of 7 days (In our exemple) these events are move to warm nodes and deleted after 30 days.

We're using two node pools, one to handle/execute Elasticsearch hot nodes and one to handle/execute Elasticsearch warn nodes. 
Using separed node pools means we can isolate the Kubernetes nodes to run only Elasticsearch workloads (In our case also spared by hot/warm) and it can be done through Kubernetes labels. You don't need to spin up one Kubernetes cluster only to run Elasticsearch, you can use node pools to run & isolate Elasticsearch from your application, it's helpful to segregate resource utilisation. 

Create SAML configmap
```kubectl create configmap saml-metadata --from-file=../Downloads/framsouza_eu_auth0_com-metadata.xml```


### Custom Template

```
PUT _template/my-metricbeat
{
    "order" : 800,
    "index_patterns" : [
      "my-metric*"
    ],
    "settings" : {
      "number_of_shards" : 3,
      "index.routing.allocation.require.type": "hot",
      "index.lifecycle.name": "my-metricbeat"
}
}
```

## ILM Policy

```
PUT _ilm/policy/my-metricbeat
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "30d",
            "max_docs": 100
          }
        }
      },
      "warm": {
        "actions": {
          "allocate": {
            "number_of_replicas": 2,
            "include": {},
            "exclude": {},
            "require": {
              "type": "warm"
            }
          }
        }
      }
    }
  }
}
```

## Admin SAML ES role

```
{
  "admin" : {
    "cluster" : [
      "all"
    ],
    "indices" : [ ],
    "applications" : [
      {
        "application" : "kibana-.kibana",
        "privileges" : [
          "all"
        ],
        "resources" : [
          "*"
        ]
      }
    ],
    "run_as" : [ ],
    "metadata" : { },
    "transient_metadata" : {
      "enabled" : true
    }
  }
}
```

## Tricks

Remember to keep at the least one node with the content rule enable if you are going to use Kibana, otherwise you will end up with the following error during Kibana startup:
```    {
      "node_id" : "N9AQzsWjSsKvOb3541LUmg",
      "node_name" : "elastic-prod-es-hot-zone-d-0",
      "transport_address" : "10.92.12.6:9300",
      "node_attributes" : {
        "k8s_node_name" : "gke-eck-hot-warm-cold-archit-hot-pool-33499f95-36pk",
        "xpack.installed" : "true",
        "type" : "hot",
        "zone" : "europe-west1-d",
        "transform.node" : "false"
      },
      "node_decision" : "no",
      "weight_ranking" : 8,
      "deciders" : [
        {
          "decider" : "data_tier",
          "decision" : "NO",
          "explanation" : "index has a preference for tiers [data_content], but no nodes for any of those tiers are available in the cluster"
        }
      ]```
