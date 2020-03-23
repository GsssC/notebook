<!-- Please only use this template for submitting enhancement requests -->

**What would you like to be added/modified**:
HA features for CloudCore(Hereinafter referred to as CC)
**Why is this needed**:
In [2019 Q4 Roadmap](https://github.com/kubeedge/kubeedge/blob/master/docs/getting-started/roadmap.md#2019-q4-roadmap),it says that we would support HA for CC，but not so far.Now I want to open a discussion.

**Make CC Stateless**:
Kubeedge is design to serve hundreds of edgenodes,and it is obviously single CC could hardly run well all the time ，especially when it comes to kubeedge system upgrades.When a CC is down, all edgenodes it connects to will not fetch new data any more until CC restarts.Copy is a powerful weapon against the uncertainty.If we add a LoadBalance(LB) bettween edgecore and cloudcore, once the connection breaks, edgecore can connect to the cloudcore that run well. The problem is whether the new cloudcore can serve the edgecore as well as the old one, whether there any loss of information during reconnection. The latter could be resolved by the [Reliable message delivery](https://github.com/fisherxu/kubeedge/blob/reliab-proposal/docs/proposals/reliable-message-delivery.md).In the proposal, SyncController will periodically compare the saved objects resourceVersion with the objects in K8s, and then trigger the events such as retry and deletion to ensure state consistency.	

But the new cloudcore can not serve the edgecore as well as the old one because the cloudcore is stateful.For example, when it comes to the delivery of configmap, CC need to maintain a LocationCache to record which node the pod (using the configmap) bind to.As shown in the following figure:

If the LocationCache is not synchronized between the old and new CC, Although SyncController trigger the events delivering the configmap, delivery will fail because of unknown of destination.Now the LocationCache is stored in memory and will lost during crash.And I do not know if there is any other data like this. Maybe many.

There are 3 possible ways, and copies are required.

- The data in edgestore is read from the edge whenever connection is established.

In view of frequent connection between cloud and edge, it is not a good idea.

- Stateful data is still stored in memory, using **raft** or other consistency protocol to Synchronize data between copies and elect a leader to provide services.[#1192](https://github.com/kubeedge/kubeedge/issues/1192)
- **Make CloudCore stateless**.Store more stateful data in etcd as CRD like [Reliable message delivery](https://github.com/fisherxu/kubeedge/blob/reliab-proposal/docs/proposals/reliable-message-delivery.md) do. 

I prefer the third way.I want to hear from your opinions before start to figer out the stateful data in CC and write the reconnection code between different cloudcores.The ideal structure is shown below:


