# ha 
```go
// downstream.go 116
nodes := dc.lc.ConfigMapNodes(configMap.Namespace,configMap.Name)

// util.go 26
node/nodeid/namespaces/resourceType/resourceID

//default.go 
Context: &EdgeControllerContext{
	SendModule:     metaconfig.ModuleNameCloudHub,
	ReceiveModule:  metaconfig.ModuleNameEdgeController,
	ResponseModule: metaconfig.ModuleNameCloudHub,	
}



```
# beehive微服务框架
- 基本概念
	详见[浙大SEL](http://www.sel.zju.edu.cn/?p=989)
- KubeEdge消息结构
	```go
	// Message struct
	type Message struct {
		Header  MessageHeader `json:"header"`
		Router  MessageRoute  `json:"route,omitempty"`
		Content interface{}   `json:"content"`
	}

	// MessageRoute contains structure of message
	type MessageRoute struct {
		// where the message come from
		Source string `json:"source,omitempty"`
		// where the message will broadcasted to
		Group string `json:"group,omitempty"`

		// what's the operation on resource
		Operation string `json:"operation,omitempty"`
		// what's the resource want to operate
		Resource string `json:"resource,omitempty"`
	}
	const (
	InsertOperation        = "insert"
	DeleteOperation        = "delete"
	QueryOperation         = "query"
	UpdateOperation        = "update"
	ResponseOperation      = "response"
	ResponseErrorOperation = "error"

	ResourceTypePod        = "pod"
	ResourceTypeConfigmap  = "configmap"
	ResourceTypeSecret     = "secret"
	ResourceTypeNode       = "node"
	ResourceTypePodlist    = "podlist"
	ResourceTypePodStatus  = "podstatus"
	ResourceTypeNodeStatus = "nodestatus"
	)
	// MessageHeader defines message header details
	type MessageHeader struct {
		// the message uuid
		ID string `json:"msg_id"`
		// the response message parentid must be same with message received
		// please use NewRespByMessage to new response message
		ParentID string `json:"parent_msg_id,omitempty"`
		// the time of creating
		Timestamp int64 `json:"timestamp"`
		// specific resource version for the message, if any.
		// it's currently backed by resource version of the k8s object saved in the Content field.
		// kubeedge leverages the concept of message resource version to achieve reliable transmission.
		ResourceVersion string `json:"resourceversion,omitempty"`
		// the flag will be set in sendsync
		Sync bool `json:"sync,omitempty"`
	}
	type MessageContext interface {
    // async mode
    Send(module string, message model.Message)
    Receive(module string) (model.Message, error)
    // sync mode
    SendSync(module string, message model.Message, timeout time.Duration) (model.Message, error)
    SendResp(message model.Message)
    // group broadcast
    SendToGroup(moduleType string, message model.Message)
    SendToGroupSync(moduleType string, message model.Message, timeout time.Duration) error
	}
	```
	消息结构体中，使用Router字段记录消息路由，`Message.Router.Source`记录消息由哪个模块发送，`Message.Router.Group`记录消息发向哪个模块组。但事实上，由于beehive发送消息的函数在发送消息时需要指定目的模块(dest module)和目的组(dest group)`Send(module string, message model.Message)`,所以哪怕`Message.Router`中记录错误，消息也可以向外发送，此字段服务于各消息解析模块。一个不恰当的比喻，Message作为一份信，邮差beehive并不会根据信封上的信息邮寄信件，在信件交到beehive手上时便得对其说明。  
	同时，beehive框架利用了golang的channel机制实现数据传输，这个机制只对单个程序内的模块有效，在云边传输中，数据传输是跨程序、跨宿主机的，所以各消息解析模块需要许多额外的工作。
- 消息operation
	消息操作类型共有5类，其中`insert`,`delete`主要为云端产生，自动同步云端数据库etcd至边端数据库edge-store，`pod及各k8s API的update `，`query`为边端发现缺少的`k8s API resource`，主动向云端请求拉取，如pod启动所需要的`configmap`。`response`和`query`成对，`query`到达云端，云端向`API-Server`拉取query的资源后使用`response`向边端回应。
- 消息Resource
	`Message.Resource`在消息路由中意义重大，`cloudcore`和`edgecore`是一对多的关系，仔细思考后发现，目前的消息路由机制无法将消息发送至指定的边缘节点。而这需要依赖`Message.Resource`记录的消息所属节点信息。其他需要对消息解析进行的模块如`cloudhub`根据此字段使用相应的`wss connectiong`将消息发送至对应节点。
	```go
	func BuildResource(nodeID, namespace, resourceType, resourceID string) (resource string, err error) {
	if namespace == "" || resourceType == "" {
		if !config.Config.EdgeSiteEnable && nodeID == "" {
			err = fmt.Errorf("required parameter are not set (node id, namespace or resource type)")
		} else {
			err = fmt.Errorf("required parameter are not set (namespace or resource type)")
		}
		return
	}

	resource = fmt.Sprintf("%s%s%s%s%s%s%s", controller.ResourceNode, constants.ResourceSep, nodeID, constants.ResourceSep, namespace, constants.ResourceSep, resourceType)
	if config.Config.EdgeSiteEnable {
		resource = fmt.Sprintf("%s%s%s", namespace, constants.ResourceSep, resourceType)
	}
	if resourceID != "" {
		resource += fmt.Sprintf("%s%s", constants.ResourceSep, resourceID)
	}
	return
	}	
	```
- 消息依赖
	K8S中`API Resource`具有依赖关系，如`Pod`和`ConfigMap`，一个`ConfigMap`被`Pod`所引用才有意义，`Pod`资源中指明了其需要部署在哪个节点`Pod.Spec.NodeName`，在消息`BuildResource`时可以根据利用字段。如果边端在启动`Pod`时，发现没有依赖的`ConfigMap`，会向云端发送`query`请求，这个请求中也包含了节点信息，所以在首次下发`ConfigMap`消息`BuildResource`时，可以根据此信息。但在后续`ConfigMap`在云端中被更新，由于没有类似字段，更新消息构建时无法指定发向哪个节点，所以每次下发`Pod`时需要记录其被部署到哪个节点，及其依赖的`ConfigMap`，这种记录表在源码中称为`LocationCache`。
	```go
	type LocationCache struct {
	// EdgeNodes is a map, key is nodeName, value is Status
	EdgeNodes sync.Map
	// configMapNode is a map, key is namespace/configMapName, value is nodeName
	configMapNode sync.Map
	// secretNode is a map, key is namespace/secretName, value is nodeName
	secretNode sync.Map
	// services is a map, key is namespace/serviceName, value is v1.Service
	services sync.Map
	// endpoints is a map, key is namespace/endpointsName, value is v1.endpoints
	endpoints sync.Map
	// servicePods is a map, key is namespace/serviceName, value is []v1.Pod
	servicePods sync.Map
	}
	```
# 以Pod下发为例
- DownStreamController
	![message][1]	
		
[1]: images/Message.png
