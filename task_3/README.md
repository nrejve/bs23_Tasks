# Design Considerations for a service-oriented ride sharing service

## Challenges ##
Foremost challenge in a riding service is to match the rider with cabs which means we need two different services in our architecture.

- Supply Service (for cars/bikes)
- Demand Service (for riders)

We will use a Dispatch system (Dispatch optimization/DISCO) in its architecture to match supply with demands. This dispatch system uses mobile phones and it takes the responsibility to match the drivers with riders.

## Dispatch System working procedure ##
DISCO must have these goals

- Reduce extra driving.
- Minimum waiting time
- Minimum overall ETA

It completely works on maps and location data/GPS, so the first thing which is important is to model our maps and location data. 

Earth has a spherical shape so it’s difficult to do summarization and approximation by using latitude and longitude. To solve this problem we will use Google S2 library. This library divides the map data into tiny cells (i.e 3km) and gives the unique ID to each cell. This is an easy way to spread data in the distributed system and store it easily.

S2 library gives the coverage for any given shape easily. Suppose you want to figure out all the supplies available within a 3km radius of a city. Using the S2 libraries you can draw a circle of 3km radius and it will filter out all the cells with IDs that lie in that particular circle. This way you can easily match the rider to the driver and you can easily find out the number of cars/bikes available in a particular region. 


## Supply Service (Cars/Bikes) ##

In our case cars/bikes are the supply services and they will be tracked by geolocation (latitude and longitude). 
All the active cars/bikes keep on sending the location to the server once every 4 seconds through a web application firewall and load balancer. The accurate GPS location is sent to the data center through Kafka’s Rest APIs once it passes through the load balancer. Here we use Apache Kafka as the data hub.

Once the latest location is updated by Kafka it slowly passes through the respective worker node’s main memory.

Also a copy of the location (latest location of cars/bikes) will be sent to the database and to the dispatch optimization to keep the latest location updated.

## Demand Service (Riders) ##

Demand service receives the request of the cars/bikes through a web socket and it tracks the GPS location of the user. 

Demand gives the location (cell ID) and user requirement to supply and make requests for the cars/bikes.

## How Does the Dispatch System Match the Riders to Drivers?

As discussed, DISCO divides the map into tiny cells with a unique ID. This ID is used as a sharding key in DISCO. When supply receives the request from demand the location gets updated using the cell ID as a shard key. These tiny cells’ responsibilities will be divided into different servers lying in multiple regions (consistent hashing). For example, we can allocate the responsibility of 12 tiny cells to 6 different servers (2 cells for each server) lies in 6 different regions. 
![image](https://user-images.githubusercontent.com/12786869/148188806-daf8e109-6029-4759-ad64-0b43e8405167.png)

Supply sends the request to the specific server based on the GPS location data. After that, the system draws the circle and filters out all the nearby cabs which meet the rider’s requirement.
After that, the list of the cab is sent to the ETA to calculate the distance between the rider and the cab, not geographically but by the road system.

Sorted ETA is then sent back to the supply system to offer it to a driver.

If we need to handle the traffic for the newly added city then we can increase the number of servers and allocate the responsibilities of newly added cities cell IDs to these servers.

## Dispatch System Scaling ##

Dispatch system is built on NodeJS. NodeJS is the asynchronous and event-based framework that allows you to send and receive messages through WebSockets whenever needed.

We will use an open-source ringpop to make the application cooperative and scalable for heavy traffic. Ring pop performs the below operation to scale the dispatch system.

It maintains the consistent hashing to assign the work across the workers. It helps in sharding the application in a way that’s scalable and fault-tolerant.

Ringpop uses RPC (Remote Procedure Call) protocol to make calls from one server to another server.

Ringpop also uses a SWIM membership protocol/gossip protocol that allows independent workers to discover each other’s responsibilities. This way each server/node knows the responsibility and the work of other nodes.
Ringpop detects the newly added nodes to the cluster and the node which is removed from the cluster. It distributes the loads evenly when a node is added or removed. 

## Map Building ##

We will use a third party map service provider to build the map in the application. to track the location and to calculate ETAs, we will use Google Maps API.

Trace coverage
Trace coverage spot the missing road segments or incorrect road geometry. Trace coverage calculation is based on two inputs: map data under testing and historic GPS traces of all Rides taken over a certain period of time. It covers those GPS traces onto the map, comparing and matching them with road segments. If we find missing road segments (no road is shown) on GPS traces then we take some steps to fix the deficiency. 

Pick-up point accuracy
We get the pickup point in our application when we book the cars/bikes. Pick-up points are a really important metric; especially for large venues such as airports, college campuses, stadiums, factories, or companies. We calculate the distance between the actual location and all the pickup and drop-off points used by drivers.


The shortest distance is then calculated and we set the pin to that location as a preferred access point on the map. When a rider requests the location indicated by the map pin, the map guides the driver to the preferred access point. The calculation continues with the latest actual pick-up and drop-off locations to ensure the freshness and accuracy of the suggested preferred access points. We will use machine learning and different algorithms to figure out the preferred access point.

## ETA Calculations ##

ETA is an extremely important metric because it directly impacts ride-matching and earnings. ETA is calculated based on the road system (not geographically) and there are a lot of factors involved in computing the ETA (like heavy traffic or road construction). When a rider requests cars/bikes from a location the app not only identifies the free/idle cars/bikes but also includes the cars/bikes which are about to finish a ride. It may be possible that one of the cars which are about to finish the ride is closer to the demand than the cab which is far away from the user. So many cars/bikes on the road send GPS locations every 4 seconds, so to predict traffic we can use the driver’s app’s GPS location data. 

We can represent the entire road network on a graph to calculate the ETAs. We can use AI simulated algorithms or simple Dijkstra’s algorithm to find out the best route in this graph. In that graph, nodes represent intersections (available cars/bikes), and edges represent road segments. We represent the road segment distance or the traveling time through the edge weight. We also represent and model some additional factors in our graph such as one-way streets, turn costs, turn restrictions, and speed limits.

Once the data structure is decided we can find the best route using Dijkstra’s search algorithm which is one of the best modern routing algorithms today. For faster performance, we also need to use OSRM (Open Source Routing Machine) which is based on contraction hierarchies. Systems based on contraction hierarchies take just a few milliseconds to compute a route — by preprocessing the routing graph.

## Databases Considerations ##

The database should be horizontally scalable. You can linearly add capacity by adding more servers.
It should be able to handle a lot of reads and writes because once every 4-second cars/bikes will be sending the GPS location and that location will be updated in the database.

The system should never give downtime for any operation. It should be highly available no matter what operation you perform (expanding storage, backup, when new nodes are added, etc).

Using the RDBMS PostgreSQL database is fine but considering scalability issues we need to switch to various databases. We will use a NoSQL database (schemaless) built on the top of the MySQL database.  

Redis for both caching and queuing. Some are behind Twemproxy (provides scalability of the caching layer). Some are behind a custom clustering system.

We will use schemaless (built in-house on top of MySQL), Riak, and Cassandra. Schemaless is for long-term data storage. Riak and Cassandra meet high-availability, low-latency demands.

## Analytics ##

To optimize the system, to minimize the cost of the operation and for better customer experience we have to perform log  collection and analysis. We will use different tools and frameworks for analytics. For log analysis, Kafka clusters can be used. Kafka takes historical data along with real-time data. Data is archived into Hadoop before it expires from Kafka. The data is also indexed into an Elastic search stack for searching and visualizations. Elastic search does some log analysis using Kibana/Grafana. Some of the different tools and frameworks are

- Track HTTP APIs
- Manage profile
- Collect feedback and ratings
- Promotion and coupons etc
- Fraud detection
- Payment fraud
- Incentive abuse by driver
- Compromised accounts by hackers.


## Disaster Recovery ##

Datacenter failure doesn’t happen very often but  still, we need to  maintain a backup data center to run the trip smoothly. 

We will use  driver phones as a source of trip data to tackle the problem of data center failure.

When The driver’s phone app communicates with the dispatch system or the API call is happening between them, the dispatch system sends the encrypted state digest (to keep track of the latest information/data) to the driver’s phone app. Every time this state digest will be received by the driver’s phone app. In case of a datacenter failure, the backup data center (backup DISCO) doesn’t know anything about the trip so it will ask for the state digest from the driver’s phone app and it will update itself with the state digest information received by the driver’s phone app. 
