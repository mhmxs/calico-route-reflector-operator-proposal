# Calico route reflector autoscaling proposal

## What are route reflectors

With default installation of Calico network overlay the Border Gateway Protocol (BGP) is configured in full mesh mode. It means that every member of the network distributes routing information to every other member. The default behaviour works well in smaller networks (<100 members), but doesn’t scales well, because increasing the peers increases the network traffic between members. As a solution Calico introduced route reflectors. The concept in short is the following:
There are a small amount of members who’s responsible to communicate within the full mesh. Those members are the route reflectors. All other members are asking for route information from the nearest route reflector.

## Problem statement

The number of route reflectors heavily depends on the network size. Calico uses node labels to select route reflectors but those labels are configured manually by the operator. Manual selection works well on networks where the number of members doesn’t change too much. On other case it could be problematic and the manual process became a bottleneck if a high scaling system. So would be nice an automated feature which can auto label nodes based on the topology and the size of the network.

## Solution

In Kubernetes world operators are the key components of monitoring and updating cluster resources by reacting events in the cluster. To solve auto scaling problem of the route reflectors we can create an operator which monitors the nodes inside the cluster. When the number of healthy nodes changes in the cluster, it can calculate the ideal number of route reflectors per zone.  After the calculation it can configure the right number by adding and removing labels of the nodes.

Calico has an existing set of controllers/operators, including one that monitors nodes for cleanup purposes.  It would make sense to add the new operator to the same repo: https://github.com/projectcalico/kube-controllers.

### Configuration options:
 * minimum number of route reflectors, less than 3 might be a bad practice
 * maximum number of route reflectors, to avoid loose cannons on the ship
 * label of the route reflector node, by default `calico-route-reflector=`
 * route reflector cluster ID, by default `224.0.0.1`
 * regular and route reflector node ratio calculation method
 * label of the zone, optional for single zone clusters
 * label of the preferred nodes, optional to help node selector find the best targets
 * label of forbidden nodes, optional to disable route reflector on special nodes in the cluster

### Features in bullets:
 * Use a dedicated Custom Resource to configure route reflector auto scaler. (Shaun: kube-controllers already has a CRD for its configuration, the new configuration could be a sub-struct of that.)
 * Calculate route reflector nodes based on the `healthy` node number in the cluster.
 * Sharding route reflector nodes across zones
   * Ballanced: same amount of route reflectors will selected per zone
   * Per zone calculation: based on the nodes per zone
 * Remove route reflector from `cordon`ed nodes
 * Prefer nodes during selection based on label
 * Disable node selection as route reflector based on label
 * Make sure node toleraits (taint) route reflector pod
 * Calculate ratio with different options
 
 ### Route Reflector topologies
 
 #### Single cluster
 
 Route Reflectors can be divided into clusters based on cluster ID.  Within a cluster, RRs do not share routes with each other that they learned from their clients.  This means that each client must be connected to a quorum of RR nodes within the cluster in order to share at least one RR node with every other client.  For example, with a cluster of three RRs, each client must peer with at least 2:
 ```
       _______________
      /                \
    RR1-------RR2-------RR3
     |   \ /        \ /  |
     |   / \        / \  |
  Client1   Client2   Client3
 ```
 The RRs also peer with each other since, in a Calico network, they also have local client routes to advertise.  However, a route learned from Client 1 by RR1 will not be passed to RR2/3.
  
 #### Hierarchy
 
 The most scalable option is to mimic the structure of a datacenter network, dividing the cluster into "racks" and having a pair of RRs per "rack", then having a second level of RR to which the "rack" RRs pair and so on.

```
     R5----R6 
    /   \ /   \
  /     / \     \
 R1----R2  R3----R4
 | X X |    | X X |
 C1 C2 Cn  C3 C4 Cm
 ```
 Every client peers with both of the RRs in its rack.  The left/right hand "leaf" RR in each rack peers with the left/right hand RR in the "spine".  No need to peer the leaves with both spines because each spine carries the same routes.
 
 #### Need for cluster ID
 
 If the RRs in a "cluster" don't share a cluster ID then route distribution loops can occur once multiple RRs are peered together:
 
 For example, 
 * client 1 advertises a route to both RR1 and RR2, 
 * RR1 advertises it to RR3, as does RR2
 * RR3 advertises RR1's copy  of the route back to RR2
 * RR3 advertises RR2's copy  of the route back to RR1
 
 ```
        _______________
      /                \
    RR1-------RR2-------RR3
     |   \ /        \ /  |
     |   / \        / \  |
  Client1   Client2   Client3
 ```
 
 
 ### Calculation methods
 
 For a simple single cluster, adding more RRs doesn't buy much because all clients must be peered to a quorum of the RRs.  For a simple cluster topology, it'd make sense to have say `num RRs = min(num nodes, 5-9)`.  
 
 For the other topologies, each RR can handle 200 clients comfortably, so there could be 200 clients served by a 3-RR cluster with a full mesh between the RR clusters or 200 clients in a "rack" with up to 200(!) "racks" served by a single spine pair.

  * Linear: ratio is configured by Custom Resource like
    * if ratio is 0.005:
    
|total number of nodes|~number of RR nodes|
|-|-|
|1|3|
|200|3|
|1000|5|
...
|5000|25|

  * Stepping: ranges are configured by Custom Resource like
 
|range of node number|number of RR nodes|
|-|-|
|1-200|3|
|201-1000|5|
...
|501-5000|25|

## Future reading:

 * Issue: https://github.com/projectcalico/calico/issues/2382
 * POC: https://github.com/mhmxs/calico-route-reflector-operator
 * Kubernetes operator: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
 * Kubernetes Custom resources: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
 * Calico BGP peering:  https://docs.projectcalico.org/getting-started/kubernetes/hardway/configure-bgp-peering
 * Route reflectors in Calico: https://www.tigera.io/blog/configuring-route-reflectors-in-calico/
      
## Contribution

We appreciate your help! Please feel free to share your ideas and feature requests about the topic, by leaving some comments, creating and issue or simply send me a PR. 
