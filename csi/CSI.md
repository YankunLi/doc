# CSI插件实现

## 待实现组件:

CSI插件实现必须实现的3部分:  **Identity Service** **Controller Service** **Node Service**.

**Identity Service:** Both the Node Plugin and the Controller Plugin MUST implement this sets of RPCs.
**Controller Service:** The Controller Plugin MUST implement this sets of RPCs.
**Node Service:** The Node Plugin MUST implement this sets of RPCs.

### Identity Service

```golang
service Identity {
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}

  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}

  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```

### Controller Service

```golang
service Controller {
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}

  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}

  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}

  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}

  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}

  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}

  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}

  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}

  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}

  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}

  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}

  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}
}
```

### Node Service

```golang
service Node {
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}

  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}

  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}

  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}

  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}


  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}


  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}

  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

## K8S与插件的交互架构图

---

![K8s <-> CSI plugin](pictures/k8s-csi-logical.png#pic_center "K8s <-> CSI plugin")  

Extentnal Components(Driver Registrar External Provisioner Exteernal Attacher)是实现存储out-tree关键.

* **Driver Registrar** : 注册插件信息到k8s，需要请求 CSI 插件的 Identity 服务来获取插件信息; 
 注: 驱动注册服务服务建议使用**node-driver-registrar**(k8s release 1.13), cluster-driver-registrar和driver-registrar已经淘汰.
* **External Provisioner** : 的生命周期管理Watch  APIServer 的 PVC 对象，PVC 被创建时，就会调用 CSI Controller 的 CreateVolume 方法，创建对应 PV;
* **External Attacher** : Watch APIServer 的 VolumeAttachment 对象，就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段

## 认证 TODO

k8s 用户操作管理wfs空间,需要wfs分配权限赋予k8s用户,实现权限打通.

## 插件的部署 TODO

##  TODO

1. 提供wfs管理服务
2. 权限打通(k8s)

## TODO

### IdentityServer

---

```golang
// DefaultIdentityServer stores driver object
type DefaultIdentityServer struct {
    Driver *CSIDriver
}
```

1. GetPluginInfo

```golang
// GetPluginInfo returns plugin information
func (ids *DefaultIdentityServer) GetPluginInfo(ctx context.Context, req *csi.GetPluginInfoRequest) (*csi.GetPluginInfoResponse, error) {
    klog.V(5).Infof(util.Log(ctx, "Using default GetPluginInfo"))

    if ids.Driver.name == "" {
        return nil, status.Error(codes.Unavailable, "Driver name not configured")
    }   

    if ids.Driver.version == "" {
        return nil, status.Error(codes.Unavailable, "Driver is missing version")
    }   

    return &csi.GetPluginInfoResponse{
        Name:          ids.Driver.name,
        VendorVersion: ids.Driver.version,
    }, nil 
}
```

2. Probe

```golang
// Probe returns empty response
func (ids *DefaultIdentityServer) Probe(ctx context.Context, req *csi.ProbeRequest) (*csi.ProbeResponse, error) {
    return &csi.ProbeResponse{}, nil 
}
```

3. GetPluginCapabilities

```golang
// GetPluginCapabilities returns plugin capabilities
func (ids *DefaultIdentityServer) GetPluginCapabilities(ctx context.Context, req *csi.GetPluginCapabilitiesRequest) (*csi.GetPluginCapabilitiesResponse, error) {
    klog.V(5).Infof(util.Log(ctx, "Using default capabilities"))
    return &csi.GetPluginCapabilitiesResponse{
        Capabilities: []*csi.PluginCapability{
            {   
                Type: &csi.PluginCapability_Service_{
                    Service: &csi.PluginCapability_Service{
                        Type: csi.PluginCapability_Service_CONTROLLER_SERVICE,
                    },  
                },  
            },  
        },  
    }, nil 
}

```

### ControllerServer

---

```golang
// DefaultControllerServer points to default driver
type DefaultControllerServer struct {
    Driver *CSIDriver
}
```

### 待实现的方法(具体协议)

1. GetPluginCapabilities

```golang
// ControllerPublishVolume publish volume on node
func (cs *DefaultControllerServer) ControllerPublishVolume(ctx context.Context, req *csi.ControllerPublishVolumeRequest) (*csi.ControllerPublishVolumeResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

2. ControllerUnpublishVolume

```golang
// ControllerUnpublishVolume unpublish on node
func (cs *DefaultControllerServer) ControllerUnpublishVolume(ctx context.Context, req *csi.ControllerUnpublishVolumeRequest) (*csi.ControllerUnpublishVolumeResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

3. ControllerExpandVolume

```golang
// ControllerExpandVolume expand volume
func (cs *DefaultControllerServer) ControllerExpandVolume(ctx context.Context, req *csi.ControllerExpandVolumeRequest) (*csi.ControllerExpandVolumeResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

4. ListVolumes

```golang
// ListVolumes lists volumes
func (cs *DefaultControllerServer) ListVolumes(ctx context.Context, req *csi.ListVolumesRequest) (*csi.ListVolumesResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

5. GetCapacity

```golang
// GetCapacity get volume capacity
func (cs *DefaultControllerServer) GetCapacity(ctx context.Context, req *csi.GetCapacityRequest) (*csi.GetCapacityResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

6. ControllerGetCapabilities

```golang
// ControllerGetCapabilities implements the default GRPC callout.
// Default supports all capabilities
func (cs *DefaultControllerServer) ControllerGetCapabilities(ctx context.Context, req *csi.ControllerGetCapabilitiesRequest) (*csi.ControllerGetCapabilitiesResponse, error) {
    klog.V(5).Infof(util.Log(ctx, "Using default ControllerGetCapabilities"))

    return &csi.ControllerGetCapabilitiesResponse{
        Capabilities: cs.Driver.cap,
    }, nil 
}

```

#### 不支持

---

7. ~~CreateSnapshot~~

```golang
// CreateSnapshot creates snapshot
func (cs *DefaultControllerServer) CreateSnapshot(ctx context.Context, req *csi.CreateSnapshotRequest) (*csi.CreateSnapshotResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

8. ~~DeleteSnapshot~~

```golang
// DeleteSnapshot deletes snapshot
func (cs *DefaultControllerServer) DeleteSnapshot(ctx context.Context, req *csi.DeleteSnapshotRequest) (*csi.DeleteSnapshotResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

9. ~~ListSnapshots~~

```golang
// ListSnapshots lists snapshosts
func (cs *DefaultControllerServer) ListSnapshots(ctx context.Context, req *csi.ListSnapshotsRequest) (*csi.ListSnapshotsResponse, error) {
}
```

### NodeServer

---

```golang
// DefaultNodeServer stores driver object
type DefaultNodeServer struct {
    Driver *CSIDriver
    Type   string
}
```

1. NodeStageVolume

```golang
// NodeStageVolume returns unimplemented response
func (ns *DefaultNodeServer) NodeStageVolume(ctx context.Context, req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

2. NodeUnstageVolume

```golang
// NodeUnstageVolume returns unimplemented response
func (ns *DefaultNodeServer) NodeUnstageVolume(ctx context.Context, req *csi.NodeUnstageVolumeRequest) (*csi.NodeUnstageVolumeResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

3. NodeExpandVolume

```golang
// NodeExpandVolume returns unimplemented response
func (ns *DefaultNodeServer) NodeExpandVolume(ctx context.Context, req *csi.NodeExpandVolumeRequest) (*csi.NodeExpandVolumeResponse, error) {
    return nil, status.Error(codes.Unimplemented, "") 
}
```

4. NodeGetInfo

```golang
// NodeGetInfo returns node ID
func (ns *DefaultNodeServer) NodeGetInfo(ctx context.Context, req *csi.NodeGetInfoRequest) (*csi.NodeGetInfoResponse, error) {
    klog.V(5).Infof(util.Log(ctx, "Using default NodeGetInfo"))

    return &csi.NodeGetInfoResponse{
        NodeId: ns.Driver.nodeID,
    }, nil 
}
```

5. NodeGetCapabilities

```golang
// NodeGetCapabilities returns RPC unknow capability
func (ns *DefaultNodeServer) NodeGetCapabilities(ctx context.Context, req *csi.NodeGetCapabilitiesRequest) (*csi.NodeGetCapabilitiesResponse, error) {
    klog.V(5).Infof(util.Log(ctx, "Using default NodeGetCapabilities"))

    return &csi.NodeGetCapabilitiesResponse{
        Capabilities: []*csi.NodeServiceCapability{
            {   
                Type: &csi.NodeServiceCapability_Rpc{
                    Rpc: &csi.NodeServiceCapability_RPC{
                        Type: csi.NodeServiceCapability_RPC_UNKNOWN,
                    },
                },  
            },  
        },  
    }, nil 
}

// NodeGetVolumeStats returns volume stats
func (ns *DefaultNodeServer) NodeGetVolumeStats(ctx context.Context, req *csi.NodeGetVolumeStatsRequest) (*csi.NodeGetVolumeStatsResponse, error) {

    var err error
    targetPath := req.GetVolumePath()
    if targetPath == "" {
        err = fmt.Errorf("targetpath %v is empty", targetPath)
        return nil, status.Error(codes.InvalidArgument, err.Error())
    }   
    /*  
        volID := req.GetVolumeId()

        TODO: Map the volumeID to the targetpath.

        CephFS:
           we need secret to connect to the ceph cluster to get the volumeID from volume
           Name, however `secret` field/option is not available  in NodeGetVolumeStats spec,
           Below issue covers this request and once its available, we can do the validation
           as per the spec.

           https://github.com/container-storage-interface/spec/issues/371

        RBD:
           Below issue covers this request for RBD and once its available, we can do the validation
           as per the spec.

           https://github.com/ceph/ceph-csi/issues/511

    */

    isMnt, err := util.IsMountPoint(targetPath)

    if err != nil {
        if os.IsNotExist(err) {
            return nil, status.Errorf(codes.InvalidArgument, "targetpath %s doesnot exist", targetPath)
        }   
        return nil, err 
    }   
    if !isMnt {
        return nil, status.Errorf(codes.InvalidArgument, "targetpath %s is not mounted", targetPath)
    }   

    cephMetricsProvider := volume.NewMetricsStatFS(targetPath)
    volMetrics, volMetErr := cephMetricsProvider.GetMetrics()
    if volMetErr != nil {
        return nil, status.Error(codes.Internal, volMetErr.Error())
    }   

    available, ok := (*(volMetrics.Available)).AsInt64()
    if !ok {
        klog.Errorf(util.Log(ctx, "failed to fetch available bytes"))
    }   
    capacity, ok := (*(volMetrics.Capacity)).AsInt64()
    if !ok {
        klog.Errorf(util.Log(ctx, "failed to fetch capacity bytes"))
        return nil, status.Error(codes.Unknown, "failed to fetch capacity bytes")
    }
    used, ok := (*(volMetrics.Used)).AsInt64()
    if !ok {
        klog.Errorf(util.Log(ctx, "failed to fetch used bytes"))
    }
    inodes, ok := (*(volMetrics.Inodes)).AsInt64()
    if !ok {
        klog.Errorf(util.Log(ctx, "failed to fetch available inodes"))
        return nil, status.Error(codes.Unknown, "failed to fetch available inodes")

    }
    inodesFree, ok := (*(volMetrics.InodesFree)).AsInt64()
    if !ok {
        klog.Errorf(util.Log(ctx, "failed to fetch free inodes"))
    }

    inodesUsed, ok := (*(volMetrics.InodesUsed)).AsInt64()
    if !ok {
        klog.Errorf(util.Log(ctx, "failed to fetch used inodes"))
    }
    return &csi.NodeGetVolumeStatsResponse{
        Usage: []*csi.VolumeUsage{
            {
                Available: available,
                Total:     capacity,
                Used:      used,
                Unit:      csipbv1.VolumeUsage_BYTES,
            },
            {
                Available: inodesFree,
                Total:     inodes,
                Used:      inodesUsed,
                Unit:      csipbv1.VolumeUsage_INODES,
            },
        },
    }, nil

}
```