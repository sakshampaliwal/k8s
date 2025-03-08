# Stateless vs Stateful

State represents the data that is stored and used to keep track of the current status of the application. Understanding and managing state is crucial for building interactive and dynamic web applications. a stateful application will have a server that remembers clients' data (that is, their state). All future requests will be routed to the same server using a load balancer with sticky sessions enabled. In this way, the server is always aware of the client.

Sticky sessions is a configuration that allows the load balancer to route a user's requests consistently to the same backend server for the duration of their session. This is in contrast to [traditional load balancing](https://lightcloud.substack.com/i/102200211/load-balancing-explained), where requests from a user can be directed to any available backend server in a round-robin or other load distribution pattern.

### Stateless:

“Stateless” architecture is a confusing term, as it implies the system is without state. A stateless architecture does not, however, mean that state information is not stored. It simply means that state information is stored outside of the server. Therefore, the state of being stateless only applies to the server. By storing the ‘state’ of a customer's order on a central system accessible by other waiters, any waiter can serve any customer.

In a stateless architecture, HTTP requests from a client can be sent to any of the servers.

State is typically stored on a separate database, accessible by all the servers. This creates a fault tolerant and scalable architecture since web servers can be added or removed as needed, without impacting state data.

The load will also be equally distributed across all servers, since the load balancer will not need a sticky session configuration to route the same clients to the same servers.

**The key principle behind something that is stateful is that it has perfect memory or knowledge of previous calls or requests, while something that is stateless has no memory or knowledge of previous calls or requests.**

Stateful applications are applications that store data and keep tracking it. All databases, such as MySQL, Oracle, and PostgreSQL, are examples of stateful applications. Stateless applications, on the other hand, do not keep the data. Node.js and Nginx are examples of stateless applications. For each request, the stateless application will receive new data and process it.

In a modern web application, the stateless application connects with stateful applications to serve the user's request. A Node.js application is a stateless application that receives new data on each request from the user. This application is then connected with a stateful application, such as a MySQL database, to process the data. MySQL stores data and keeps updating the data based on the user's request.

A StatefulSet is the Kubernetes controller used to run the stateful application as containers (Pods) in the Kubernetes cluster. StatefulSets assign a sticky identity—an ordinal number starting from zero—to each Pod instead of assigning random IDs for each replica Pod. A new Pod is created by cloning the previous Pod’s data. If the previous Pod is in the pending state, then the new Pod will not be created. If you delete a Pod, it will delete the Pod in reverse order, not in random order. For example, if you had four replicas and you scaled down to three, it will delete the Pod numbered 3.

**When to use statefulset:**
1. Assume you deployed a MySQL database in the Kubernetes cluster and scaled this to three replicas, and a frontend application wants to access the MySQL cluster to read and write data. The read request will be forwarded to three Pods. However, the write request will only be forwarded to the first (primary) Pod, and the data will be synced with the other Pods. You can achieve this by using StatefulSets.