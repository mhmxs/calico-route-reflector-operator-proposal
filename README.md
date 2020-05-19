# Calico route reflector autoscaling proposal

## What are route reflectors

With default installation of Calico network overlay the Border Gateway Protocol (BGP) is configured in full mesh mode. It means that every member of the network distributes routing information to every other member. The default behaviour works well in smaller networks (<100 members), but doesn’t scales well, because increasing the peers increases the network traffic between members. As a solution Calico introduced route reflectors. The concept in short is the following:
There are a small amount of members who’s responsible to communicate within the full mesh. Those members are the route reflectors. All other members are asking for route information from the nearest route reflector.

## Problem statement

The number of route reflectors heavily depends on the network size. Calico uses node labels to select route reflectors but those labels are configured manually by the operator. Manual selection works well on networks where the number of members doesn’t change too much. On other case it could be problematic and the manual process became a bottleneck if a high scaling system. So would be nice an automated feature which can auto label nodes based on the topology and the size of the network.

## Solution

In Kubernetes world operators are the key components of monitoring and updating cluster resources by reacting events in the cluster. To solve auto scaling problem of the route reflectors we can create an operator which monitors the nodes inside the cluster. When the number of healthy nodes changes in the cluster, it can calculate the ideal number of route reflectors per zone.  After the calculation it can configure the right number by adding and removing labels of the nodes.

### Configuration options:
 * minimum number of route reflectors, less than 3 might be a bad practice
 * maximum number of route reflectors, to avoid loose cannons on the ship
 * label of the route reflector node, by default `calico-route-reflector=`
 * regular and route reflector node ratio calculation method
 * label of the zone, optional for single zone clusters
 * label of the preferred nodes, optional to help node selector find the best targets
 * label of forbidden nodes, optional to disable route reflector on special nodes in the cluster

### Features in bullets:
 * Use a dedicated Custom Resource to configure route reflector auto scaler.
 * Calculate route reflector nodes based on the `healthy` node number in the cluster.
 * Sharding route reflector nodes across zones
   * Ballanced: same amount of route reflectors will selected per zone
   * Per zone calculation: based on the nodes per zone
 * Remove route reflector from `cordon`ed nodes
 * Prefer nodes during selection based on label
 * Disable node selection as route reflector based on label
 * Calculate ratio with different options
 
 ### Calculation methods
 
  * Exponential: `x=a(b^y)+c`
    * if `a` equals `120`, `b` `1.0003` and `c` `-117`:

|total number of nodes (y)|~number of RR nodes (x)|
|-|-|
|0|3|
|100|7|
|200|10|
|500|22|
|5000|420|

Calculate your own: https://www.desmos.com/calculator/o3nnc3aho8

  * Stepping: ranges are configured by Custom Resource like
 
|range of node number|number of RR nodes|
|-|-|
|0-50|3|
|51-100|9|
|101-500|20|
|501-5000|42|

  * Linear: ratio is configured by Custom Resource like
    * if ratio is 0.111:
    
|total number of nodes|~number of RR nodes|
|-|-|
|0|3|
|50|6|
|100|11|
|200|22|
|500|55|
|5000|555|

## Future reading:

 * Issue: https://github.com/projectcalico/calico/issues/2382
 * POC: https://github.com/mhmxs/calico-route-reflector-operator
 * Kubernetes operator: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
 * Kubernetes Custom resources: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
 * Calico BGP peering:  https://docs.projectcalico.org/getting-started/kubernetes/hardway/configure-bgp-peering
 * Route reflectors in Calico: https://www.tigera.io/blog/configuring-route-reflectors-in-calico/
      
## Contribution

We appreciate your help! Please feel free to share your ideas and feature requests about the topic, by leaving some comments, creating and issue or simply send me a PR. 
