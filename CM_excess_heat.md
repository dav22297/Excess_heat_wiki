## Table of Contents

## Introduction
The use of excess heat for district heating.


## Method
### Overview
The key element of the excess heat module is the source sink model used.
It constructs a transmission network of minimum length and computes the flow for every hour of the year based on residential heating load profiles with Nuts2 resolution and industry load profiles with Nuts0 resolution. Based on averaged peak flows throughout the year costs for every transmission line and heat exchanger on the source and sink side can be computed.

### Details
#### Modeling of sources
Based on the Nuts0 ID and the industrial sector a yearlong hourly resolved load profile is assigned to every source.

#### Modeling of sinks
Based on the district heating potential calculation module equidistantly entry points are created in the coherent areas. Depending on the Nuts2 ID of the entry points a load profile is assigned.

#### Fixed radius search
Within a set radius it is checked which sources are in range of each other, which sinks are in range of each other and which sinks are in range for sources. This can be represented by a graph with sources and sinks forming the vertices and the vertices in range being connected by an edge.

#### Reduction to minimum length network
A minimum spanning tree is computed with the distance of the edges as weights. This results in a graph retaining its connectivity while having a minimum total length of edges. Note that the entry points of coherent areas are connected internally for free since they form their own distribution network.

#### Flow computation
The maximum flow from the sources to the sinks is computed for every hour of the year.

#### Cost determination
The peak flow of the year averaged over 3 hours determines the required capacity for the transmission lines and heat exchangers. The costs of the transmission lines depend on the length and capacity, while the costs of the heat exchangers are only influenced by the capacity. On the source side an air to liquid heat exchanger with integrated pump for the transmission line and on the sink side a liquid to liquid heat exchanger are assumed.

#### Variation of network
Since the cost and flow of every transmission line are known the lines with the highest cost to flow ratio can be removed and the flow recomputed until a desired cost per flow is achieved. 

### Implementation
#### Fixed radius search
For the computation of the distance between two points a small angle approximation of the loxodrome length is used. While there is also an accurate implementation of the orthodrome distance the increased accuracy has no real benefit because of the small distances mostly lower than 20km and the uncertainty of the real transmission line length because of many factors like topology.
If two points are in range of the radius it is stored in an adjacency list. The creation of such adjacency lists is performed between sources and sources, sinks and sinks, and sources and sinks. The reason for the separation lies in the flexibility to add certain temperature requirements for sources or sinks.

#### NetworkGraph class
Based on the igraph library a NetworkGraph class is implemented with all functionality needed for the calculation module. While igraph is poorly documented it offers much better performance than pure python modules like NetworkX and a wider platform support beyond Linux unlike graph-tool.
The NetworkGraph class describes only one network on the surface but contains 3 different graphs. Firstly, the graph describing the network as it is defined by the three adjacency lists. Secondly the correspondence graph internally connecting sinks of the same coherent area and last the maximum flow graph used for the maximum flow computation.

##### Graph
Only contains the real sources and sinks as vertices.

<figure>
  <img src="https://github.com/dav22297/Excess_heat_wiki/blob/master/figures/graph.svg" alt=""/>
  <figcaption>Example of a graph. The red vertices represent sources and the blue ones sinks.</figcaption>
</figure>


##### Correspondence graph
Every sink needs a correspondence id, which indicates, if it is internally connected by an already existing network like in coherent areas. Sinks with the same correspondence id are connected to a new vertex with edges with zero weights. This is crucial for the computation of a minimum spanning tree and the reason the correspondence graph is used for it. This feature is also implemented for sources but not used.

<figure>
  <img src="https://github.com/dav22297/Excess_heat_wiki/blob/master/figures/correspondence_graph.svg" alt=""/>
  <figcaption><i>Example of a correspondence graph. The red vertices represent sources and the blue ones sinks. The three sinks on the right are coherent connected by an additional larger vertex</i></figcaption>
</figure>

##### Maximum flow graph
Since igraph does not support multiple sources and sinks in its maximum flow function an auxiliary graph is needed. It introduces an infinite source and sink vertex. Every real source is connected to the infinite source and every real sink is connected to the infinite sink by an edge. Note that if a sink is connected to a correspondence vertex this vertex will be connected rather than the sink itself.

##### Minimum spanning tree computation
Based on the correspondence graph the minimum spanning tree is computed. The edges connecting the coherent sinks always have the weight 0 so they will always remain part of the minimum spanning tree.

##### Maximum flow computation
The flow through the edges connecting the real sources or sinks to the infinite source or sink respectively is capped to the real capacity of each source or sink. For numerical reasons the capacities are normalized so that the largest capacity is 1. The flow through the subset of edges contained in the correspondence graph is limited to 1000 which should, for all intense and purposes offer unrestricted flow. Then the maximum flow from the infinite source to the infinite sink is computed and the flow rescaled to its original size. Since coherent sinks are not directly connected to the infinite sink vertex but by the correspondence vertex the flow through it is limited to the sum of all coherent sinks.

