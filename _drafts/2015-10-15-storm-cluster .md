# Storm Basics - Cluster 

Storm cluster is centered around Zookeeper as below.

![storm cluster](https://lh3.googleusercontent.com/pND94of7R4gGvttxITsTXFkqAmNi8mORJAtpSywLULjaaQzF_7MT3Ajr2IYLVVIxZYeIE9BUXq999gNzyK-dBSQ7dl4N47Ea_UBEmQ8wY9dHGweoRw3cuUgndYn5fnk9tRLJDj9kR9YuXyj5kV55xOVT7EE8TtjPDacr9RUsan5soU0p5923jl0ggdt4uRuGksp8eBZ3fiHK7t6XFZSXyQy3BSLt0F1V2rRTcybXS9AicOXnOm9IHVSXsTgPnlvi76B1W3w-GaPmb0VIRNQSxuBd2FyY0ps1s2SVL8rN9A759IcNWI0RUECDDM7Ha9w18daLAA_s-1OnipPjwCbgX7IEAthnt8Mjno3r77-W7gFc0J1OiD9nybEy2N4zeAvasb2IM2B7uFjIGDq870ZLYrWzqcmb8xBw10t-JYvM8s1N72Ins57I8efpqGcLWFidnatppdnfC9Txj6R5u8juXkynL4gUtQYzF1NUDJ1I40kuJE2GdvmlOp8avlBUXQzeg6fz1UH71YDDYZqo_6Rk9FhxsQYBdeUeOIY6BP3nbRs=w845-h688-no)

Here is the workflow,

1. one Nimbus and one or more Supervisors are launched on master and worker nodes.
2. Supervisor heartbeats its local resources to Zookeeper `supervisors/`, which Nimbus subscribes to
3. multiple clients submit topologies to Nimbus
4. Nimbus stores jar file, codes and configuration locally, and make assignments to Supervisors through Zookeeper nodes `assignments/`.
5. Supervisor learns that, downloads jar file, codes and configuration from Nimbus and also stores locally
6. Supervisor launchs Workers according to the assignment and write local assigments to local store
7. Workers create executor threads and heartbeats to Supervisor via local store
8. Nimbus start topology by setting the status in Zookeeper `storms/`, and worker will start executor threads. Worker heartbeats executor status to Nimbus through Zookeeper `workerbeats/`
9. executors report errors to Zookeeper `errors/`


## Nimbus 

The master node runs a thrift server, called `Nimbus`, who is responsible for 

1. distributing code around the cluster
2. assigning tasks to worker nodes
3. monitoring for failures
4. collecting metrics

Clients communicate with Nimbus through thrift API. That is quite flexible and powerful,

1. clients could be written in any language. 
2. Nimbus could bridge Storm clients with other streaming engines 


## Supervisor 

Each worker node has a `Supervisor` that 

1. listens to assignment from Nimbus
2. starts and stops worker process 

Supervisor periodically informs Nimbus about available local resources, which are the amount of worker processes (JVMs) that can be launched on the Supervisor. 

```yaml
# Define the amount of workers that can be run on this machine. Each worker is assigned a port to use for communication
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

When a topology is submit to it, Nimbus will schedule tasks among the available resources and assign them to each Supervisor. Then Supervisors launch worker processes to run tasks according to the assignments.  

Supervisors don't rely on Nimbus for its existence which means existing topologies will continue to run when Nimbus is down.

## Worker

A `Worker` is a JVM process which launches three types of threads.

* `Executor` threads to execute tasks. 
* one or more receive threads to receive incoming data and publish to corresponding executor's receive queues. 
* one transfer thread to push out data from executor's send queues 



![nimbus_supervisor_worker_lifecycle](https://lh3.googleusercontent.com/eidFRqQ7_uXyCfmdtFT3pt-oZTZYczcHiCN28RsOqJDdWOEDz1naY-UvLpW3yqWX50rI4fT1hiji-Jp8vcLBjtE8zf7ABLRRqNAteqQi85Bb9FzbC2FPt3pqnTF9fhnrVXiC6DokFoP0rYUSpoEdC3FVscmJz01MLdkggrY0QPRzQSxVbqDG9b8MQeveFRa-L4ES04go4BmRbd81a8NOv_4LBN3q2gkkXt7BXrNbAmj8BFcYx5-ahZz8xufJulLzcga4pXMSn8Xuz_fDXGqvqa5J6-VFrQU0uKd5i_or_BjNQ7gl9943gSzhdIgnZKSWexMoJ142UnrvyylvfY3XbOaUpFdx3-cupHD7t_XH08z_HxBzUpOEp1dqZ3QSqONuyNJ-zcys4dXZYwd2jGHV6ZxL5yvq_K0DfraQrwiovlZyhkjYG6EoUfCMwWAEs-1FF_cT6aWjPztAXMtDW4Vodc4FA_EDq4vx2JHm3vCQIPN-ca0Uip5HesUqP15sXj_f738fEsbO17WwMWBOy7bLlXsr9YKgnnmRchBcZrR_6O0=w758-h630-no)

