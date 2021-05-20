This document explains how to set up SAML with auth0 as idp using a hot/warm/cold artechicture with ECK. The purpose of this project is to use production features such as dedicated nodes, zone awareness, pod and node affinity, podDisruption and dedicated storage class. To know more, check this ECK article. Also, you should be familiar with SAML.

### Scenario
In this scenario we deploy ECK to handle application logging along with a centralized way to authenticate into Kibana using auth0. As logging data (time-series) has a predictable time of life, we can use hot, warm and cold architecture to better deal with your data over time.

We will deploy the following resources:
- ECK Operator
- StorageClass
- Elasticsearch
- Elasticsearch role Job
- Kibana
- Ingress Controller
- ConfigMap SAML metadata

### Architecture

To set up this environment, we use GKE with the following configurations:

_3 Nodes pools_
- 1 Hot Node Pool wtih 6 Kubernetes instances running spread across 3 availability zones
- 1 Warm Node Pool with Kubernetes instances running spread across 3 availability zones
- 1 Cold Node Pool with Kubernetes instances running spread across 3 availability zones

_Instances configuration_
- hot nodes: c2-standard-4 (4 vCPUs, 16 GB memory)
- warm nodes: e2-standard-2 (2 vCPUs, 8 GB memory)
- cold nodes: e2-standard-2 (2 vCPUs, 8 GB memory)

_9 Elasticsearch instances (Split by hot and warm, cold and dedicated master) plus 2 Kibana instances running at zones europe-west1-b, europe-west1-c, europe-west1-d_ 

For each node pool we use a specific Kubernetes label to attach the Elasticsearch instance into the right hard configuration. The label name is called "type" and the values are: hot, warm or cold. You can change and add the label later by using the command line.

### This document contains two parts:
- Manifest explanation
- How to setup

## Manifest explanation
The Elasticsearch manifest file eck-saml-hot-warm-cold.yml contains the Elasticsearch settings.

### nodeSets:
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

We have one node called hot-zone-b deployed at zone europe-west1-b. Then, we define two routing awareness attributes:
_k8s_node_name_: Makes sure that Elasticsearch allocates primary and replica shards to pods running on different Kubernetes nodes and never to pods that are scheduled onto a single Kubernetes node.

_zone_: Uses the Kubernetes label called domain.beta.kubernetes.io/zone and it makes sure that the cluster will redirect the shards according to these settings.

We explicitly say that this node will have _data_hot_ and _data_content_ role. 
Finally, _node.store.allow_mmap : false_ will prevent the virtual address space (on Linux distribution) ro run into errors or exceptions because this default configuration is too low.

### xpack.security
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

This is the SAML configuration. You should follow this guide to configure auth0 as idp and then get the metadata file and the idp.* settings.

- attributes.principal: Defines which SAML attribute is going to be mapped to the principal (username) of the authenticated user in Kibana
- idp.entity_id: Is the SAML EntityID of your Identity Provider (you can get it on SAML metadata file)
- idp.metadata.path: Is the file path or the HTTPs URL where your Identity Provider metadata is available. In this example,, we use a file to demonstrate how to mount a volume
- sp.acs: Is the Assertion Consumer Service URL where Kibana is listening for incoming SAML messages.
- sp.entity_id: Is the SAML EntityID of our Service Provider.
- sp.logout: Is the SingleLogout endpoint where the Service Provider is listening for incoming SAML LogoutResponse and LogoutRequest messages

Keep in mind the **sp.*** configurations must point to Kibana endpoint and the **idp.*** settings you must collect from your Identity Provider. Also, the SAML metadata is stored in a ConfigMap and mounted as a Volume inside Elasticsearch. The idp also may provide the metadata via HTTP, in this case, auth0 provides the metadata file via HTTP but the purpose is to show to you how to mount configMap (or secret) as a volume into Elasticsearch pods.

### volumeClaimTemplates
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

The hot node will have a 50Gi available of disk which refers to the storage class called sc-zone-b, this is the space available to store shards on this instance.

### Node and Pod affinity
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

nodeAffinity_ is using _requiredDuringSchedulingIgnoredDuringExecution_ affinity type. That means, a hard limit that rules must be met for a pod to be scheduled in a node. In this example, we're defining the following: "only run the pod on nodes in the zone europe-west1-b". The nodeSelector will be matched with the Kubernetes Label you are using at the node, in this case type: hot.

### Containers definition
```
       containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: -Xms4g -Xmx4g
          - name: READINESS_PROBE_TIMEOUT
            value: "10"
          resources:
            requests:
              memory: 8Gi
              cpu: 200m
            limits:
              memory: 8Gi
              cpu: 200m
```

The JVM configuration is defined via environment variables, this container will have 4GB of heap space.  Keep request and limit with the same value to minimize disruption caused by pod evictions due a resources utilization. In this example, this node has 8Gi of memory and 200m CPU.

### Volume SAML metadata
```
         volumeMounts:
          - name: saml-metadata
            mountPath:  /usr/share/elasticsearch/config/framsouza_eu_auth0_com-metadata.xml
        volumes:
        - name: saml-metadata
          configMap:
            name: saml-metadata
```

The volumeMounts session is where the volume should be mounted inside the pod. In this example, you are mounting a volume saml-metadata and the file is located in ES_CONFIG directory _/usr/share/elasticsearch/config/framsouza_eu_auth0_com-metadata.xml_. Then, volumes means the volume must be mounted according to a configMap called saml-metadata.

### Creating a configMap
You must create this file before the ES manifest.

You get the metadata file from your idp. To put the content inside a configMap you can run the following:

`kubectl create configmap saml-metadata --from-file=../framsouza_eu_auth0_com-metadata.xml
`

Remember to adjust the file location.

Easy, right?

If you want to use a secret instead, the manifest should have something like this:
```
         volumeMounts:
          - name: saml-metadata
            mountPath:  /usr/share/elasticsearch/config/framsouza_eu_auth0_com-metadata.xml
        volumes:
        - name: saml-metadata
          secret:
            secretnNme: saml-metadata
```

`kubectl create secret generic saml-metadata --from-file=../framsouza_eu_auth0_com-metadata.xml
`

### redinessProbe
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
```

Readiness probe is used to know when a container is ready to start accepting traffic. Here we are increasing the timeout to 10 seconds. The default value is 3 seconds.

### initContainer
```
       initContainers:
        - command:
          - sh
          - -c
          - |
            bin/elasticsearch-plugin install --batch repository-gcs
          name: install-plugins
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
```

To install or perform any task at the operational system level before Elasticsearch starts, you should use _initContainer_. In this example, we are installing the GCS repository where we can send the indices snapshots.

Next, the sysctl container will increase the mmapfs, that is some operational system is too low by default, which may result in out-of-memory exception. The container will run and be killed before Elasticsearch starts.

These are the sessions from one node. The rest of the node configuration is basically the same, except for the name, zone, and _storageClass_ name. At the end of the file, you see the following setting (which applies to the whole cluster):
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

_updateStrategy_ controls the number of simultaneous changes in the Elasticsearch cluster. maxSurge: 1 means only one new Pod is created at a time. After the first new Pod is Ready, an old Pod is killed and the second new Pod is created. Where _maxSurge_ determines how many new Pods to create, maxUnavailable determines how many old Pods to kill. In this case, we can only kill one old Pod at a time. This ensures the capacity is always at least 3 - 1 Pods. If this is not enough for your environment configuration, you can disable the podDisruption and configure your own.

_podDisruptionBudget_ determines how many nodes must continue running in case of disruption.

## How to
### Install ECK

Set up Google cloud RBAC by running the following command:

`kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")`

### Install the ECK operator:
`kubectl apply -f https://download.elastic.co/downloads/eck/1.5.0/all-in-one.yaml
`

This creates the ECK Operator pod and cluster permissions in the namespace elastic-system.

### Configuring StorageClass

With StorageClass you can describe the "classes" of storage you want to use. Different classes might map to quality of service levels, backup purpose or any arbitrary policy determined by the administrator.
As we are using different zones and different hard configurations because of the  _hot/warm/cold_ architecture, we need to specify a StorageClass for each zone and associate it with the proper _PersistentVolume_.

This is an example of StorageClass that is attached to the PersistentVolume in the hot nodes:
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
- provisioner: kubernetes.io/gce-pd : It means we are using GCE as provider;
- type: pd-ssd: - It provides ssd disks for the volumes attached to this StorageClass;
- reclaimPolicy: Delete - It deletes PersistentVolumeClaim resources if the owning Elasticsearch nodes are scaled down or a deletion;
- volumeBindingMode: WaitForFirstConsumer - It prevents the pod from being scheduled, because of affinity settings, on a host where the bound PersistentVolume is not available, this is a very crucial configuration;
- allowedTopologies: - It binds your StorageClass with the zone labels (you can use any kind of label)

### Create SAML configmap

`kubectl create configmap saml-metadata --from-file=../framsouza_eu_auth0_com-metadata.xml
`

### Create Elasticsearch and Kibana resource
Once you have the saml configmap, you can apply the Elasticsearch and Kibana manifest:

`kubectl create -f eck-saml-hot-warm-cold.yml && kubectl create -f kibana.yml
`

It may take a while until all the resources get ready, but after some minutes you should see something like this:

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
elastic-prd-es-warm-zone-b-0     1/1     Running    0          16m
elastic-prd-es-warm-zone-c-0     1/1     Running    0          16m
elastic-prd-es-warm-zone-d-0     1/1     Running    0          15m
kibana-prd-kb-7467b79f54-btzhq   1/1     Running    0          4m8s
```

Once you have all the pods running, there's a Job you must run to create the SAML rolling mapping and give the right permission to the user who will login using SAML. You can also create the role & rolling mapping via Kibana DevTools or curl.
If you try to access Kibana SAML configuration without running this Job, you will get a permission error. In summary, the job will spin up a container, execute the API calls and kill the pod.

### Test connection

At this point you can test the connection via curl or exposing Kibana service.

Grab the elastic password
`kubectl get secret elastic-prd-es-elastic-user -o yaml
`

And the ES service name and run the following:
`curl -k https://elastic:es-password@es-service-name8:9200/_cluster/health?pretty
`

You can also expose the Kibana service and try to access it with your browser:
`kubectl port-forward svc/kibana-prd-kb-http 5601
`

### Ingress controller

Deploy the ingress resource:
`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/cloud/deploy.yaml
`

Before deploy the controller, create the SSL certificate,
`kubectl create secret tls framsouza-cert --key framsouza_co_key.txt --cert framsouza_co.crt
`

Once the pods are running, you can deploy the ingress controller manifest:
`kubectl create -f ingress.yml
`

In this example, we have a domain called framsouza.co and a valid certificate, remember to adjust these settings.
And with that, we can access Elasticsearch using our own domain with our  TLS certificate. This also covers the most important features to run an ECK environment in a production environment.


Cheers,

