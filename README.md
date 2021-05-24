This document explains how to set up SAML with auth0 as idp using a hot/warm/cold artechicture with ECK. The purpose of this project is to use production features such as dedicated nodes, zone awareness, pod and node affinity, podDisruption and dedicated storage class. To know more, check this [ECK](https://github.com/framsouza/eck) article. Also, you should be familiar with [SAML](https://www.elastic.co/guide/en/elasticsearch/reference/7.12/saml-guide-stack.html).


### **Scenario**

In this scenario we deploy ECK to handle application logging along with a centralized way to authenticate into Kibana using auth0. 

ECK is our Elastic Cloud on Kubernetes that automates the deployment, provisioning, management and setup of Elaticsearch, APM, Kibana, Beats and Enterprise Search. It’s an easy way to get Elastic Stack up and running on top of Kubernetes.

As logging and metric data (time-series) has a predictable time of life, we can use hot, warm and cold architecture to better deal with your data over time.

We will deploy the following resources:



*   ECK Operator
*   StorageClass
*   Elasticsearch
*   Elasticsearch role Job
*   Kibana
*   Ingress Controller
*   ConfigMap SAML metadata


### **Architecture**

To set up this environment, we use [GKE](https://cloud.google.com/kubernetes-engine) (Google Kubernetes Engine) with the following configurations:

_3 Kubernetes Nodes pools_



*   1 Hot Node Pool with 6 Kubernetes instances running spread across 3 availability zones
*   1 Warm Node Pool with Kubernetes instances running spread across 3 availability zones
*   1 Cold Node Pool with Kubernetes instances running spread across 3 availability zones

_Instances configuration_



*   Hot nodes: c2-standard-4 (4 vCPUs, 16 GB memory)
*   Warm nodes: e2-standard-2 (2 vCPUs, 8 GB memory)
*   Cold nodes: e2-standard-2 (2 vCPUs, 8 GB memory)

_9 Elasticsearch instances total (with individual tiers for hot, warm, and cold data, as well as dedicated masters)_ plus 2 Kibana instances running on GKE zones _europe-west1-b_, _europe-west1-c_ and _europe-west1-d._

For each GKE node pool, we use a specific Kubernetes label to attach the Elasticsearch instance into the right hardware configuration. The label name is called "type" and the values are: hot, warm or cold. You can change and add the label later by using the [command line](https://cloud.google.com/kubernetes-engine/docs/how-to/update-existing-nodepools#updating_node_labels).

This document contains two parts:



1. Manifest explanation
2. How to set up


# **Manifest explanation**

The Elasticsearch manifest file [eck-saml-hot-warm-cold.yml](https://github.com/framsouza/eck-saml-hot-warm-cold/blob/main/eck-saml-hot-warm-cold.yml) contains the Elasticsearch settings and we will examine the relevant parts of the manifest one by one.


### **nodeSets:**


```
 nodeSets:
  - name: hot-zone-b
    count: 1
    config:
      node.attr.zone: europe-west1-b
      cluster.routing.allocation.awareness.attributes: k8s_node_name,zone
      node.roles: [ data_hot, data_content ]
      node.store.allow_mmap: false
```


We have _one_ node called _hot-zone-b_ deployed at zone _europe-west1-b_. Then, we define two routing awareness attributes:



*   _k8s_node_name_: Makes sure that Elasticsearch allocates primary and replica shards to pods running on different Kubernetes nodes and never to pods that are scheduled onto a single Kubernetes node.
*   _zone:_ Uses the Kubernetes label called _domain.beta.kubernetes.io/zone. _If Elasticsearch knows which nodes are on the same zone it can distribute the primary shard and its replica shards to minimise the risk of losing all shard copies in the event of a failure.

We explicitly say that this node will have a _data_hot_ and _data_content_ [role](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#modules-node). 

_node.store.allow_mmap : false_ will prevent the virtual address space (on Linux distribution) to run into errors or exceptions because this default configuration is too low. You can find more information about it [here](https://www.elastic.co/guide/en/cloud-on-k8s/1.5/k8s-virtual-memory.html#k8s-virtual-memory).


### **xpack.security**


```
     xpack:
        security:
          authc:
            realms:
              saml:
                saml1:
                  attributes.principal: nameid
                  idp.entity_id: urn:framsouza.eu.auth0.com
                  idp.metadata.path: /usr/share/elasticsearch/config/framsouza_eu_auth0_com-metadata.xml
                  order: 2
                  sp.acs: https://framsouza.co/api/security/v1/saml
                  sp.entity_id: https://framsouza.co
                  sp.logout: https://framsouza.co/logout
```


This is the SAML configuration. You should follow this [guide](https://auth0.com/docs/protocols/saml-protocol/configure-auth0-as-saml-identity-provider#configure-auth0-as-idp) to configure auth0 as idp and then get the metadata file and the idp.* settings.



*   _attributes.principal:_ Defines which SAML attribute is going to be mapped to the principal (username) of the authenticated user in Kibana
*   _idp.entity_id:_ Is the SAML EntityID of your Identity Provider (you can get it from SAML metadata file)
*   _idp.metadata.path:_ Is the file path or the HTTPs URL where your Identity Provider metadata is available. In this example,, we use a file to demonstrate how to mount a volume
*   _sp.acs:_ Is the Assertion Consumer Service URL where Kibana is listening for incoming SAML messages.
*   _sp.entity_id:_ Is the SAML EntityID of our Service Provider.
*   _sp.logout:_ Is the SingleLogout endpoint where the Service Provider is listening for incoming SAML LogoutResponse and LogoutRequest messages

Keep in mind the sp.* configurations must point to Kibana endpoint and the idp.* settings you must collect from your Identity Provider. Also, the SAML metadata is stored in a ConfigMap and mounted as a Volume inside Elasticsearch. The idp also may provide the metadata via HTTP, in this case, auth0 provides the metadata file via HTTP but the purpose is to show to you how to mount configMap (or secret) as a volume into Elasticsearch pods.


### **volumeClaimTemplates**


```
   volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 50Gi
        storageClassName: sc-zone-b
```


The hot node will have a 50Gi available of disk which refers to the storage class called _sc-zone-b (we will define the storageclass later)_, this is the space available to store elasticsearch data.


### **Node and Pod affinity**


```
   podTemplate:
      spec:
        nodeSelector:
          type: hot
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: failure-domain.beta.kubernetes.io/zone
                  operator: In
                  values:
                  - "europe-west1-b"
```


The affinity feature restricts scheduling pods in a group of Kubernetes nodes based on labels. Node affinity is conceptually similar to nodeSelector that defines which node the pod will be scheduled based on the label. _nodeAffinity_ greatly extends the types of constraints you can express using enhancements labels. 

nodeAffinity is using requiredDuringSchedulingIgnoredDuringExecution affinity type. That means, a hard requirement that rules _must_ be met for a pod to be scheduled on a node. In this example, we're defining the following: "only run the pod on nodes in the zone europe-west1-b". The _nodeSelector_ will be matched with a Label on the Kubernetes node, in this case _type: hot_.


### **Containers definition**


```
       containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 8Gi
              cpu: 2
            limits:
              memory: 8Gi
```


In this example, this node has 8Gi of memory and is requesting at least 2 cpu cores. New Elasticsearch version (from 7.11) automatically sets the heap size based on the node roles and available memory, there are more information about this new approach [here](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-managing-compute-resources.html#k8s-compute-resources-elasticsearch). Here we will have 4Gi of heap.


### **Creating a configMap**

_You must create this file before the ES manifest_

You get the metadata file from your idp (in this case, [auth0](https://auth0.com/docs/protocols/saml-protocol/configure-auth0-as-saml-identity-provider#configure-auth0-as-idp)). To put the content inside a configMap you can run the following:


```
kubectl create configmap saml-metadata --from-file=../framsouza_eu_auth0_com-metadata.xml
```


Remember to adjust the file location.


### **Volume SAML metadata**


```
         volumeMounts:
          - name: saml-metadata
            mountPath:  /usr/share/elasticsearch/config/framsouza_eu_auth0_com-metadata.xml
        volumes:
        - name: saml-metadata
          configMap:
            name: saml-metadata
```


The _volumeMounts_ session is where the volume should be mounted inside the pod. In this example, you are mounting a volume _saml-metadata_ and the file is located in _ES_CONFIG_ directory _/usr/share/elasticsearch/config/framsouza_eu_auth0_com-metadata.xml_. 

Then, _volumes_ means the volume must be mounted according to a configMap called _saml-metadata_.


### **readinessProbe**


```
         readinessProbe:
            exec:
              command:
              - bash
              - -c
              - /mnt/elastic-internal/scripts/readiness-probe-script.sh
            failureThreshold: 3
            initialDelaySeconds: 10
            periodSeconds: 12
            successThreshold: 1
            timeoutSeconds: 12
          env:
          - name: READINESS_PROBE_TIMEOUT
            value: "10"
```


Readiness probe is used to know when a container is ready to start accepting traffic. By default the timeout is 3 seconds, this is acceptable in most cases but if it’s under very heavy load you might need to increase the timeout. This example we are increasing the timeout to 10 seconds and adjusting the check time to 12 seconds. 


### **initContainer**


```
       initContainers:
        - command:
          - sh
          - -c
          - |
            bin/elasticsearch-plugin install --batch repository-gcs
          name: install-plugins
```


To install or perform any task at the operational system level before Elasticsearch starts, you should use an initContainer. In this example, we are installing the GCS repository where we can send snapshots.

You can also build your own image and include the GCS plugin, to do so you can follow this [doc](https://www.elastic.co/blog/elasticsearch-docker-plugin-management). This approach is more production ready, so if you decide to go for it, you don’t need to specify the _install-plugins _initContainer.

These are the sessions from one node. The rest of the node configuration is basically the same, except for the name, zone, and storageClass name. At the end of the file, you see the following setting (which applies to the whole cluster):


```
 updateStrategy:
    changeBudget:
      maxSurge: 1
      maxUnavailable: 1
  podDisruptionBudget:
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          elasticsearch.k8s.elastic.co/cluster-name: elastic-prd
```


_updateStrategy_ controls the number of simultaneous changes in the Elasticsearch cluster. maxSurge: 1 means only one new Pod is created at a time. After the first new Pod is Ready, an old Pod is killed and the second new Pod is created. 

If you don’t specify the strategy you want, the default behaviour is maxSurge: -1 which means that all the required pods are created immediately, it may cause issues if you don’t have enough resources to deal with it. You can find more information [here](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-update-strategy.html#k8s-update-strategy).

While maxSurge determines how many new Pods to create, maxUnavailable determines how many old Pods to kill. In this case, we can only kill one old Pod at a time. This ensures the capacity is always at least 3 - 1 Pods. If this is not enough for your environment configuration, you can disable the podDisruption and configure your own.

_podDisruptionBudget_ determines how many nodes must continue running in case of disruption, for instance hardware failures, pod eviction due to out of resource, draining node and so on.


# **How to set up**


### **Install ECK**

First let’s give your google account administrator privileges on the cluster by setting up the RBAC by running the following command:


```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")

```



1. Install the ECK operator, keep in mind we are installing ECK operator version 1.5.0 but you may want to check for updated versions at our [website](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html#k8s-deploy-eck).


```
kubectl apply -f https://download.elastic.co/downloads/eck/1.5.0/all-in-one.yaml
```


This creates the ECK Operator pod and cluster permissions in the namespace _elastic-system_, you can monitor it by running:


```
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```



### **Configuring StorageClass**

With StorageClass you can describe the "classes" of storage you want to use. Different classes might map to quality of service levels, disk types, backup purpose or any arbitrary policy determined by the administrator.

As we are using hot/warm/cold architecture, we need to specify a StorageClass for the hot and warm/cold nodes (as they must use different disk types) and associate it with the proper PersistentVolume.

This is an example of StorageClass that is attached to the PersistentVolume in the hot nodes:


```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-hot
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```


Some key points:



*   _provisioner: kubernetes.io/gce-pd_ : It means we are using GCE as provider;
*   _type: pd-ssd_: - It provides ssd disks for the volumes attached to this StorageClass;
*   _reclaimPolicy: Delete_ - It deletes PersistentVolumeClaim resources if the owning Elasticsearch nodes are scaled down or a deletion;
*   _volumeBindingMode: WaitForFirstConsumer_ - It prevents the pod from being scheduled, because of affinity settings, on a host where the bound PersistentVolume is not available, it will also create it in the zone with the unfulfilled claim. 
*   _allowedTopologies_: - It binds your StorageClass with the zone labels (you can use any kind of label)

The warm and cold ones has the following configuration:


```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-hot
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```


Basically the same apart from the disk type, which is pd-standard (a slow disk).

**Create SAML configmap**


```
kubectl create configmap saml-metadata --from-file=../framsouza_eu_auth0_com-metadata.xml
```



## **Create Elasticsearch and Kibana resource**

Once you have the saml configmap, you can apply the Elasticsearch and Kibana manifest:


```
kubectl create -f eck-saml-hot-warm-cold.yml && kubectl create -f kibana.yml
```


It may take a while until all the resources get ready, but after some minutes you should see something like this:


```
kubectl get pods
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
elastic-prd-es-warm-zone-b-0     1/1     Running    0          16m
elastic-prd-es-warm-zone-c-0     1/1     Running    0          16m
elastic-prd-es-warm-zone-d-0     1/1     Running    0          15m
kibana-prd-kb-7467b79f54-btzhq   1/1     Running    0          4m8s
```


Once you have all the pods running, there's a [Job](https://github.com/framsouza/eck-saml-hot-warm-cold/blob/main/es-config-job.yml) you must run to create the SAML role mapping inside of Elasticsearch and give the right permission to the user who will login using SAML. You can also create the [role & role mapping](https://www.elastic.co/guide/en/elasticsearch/reference/7.12/saml-guide-stack.html#saml-role-mapping) via Kibana DevTools or curl.

If you try to access Kibana via SAML without running this Job, you will get a permission error. In summary, the job will spin up a container, execute the API calls and kill the pod.


## **Test connection**

At this point you can test the connection via curl or exposing Kibana service.

Grab the elastic password


```
kubectl get secret elastic-prd-es-elastic-user -o yaml
```


And the ES service name and run the following:


```
curl -k https://elastic:es-password@es-service-name8:9200/_cluster/health?pretty
```


You can also temporarily expose the Kibana service and access it with your browser:


```
kubectl port-forward svc/kibana-prd-kb-http 5601
```



### **Ingress controller**

By using ingress controller, you can access your Kibana via domain controller, there are some Ingress Controllers available but in this example we are using ingress-nginx, you can read more about it [here](https://kubernetes.github.io/ingress-nginx/).

To deploy it, you must run the following command:


```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/cloud/deploy.yaml
```


In this example, we are using a valid SSL certificate. To add it as part of our ingress controller we must first create a secret which contains the key and certificate.


```
kubectl create secret tls framsouza-cert --key framsouza_co_key.txt --cert framsouza_co.crt
```


Once you created the secret, you can deploy the ingress controller manifest, but first let’s have a look at the ingress manifest:


```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  name: ingress
spec:
  tls:
      - hosts:
          - framsouza.co
        secretName: framsouza-cert
  rules:
    - host: framsouza.co
      http:
        paths:
          - path: /
            backend:
              serviceName: kibana-prd-kb-http
              servicePort: 5601
```


At the annotations level, we are explicitly saying that we are using nginx as a controller and configuring the communication between the ingress controller and the service to be established via HTTP (`nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`). By the default, Elasticsearch uses HTTPS, without this annotation you may receive a 503 error.

At the tls level, we are defining the hostname (In this case framsouza.co) to check the secretname called framsouza-cert which is the secret that contains the SSL certificate.

At the rules level, we are creating a redirection. Every request that arrives at the domain [https://framsouza.co](http://framsouza.co) must be redirected to the Kibana service (`kibana-prd-kb-http`) at the port 5601.

In some cases, you also want to expose Elasticsearch to be access by some application using your own ssl certificate, to do so you can add a new rule like:


```
          - path: /elasticsearch
            backend:
              serviceName: elastic-prd-es-http
              servicePort: 9200
```


Here, every request to [https://framsouza.co/elasticsearch](https://framsouza.co/elasticsearch) will be redirected to Elasticsearch service.

Now, we are ready to apply the ingress manifest:


```
kubectl create -f ingress.yml
```


With that, we can access Kibana using our own domain and our own TLS certificate. 

Hope you can enjoyed the most powerfull Kubernetes feature together with the benefits to use ECK integrated with SAML to handle time series with hot-warm-cold architecture.

[eck-saml.zip](https://github.com/framsouza/eck-saml-hot-warm-cold/blob/main/eck-saml.zip)

Cheers,

