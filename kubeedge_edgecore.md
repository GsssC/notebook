### edged
```go
// podReady holds the initPodReady flag and its lock
type podReady struct {
	// initPodReady is flag to check Pod ready status
	initPodReady bool
	// podReadyLock is used to guard initPodReady flag
	podReadyLock sync.RWMutex
}
//Define edged
type edged struct {
	//dns config
	dnsConfigurer             *kubedns.Configurer
	context                   *context.Context
	hostname                  string
	namespace                 string
	nodeName                  string
	interfaceName             string
	uid                       types.UID
	nodeStatusUpdateFrequency time.Duration
	registrationCompleted     bool
	containerManager          cm.ContainerManager
	containerRuntimeName      string
	// container runtime
	containerRuntime   kubecontainer.Runtime
	podCache           kubecontainer.Cache
	os                 kubecontainer.OSInterface
	runtimeService     internalapi.RuntimeService
	podManager         podmanager.Manager
	pleg               pleg.PodLifecycleEventGenerator
	statusManager      kubestatus.Manager
	kubeClient         clientset.Interface
	probeManager       prober.Manager
	livenessManager    proberesults.Manager
	server             *server.Server
	podAdditionQueue   *workqueue.Type
	podAdditionBackoff *flowcontrol.Backoff
	podDeletionQueue   *workqueue.Type
	podDeletionBackoff *flowcontrol.Backoff
	imageGCManager     images.ImageGCManager
	containerGCManager kubecontainer.ContainerGC
	metaClient         client.CoreInterface
	volumePluginMgr    *volume.VolumePluginMgr
	mounter            mount.Interface
	volumeManager      volumemanager.VolumeManager
	rootDirectory      string
	gpuPluginEnabled   bool
	version            string
	// podReady is structure with initPodReady flag and its lock
	podReady
	// cache for secret
	secretStore    cache.Store
	configMapStore cache.Store
	workQueue      queue.WorkQueue
	clcm           clcm.ContainerLifecycleManager
	//edged cgroup driver for container runtime
	cgroupDriver string
	//clusterDns dns
	clusterDNS []net.IP
	// edge node IP
	nodeIP net.IP

	// pluginmanager runs a set of asynchronous loops that figure out which
	// plugins need to be registered/unregistered based on this node and makes it so.
	pluginManager pluginmanager.PluginManager

	recorder recordtools.EventRecorder
}

//Config defines configuration details
type Config struct {
	nodeName                  string
	nodeNamespace             string
	interfaceName             string
	memoryCapacity            int64
	nodeStatusUpdateInterval  time.Duration
	devicePluginEnabled       bool
	gpuPluginEnabled          bool
	imageGCHighThreshold      int
	imageGCLowThreshold       int
	imagePullProgressDeadline int
	MaxPerPodContainerCount   int
	DockerAddress             string
	runtimeType               string
	remoteRuntimeEndpoint     string
	remoteImageEndpoint       string
	RuntimeRequestTimeout     metav1.Duration
	PodSandboxImage           string
	cgroupDriver              string
	nodeIP                    string
	clusterDNS                string
	clusterDomain             string
}
```
### container
```go
type Version interface {
	// Compare compares two versions of the runtime. On success it returns -1
	// if the version is less than the other, 1 if it is greater than the other,
	// or 0 if they are equal.
	Compare(other string) (int, error)
	// String returns a string that represents the version.
	String() string
}

// ImageSpec is an internal representation of an image.  Currently, it wraps the
// value of a Container's Image field, but in the future it will include more detailed
// information about the different image types.
type ImageSpec struct {
	Image string
}

// ImageStats contains statistics about all the images currently available.
type ImageStats struct {
	// Total amount of storage consumed by existing images.
	TotalStorageBytes uint64
}

// Runtime interface defines the interfaces that should be implemented
// by a container runtime.
// Thread safety is required from implementations of this interface.
// 容器运行时接口
type Runtime interface {
	// Type returns the type of the container runtime.
	Type() string

	// Version returns the version information of the container runtime.
	Version() (Version, error)

	// APIVersion returns the cached API version information of the container
	// runtime. Implementation is expected to update this cache periodically.
	// This may be different from the runtime engine's version.
	// TODO(random-liu): We should fold this into Version()
	APIVersion() (Version, error)
	// Status returns the status of the runtime. An error is returned if the Status
	// function itself fails, nil otherwise.
	// 运行时状态返回
	Status() (*RuntimeStatus, error)
	// GetPods returns a list of containers grouped by pods. The boolean parameter
	// specifies whether the runtime returns all containers including those already
	// exited and dead containers (used for garbage collection).
	// 返回pods(true:返回包括死掉的，false:返回不包括死掉的)
	GetPods(all bool) ([]*Pod, error)
	// GarbageCollect removes dead containers using the specified container gc policy
	// If allSourcesReady is not true, it means that kubelet doesn't have the
	// complete list of pods from all avialble sources (e.g., apiserver, http,
	// file). In this case, garbage collector should refrain itself from aggressive
	// behavior such as removing all containers of unrecognized pods (yet).
	// If evictNonDeletedPods is set to true, containers and sandboxes belonging to pods
	// that are terminated, but not deleted will be evicted.  Otherwise, only deleted pods will be GC'd.
	// TODO: Revisit this method and make it cleaner.
	GarbageCollect(gcPolicy ContainerGCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error
	// Syncs the running pod into the desired pod.
	SyncPod(pod *v1.Pod, podStatus *PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) PodSyncResult
	// KillPod kills all the containers of a pod. Pod may be nil, running pod must not be.
	// TODO(random-liu): Return PodSyncResult in KillPod.
	// gracePeriodOverride if specified allows the caller to override the pod default grace period.
	// only hard kill paths are allowed to specify a gracePeriodOverride in the kubelet in order to not corrupt user data.
	// it is useful when doing SIGKILL for hard eviction scenarios, or max grace period during soft eviction scenarios.
	KillPod(pod *v1.Pod, runningPod Pod, gracePeriodOverride *int64) error
	// GetPodStatus retrieves the status of the pod, including the
	// information of all containers in the pod that are visible in Runtime.
	GetPodStatus(uid types.UID, name, namespace string) (*PodStatus, error)
	// TODO(vmarmol): Unify pod and containerID args.
	// GetContainerLogs returns logs of a specific container. By
	// default, it returns a snapshot of the container log. Set 'follow' to true to
	// stream the log. Set 'follow' to false and specify the number of lines (e.g.
	// "100" or "all") to tail the log.
	GetContainerLogs(ctx context.Context, pod *v1.Pod, containerID ContainerID, logOptions *v1.PodLogOptions, stdout, stderr io.Writer) (err error)
	// Delete a container. If the container is still running, an error is returned.
	DeleteContainer(containerID ContainerID) error
	// ImageService provides methods to image-related methods.
	ImageService
	// UpdatePodCIDR sends a new podCIDR to the runtime.
	// This method just proxies a new runtimeConfig with the updated
	// CIDR value down to the runtime shim.
	UpdatePodCIDR(podCIDR string) error
}

// StreamingRuntime is the interface implemented by runtimes that handle the serving of the
// streaming calls (exec/attach/port-forward) themselves. In this case, Kubelet should redirect to
// the runtime server.
type StreamingRuntime interface {
	GetExec(id ContainerID, cmd []string, stdin, stdout, stderr, tty bool) (*url.URL, error)
	GetAttach(id ContainerID, stdin, stdout, stderr, tty bool) (*url.URL, error)
	GetPortForward(podName, podNamespace string, podUID types.UID, ports []int32) (*url.URL, error)
}

type ImageService interface {
	// PullImage pulls an image from the network to local storage using the supplied
	// secrets if necessary. It returns a reference (digest or ID) to the pulled image.
	PullImage(image ImageSpec, pullSecrets []v1.Secret, podSandboxConfig *runtimeapi.PodSandboxConfig) (string, error)
	// GetImageRef gets the reference (digest or ID) of the image which has already been in
	// the local storage. It returns ("", nil) if the image isn't in the local storage.
	GetImageRef(image ImageSpec) (string, error)
	// Gets all images currently on the machine.
	ListImages() ([]Image, error)
	// Removes the specified image.
	RemoveImage(image ImageSpec) error
	// Returns Image statistics.
	ImageStats() (*ImageStats, error)
}

type ContainerAttacher interface {
	AttachContainer(id ContainerID, stdin io.Reader, stdout, stderr io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize) (err error)
}

type ContainerCommandRunner interface {
	// RunInContainer synchronously executes the command in the container, and returns the output.
	// If the command completes with a non-0 exit code, a k8s.io/utils/exec.ExitError will be returned.
	RunInContainer(id ContainerID, cmd []string, timeout time.Duration) ([]byte, error)
}

// Pod is a group of containers.
type Pod struct {
	// The ID of the pod, which can be used to retrieve a particular pod
	// from the pod list returned by GetPods().
	ID types.UID
	// The name and namespace of the pod, which is readable by human.
	Name      string
	Namespace string
	// List of containers that belongs to this pod. It may contain only
	// running containers, or mixed with dead ones (when GetPods(true)).
	Containers []*Container
	// List of sandboxes associated with this pod. The sandboxes are converted
	// to Container temporariliy to avoid substantial changes to other
	// components. This is only populated by kuberuntime.
	// TODO: use the runtimeApi.PodSandbox type directly.
	Sandboxes []*Container
}

// PodPair contains both runtime#Pod and api#Pod
type PodPair struct {
	// APIPod is the v1.Pod
	APIPod *v1.Pod
	// RunningPod is the pod defined in pkg/kubelet/container/runtime#Pod
	RunningPod *Pod
}

// ContainerID is a type that identifies a container.
type ContainerID struct {
	// The type of the container runtime. e.g. 'docker'.
	Type string
	// The identification of the container, this is comsumable by
	// the underlying container runtime. (Note that the container
	// runtime interface still takes the whole struct as input).
	ID string
}

func BuildContainerID(typ, ID string) ContainerID {
	return ContainerID{Type: typ, ID: ID}
}

// Convenience method for creating a ContainerID from an ID string.
func ParseContainerID(containerID string) ContainerID {
	var id ContainerID
	if err := id.ParseString(containerID); err != nil {
		klog.Error(err)
	}
	return id
}

func (c *ContainerID) ParseString(data string) error {
	// Trim the quotes and split the type and ID.
	parts := strings.Split(strings.Trim(data, "\""), "://")
	if len(parts) != 2 {
		return fmt.Errorf("invalid container ID: %q", data)
	}
	c.Type, c.ID = parts[0], parts[1]
	return nil
}

func (c *ContainerID) String() string {
	return fmt.Sprintf("%s://%s", c.Type, c.ID)
}

func (c *ContainerID) IsEmpty() bool {
	return *c == ContainerID{}
}

func (c *ContainerID) MarshalJSON() ([]byte, error) {
	return []byte(fmt.Sprintf("%q", c.String())), nil
}

func (c *ContainerID) UnmarshalJSON(data []byte) error {
	return c.ParseString(string(data))
}

// DockerID is an ID of docker container. It is a type to make it clear when we're working with docker container Ids
type DockerID string

func (id DockerID) ContainerID() ContainerID {
	return ContainerID{
		Type: "docker",
		ID:   string(id),
	}
}

type ContainerState string

const (
	ContainerStateCreated ContainerState = "created"
	ContainerStateRunning ContainerState = "running"
	ContainerStateExited  ContainerState = "exited"
	// This unknown encompasses all the states that we currently don't care.
	ContainerStateUnknown ContainerState = "unknown"
)

// Container provides the runtime information for a container, such as ID, hash,
// state of the container.
type Container struct {
	// The ID of the container, used by the container runtime to identify
	// a container.
	ID ContainerID
	// The name of the container, which should be the same as specified by
	// v1.Container.
	Name string
	// The image name of the container, this also includes the tag of the image,
	// the expected form is "NAME:TAG".
	Image string
	// The id of the image used by the container.
	ImageID string
	// Hash of the container, used for comparison. Optional for containers
	// not managed by kubelet.
	Hash uint64
	// State is the state of the container.
	State ContainerState
}

// PodStatus represents the status of the pod and its containers.
// v1.PodStatus can be derived from examining PodStatus and v1.Pod.
type PodStatus struct {
	// ID of the pod.
	ID types.UID
	// Name of the pod.
	Name string
	// Namespace of the pod.
	Namespace string
	// IP of the pod.
	IP string
	// Status of containers in the pod.
	ContainerStatuses []*ContainerStatus
	// Status of the pod sandbox.
	// Only for kuberuntime now, other runtime may keep it nil.
	SandboxStatuses []*runtimeapi.PodSandboxStatus
}

// ContainerStatus represents the status of a container.
type ContainerStatus struct {
	// ID of the container.
	ID ContainerID
	// Name of the container.
	Name string
	// Status of the container.
	State ContainerState
	// Creation time of the container.
	CreatedAt time.Time
	// Start time of the container.
	StartedAt time.Time
	// Finish time of the container.
	FinishedAt time.Time
	// Exit code of the container.
	ExitCode int
	// Name of the image, this also includes the tag of the image,
	// the expected form is "NAME:TAG".
	Image string
	// ID of the image.
	ImageID string
	// Hash of the container, used for comparison.
	Hash uint64
	// Number of times that the container has been restarted.
	RestartCount int
	// A string explains why container is in such a status.
	Reason string
	// Message written by the container before exiting (stored in
	// TerminationMessagePath).
	Message string
}

// FindContainerStatusByName returns container status in the pod status with the given name.
// When there are multiple containers' statuses with the same name, the first match will be returned.
func (podStatus *PodStatus) FindContainerStatusByName(containerName string) *ContainerStatus {
	for _, containerStatus := range podStatus.ContainerStatuses {
		if containerStatus.Name == containerName {
			return containerStatus
		}
	}
	return nil
}

// Basic information about a container image.
type Image struct {
	// ID of the image.
	ID string
	// Other names by which this image is known.
	RepoTags []string
	// Digests by which this image is known.
	RepoDigests []string
	// The size of the image in bytes.
	Size int64
}

type EnvVar struct {
	Name  string
	Value string
}

type Annotation struct {
	Name  string
	Value string
}

type Mount struct {
	// Name of the volume mount.
	// TODO(yifan): Remove this field, as this is not representing the unique name of the mount,
	// but the volume name only.
	Name string
	// Path of the mount within the container.
	ContainerPath string
	// Path of the mount on the host.
	HostPath string
	// Whether the mount is read-only.
	ReadOnly bool
	// Whether the mount needs SELinux relabeling
	SELinuxRelabel bool
	// Requested propagation mode
	Propagation runtimeapi.MountPropagation
}

type PortMapping struct {
	// Name of the port mapping
	Name string
	// Protocol of the port mapping.
	Protocol v1.Protocol
	// The port number within the container.
	ContainerPort int
	// The port number on the host.
	HostPort int
	// The host IP.
	HostIP string
}

type DeviceInfo struct {
	// Path on host for mapping
	PathOnHost string
	// Path in Container to map
	PathInContainer string
	// Cgroup permissions
	Permissions string
}

// RunContainerOptions specify the options which are necessary for running containers
type RunContainerOptions struct {
	// The environment variables list.
	Envs []EnvVar
	// The mounts for the containers.
	Mounts []Mount
	// The host devices mapped into the containers.
	Devices []DeviceInfo
	// The port mappings for the containers.
	PortMappings []PortMapping
	// The annotations for the container
	// These annotations are generated by other components (i.e.,
	// not users). Currently, only device plugins populate the annotations.
	Annotations []Annotation
	// If the container has specified the TerminationMessagePath, then
	// this directory will be used to create and mount the log file to
	// container.TerminationMessagePath
	PodContainerDir string
	// The type of container rootfs
	ReadOnly bool
	// hostname for pod containers
	Hostname string
	// EnableHostUserNamespace sets userns=host when users request host namespaces (pid, ipc, net),
	// are using non-namespaced capabilities (mknod, sys_time, sys_module), the pod contains a privileged container,
	// or using host path volumes.
	// This should only be enabled when the container runtime is performing user remapping AND if the
	// experimental behavior is desired.
	EnableHostUserNamespace bool
}

// VolumeInfo contains information about the volume.
type VolumeInfo struct {
	// Mounter is the volume's mounter
	Mounter volume.Mounter
	// BlockVolumeMapper is the Block volume's mapper
	BlockVolumeMapper volume.BlockVolumeMapper
	// SELinuxLabeled indicates whether this volume has had the
	// pod's SELinux label applied to it or not
	SELinuxLabeled bool
	// Whether the volume permission is set to read-only or not
	// This value is passed from volume.spec
	ReadOnly bool
	// Inner volume spec name, which is the PV name if used, otherwise
	// it is the same as the outer volume spec name.
	InnerVolumeSpecName string
}

type VolumeMap map[string]VolumeInfo

// RuntimeConditionType is the types of required runtime conditions.
type RuntimeConditionType string

```
