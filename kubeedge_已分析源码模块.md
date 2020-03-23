# 已分析源码模块
## edged(./edge/pkg/edged)
- **Meta Client(./edge/pkg/metamanager/client)**，与Meta Manager通信，云端消息接收，Container等状态上传至Meta Manager
- **Pod Status Management(status)**，底层运行时上传的Pod状态及存储，由Meta Client上传转发至云端，且受PLEG调用
- **Pod Management(podmanager)**，上层下发的Pod状态存储
- **Pod Lifecycle Event Generator(pleg)**，pod生命周期管理，功能由上述几个组件构成
## 非kubeedge原生组件(vendor/k8s.io/kubernetes/pkg/kubelet)
- **Container Runtime Manager(kuberuntime)**,client的封装及cri/req的构造逻辑，及进一步解析res
- **Container Runtime Client(remote)**，构造cri的req,初步解析res
- **Image Garbage Collection(images)**，镜像垃圾自动回收
- **Container Garbage Collection(container)**,容器垃圾自动回收
- **Probe Management(prober)**，探针管理，受PLEG调用，底层调用cri
