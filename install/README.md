# Installation Guidance

## Steps to prepare your environment

Clone this [repository](https://github.com/jbogie-vmware/tkgi-confluent-operator).  

Download and extract the [Confluent Operator bundle](https://platform-ops-bin.s3-us-west-1.amazonaws.com/operator/confluent-operator-1.6.1.tar.gz) into the cloned repository directory.  

All commands from here on are being performed in the repository directory which will be referred to as `$WORKDIR`. 

The directory structure should look like this:    

```txt
tkgi-confluent-operator ($WORKDIR)
├── README.md
├── confluent-operator-1.6.1
│   ├── COPYRIGHT
│   ├── IMAGES
│   ├── grafana-dashboard
│   ├── helm
│   ├── resources
│   └── scripts
├── install
│   └── README.md
├── network-topics
│   └── README.md
├── ref-arch-diagram
│   ├── Confluent-Operator-TKGI-Reference-Architecture-Diagram.pdf
│   └── Confluent-Operator-TKGI-Reference-Architecture-Image.jpg
├── tkgi-config-files
│   ├── vmw-tkgi-default-storage-class.yaml
│   └── vmw-tkgi-network-profile-lb-medium.json
└── tkgi-provider
    └── vmw-tkgi.yaml ($PFILE)
```

## Prepare the Providers File

In `$WORKDIR` locate the `tkgi-provider/vmw-tkgi.yaml`. This file will be referenced as the `$PFILE`. All custom settings to any of the components should be made to this file.  

Any customizations that have been pre-configured outside of what Confluent recommends have been commented with the following pattern for ease of identification: `##[//]:`. The changes will also be noted below in the corresponding sections.

## Cluster Preparation

1. In the Tanzu Kubernetes Grid Integrated Edition tile in Ops Manager, choose an unused plan and set the plan to active. Then perform the following steps:  

   - Set the `Plan Name` to `confluent-operator`.  
   - Check all three boxes for `Master/ETCD Availability Zones`  
   - Change the `Worker Node Instances` count to `12`  
   - Change the `Worker VM Type` to `large.disk`  
   - Change the `Worker Persistent Disk Type` to `100 GB`  
   - Check all three boxes for `Worker Availability Zones`  
   - Check the `Allow Privileged` box  
   - Click `Save` at the bottom of the page to create this plan  
   - Go to the `Installation Dashboard` and click `Review Pending Changes`  
   - Apply the changes you have made  

2. Create the following network profile and apply it using `tkgi create-network-profile tkgi-config-files/vmw-tkgi-network-profile-lb-medium.json`. The default 'small' load balancer seemed like it could become a bottleneck quickly so creating the 'medium' load balancer right to start seemed like the smart idea.

   ```json
   {
       "name": "cotkgi-lb-med",
       "description": "Network profile for medium NSX-T load balancer",
       "parameters": {
           "lb_size": "medium"
       }
   }
   ```

3. Create a 12 node cluster using the `confluent-operator` plan.

   ```zsh
   tkgi create-cluster cotkgi --external-hostname cotkgi.foo.bar --plan confluent-operator --network-profile cotkgi-lb-med
   ```

4. Create a DNS `A` record to point to your Kubernetes Master IP:

5. Retrieve cluster credentials and verify connectivity:

   ```zsh
   tkgi get-credentials cotkgi && kubectl get pods -o wide
   ```

6. Create the following storage class and apply it using `kubectl apply -f tkgi-config-files/vmw-tkgi-default-storage-class.yaml`. Doing this ensure any dynamically created persistent volumes would always use the vSphere Volume as default.

   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: cotkgi-vsphere-local
     annotations:
       storageclass.kubernetes.io/is-default-class: "true"
   provisioner: kubernetes.io/vsphere-volume
   reclaimPolicy: Retain
   parameters:
     diskformat: thick
   ```

7. Create a namespace to deploy the Confluent Operator into:

   ```zsh
   kubectl create namespace confluent-operator
   ```

## Deploy Confluent Operator Bundle

### Operator Pod

1. Deploy the Confluent Operator Pod:

   ```zsh
   helm install operator confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set operator.enabled=true
   ```

### Zookeeper StatefulSet

2. Deploy the Zookeeper ensemble:

   ```zsh
   helm install zookeeper confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set zookeeper.enabled=true
   ```

### Kafka StatefulSet

3. Deploy the Kafka Broker Cluster:

   ```zsh
   helm install kafka confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set kafka.enabled=true
   ```

4. Create DNS `A` records for each Kafka Broker pod's corresponding external load balancer service IP address along with the kafka bootstrap load balancer. To get the load balancer external IP addresses do the following:

    ```zsh
    kubectl get service -n confluent-operator | grep LoadBalancer
    ```

    In the example environment the DNS zone was `foo.bar` so the `A` records created were the following:

    ```zsh
    kafka.cotkgi.foo.bar
    kb0.kafka.cotkgi.foo.bar
    kb1.kafka.cotkgi.foo.bar
    kb2.kafka.cotkgi.foo.bar
    ```

### Schema Registry StatefulSet

5. Deploy the the Schema Registry Cluster:

    ```zsh
    helm install schemaregistry confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set schemaregistry.enabled=true
    ```

6. Deploy the Confluent Connect Cluster:

    ```zsh
    helm install connectors confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set connect.enabled=true
    ```

7. Deploy the Confluent Replicator:

    ```zsh
    helm install replicator confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set replicator.enabled=true
    ```

8. Deploy the Confluent Control Center:

    ```zsh
    helm install controlcenter confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set controlcenter.enabled=true
    ```

9. Create DNS `A` records for the Confluent Control Center load balancer. To get the load balancer address do the following:

    ```zsh
    kubectl get service -n confluent-operator | grep LoadBalancer
    ```

    In the example environment the DNS zone was `foo.bar` so the `A` records created were the following:

    ```zsh
    c3.cotkgi.foo.bar
    ```

    Test connectivity to the Confluent Control Center with your browser using the credentials in the

10. Deploy ksqlDB:

    ```zsh
    helm install ksql ./confluent-operator-1.6.1/helm/confluent-operator -f $PFILE --namespace confluent-operator --set ksql.enabled=true
    ```
