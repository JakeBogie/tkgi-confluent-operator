# Networking Considerations

## Load Balancing Ingress Traffic with NSX-T

Load balancers can be easily provisioned as part of the cluster creation task in TKGI. Along with the default load balancers for management of your cluster; deployments can also create load balancer services that are provisioned and managed in NSX-T. This is all done through a 'network profile' that can be customized based upon cluster needs in TKGI.

When a namespace is created in a cluster in TKGI a new NSX-T logical segment is created for the namespace and is attached to the shared tier-1 gateway for the Kubernetes cluster. A static NAT rule is also created for the logical segment for outbound access. The translated IP is taken from the floating pool of addresses pre-defined in NSX-T.

During installation of Confluent Operator components load balancers are created in NSX-T as part of the creation of StatefulSet resource like Kafka. The sizing for your load balancers can be as easy as t-shirt sizing. Depending on your intended use for the cluster you can create a default load balancer template that is used to create all subsequent load balancers in your cluster.

In the installation of the Confluent Platform on TKGI we utilize a `medium` network profile for any load balancers created in this cluster. These load balancers will be displayed in Kubernetes as LoadBalancer services and in NSX-T as VIPs on the clusters tier-1 load balancer.

For further information regarding load balancer sizing in TKGI review the following article:

[Size a Load Balancer](https://docs.pivotal.io/tkgi/1-8/network-profiles-lb-size.html)

Enabling load balancers for Confluent Platform components example:

**Kafka cluster without load balancing enabled**

```yaml
## Kafka Cluster
##
kafka:
  name: kafka
  replicas: 3
  loadBalancer:
    enabled: false
    domain: ""
```

Changing the `enabled` value to true and entering the `domain` information for our cluster will guide Kubernetes to provision a load balancer via the NSX-T integration and assign it to the Kafka cluster.

```yaml
## Kafka Cluster
##
kafka:
  name: kafka
  replicas: 3
  loadBalancer:
    enabled: true
    domain: "cotkgi.foo.bar"
```

By performing this step above a LoadBalancer service is instantiated as the front-end ingress access to the Kafka cluster.