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
 * wait timeout, optional to wait some time before removing old route reflectors to avoid loosing all route reflectors of a node

### Features in bullets:
 * Use a dedicated Custom Resource to configure route reflector auto scaler. (Shaun: kube-controllers already has a CRD for its configuration, the new configuration could be a sub-struct of that.)
 * Calculate route reflector nodes based on the `healthy` node number in the cluster
 * Sharding route reflector nodes across zones
   * In single cluster topology zones are handled individually 
   * In all other topologies controller must select route reflectors in an equally balanced mode per zone.
   If zone has less nodes than the expected number of route reflectors the controller tries to place missing route reflectors in the next zone.
 * Distribute selected route reflectors for of a node across multi zones
  * In single cluster topology all client connects to all route reflectors in the same zone
  * In all other topologies controller selects at least two different zones, prefered from the same zone at the node located
 * Remove route reflector from `cordon`ed nodes
 * Prefer nodes during selection based on label
 * Disable node selection as route reflector based on label
 * Make sure node toleraits (taint) route reflector pod
 * Calculate ratio with different options
 * Support multiple datastores
 
 #### Robustness
 * Avoid rebalancing route reflectors too often
   * Controller sorts nodes by creation time and hopes old ones have more chance to survive
   * Controller reuse old route reflector selections as much as possible
 * Protect node to loose all route reflectors
    * Controller should wait some time before removing obsolete route reflectors
    * In long term should be more robust to check BGP sessions instead of waitig fixed time foolishly
 * Avoid loosing a zone have effect on working zones
    * Select one route reflector from the same zone, one from any other zone and the third one from any zone. In this way each client has route reflectors from different zones.
 
 ### Route Reflector topologies
 
 #### Single cluster [50-500 nodes]
 
  The simplest route reflector topology contains only one cluster ID. There are only one group of route reflectors and one group for clients. This topology doesn't scale well and useful only for single zone or single region clusters. The number of client connections per route reflector could be a bottleneck very easy.
 ```
       _________________
      /                  \
     RR1-------RR2-------RR3
     |/ \ / \ /  \ / \ / \|
     |\ / \ / \  / \ / \ /|
  Client1    Client2    Client3
 ```
 
| | |
|-|-|
| # of nodes | 500 |
| # of RRs | 3 |
| Redundancy | 3 |
| # of clients / RR | 497 |
| # of RRs / RR | 2 |
| Connections / RR | 499 |
 
  #### Multi cluster [500-2000 nodes]
 
 Each Route Reflector has its own cluster ID. Clients are connecting to 3 different clusters and route reflectors are constituting one mesh.
 ```
       _______________
      /         |      \
    RR1        RR2      RR3
     |   \ /        \ /  |
     |   / \        / \  |
  Client1   Client2   Client3
 ```
 Route reflector during it advertises a route it got from a client, it adds its cluster ID to it. When an RR advertises a route it got from another RR, it appends its cluster ID to the list. When an RR receives multiple copies of the same route (i.e the same dest CIDR and same next hop), it will only advertise one copy of the route. It will always choose a copy of the route with fewest cluster IDs associated with it and also doesn't advertise routes which already has it's cluster ID in the list. BGP update message size and number could be a bottleneck because all route reflectors advertise the full table to all other route reflectors.

| | |
|-|-|
| # of nodes | 2000 |
| # of RRs | 11 |
| Redundancy | 3 |
| # of clients / RR | 542 |
| # of RRs / RR | 10 |
| Connections / RR | ~552 |

 #### Quorum cluster [1000-2000 nodes]
 
 Route Reflectors can be divided into clusters based on cluster ID.  Within a cluster, RRs do not share routes with each other that they learned from their clients.  This means that each client must be connected to a quorum of RR nodes within the cluster in order to share at least one RR node with every other client.  For example, with a cluster of three RRs, each client must peer with at least 2:
 ```
       ________________
      /           ______\
    RR1_______ RR2      RR3
     |   \ /        \ /  |
     |   / \        / \  |
  Client1   Client2   Client3
 ```
 The RRs also peer with each other since, in a Calico network, they also have local client routes to advertise.  However, a route learned from Client 1 by RR1 will not be passed to RR2/3.
 
 This topology scales well but does not suggersted for giant clusters. Near 3000 (depends on the flavor of nodes) nodes the BGP connection number became bottleneck and there should be a "which finger to bite" situation, because increasing the number of route reflectors can decrease the number of BGP connections but increases the size of the BGP update messages in the same time and that became the new bottleneck of the system.
 
| | |
|-|-|
| # of nodes | 2000 |
| # of RRs | 13 |
| Redundancy | 3 |
| # of Quorums | 286 |
| # of clients per quorum | 7 |
| # of quorum / RR | 63 |
| # of clients / RR | 456 |
| # of RRs / RR | 12 |
| Connections / RR | ~468 |

 #### Hierarchy [2000-5000 nodes]
 
 The most scalable option is to mimic the structure of a datacenter network, dividing the cluster into "racks" and having a pair of RRs per "rack", then having a second level of RR to which the "rack" RRs are essentially clients.

```
      ________
     /        \
    S1---S2---S3 
    X          X
 R1 R2 R3   R4 R5 R6
 | X X |    | X X |
 C1 C2 Cn  C3 C4 Cm
 ```

| | |
|-|-|
| # of nodes | 5000 |
| # of racks | 10 |
| # of RRs | 33 |
| Redundancy | 3 |
| # of clients / RR | 447 |
| # of RRs / RR | 3 |
| Connections / RR | ~500 |

| calico-node | RouteReflectorClusterID |
|-------------|-------------------------|
| S1,S2,S3       | 0.0.0.1                 |
| R1,R2,R3       | 0.0.0.2                 |
| R4,R5,R6       | 0.0.0.3                 |

 Every client peers with both of the RRs in its "rack". The rack RRs then peer with:

 **(a)** At least a quorum of the spine RR if the rack RR are running other workloads as well. (i.e. non-dedicated)<br>
 **(b)** With a minimum of 1 spine RR if the rack RR is dedicated to RR function. 

There's no need for a direct session between R1<>R2 and R3<>R4 as they'll receive each other's routes via a spine RR, with the spine's RouteReflectorClusterID in the CLUSTER_LIST BGP attribute. A BGP UPDATE from C1 on C3 will have all three ClusterIDs in the CLUSTER_LIST for routing information loop prevention. Because of how Calico configures BIRD, a drawback of this solution is that UPDATES from a spine are reflected back to the other spine(s), which in turn will drop such UPDATES as it has its own Cluster ID in the CLUSTER_LIST.
 
 The following BGP sessions needs to be configured:
 
 * Route reflector -> Client
   * [S1,S2] -> [R1,R2]
   * [S1,S2] -> [R3,R4]
   * [R1,R2] -> [C1,Cn]
   * [R3,R4] -> [C3,Cm]
  
 * Route reflector <-> Route reflector (i.e. a regular iBGP session)
   * S1 <-> S2
 
 
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
 
 For the other topologies, each RR can handle 500 clients comfortably, so there could be 500 clients served by a 3-RR cluster with a full mesh between the RR clusters or 500 clients in a "rack" with up to 500(!) "racks" served by a single spine pair.

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
