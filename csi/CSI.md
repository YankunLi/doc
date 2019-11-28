# CSI插件实现

## CSI支持的版本信息

### **Kubernetes CSI Spec Compatibility Status**

k8s version| CSI version| CSI release
:-:|:-:|:-:
v1.9|v0.1.0|Alpha
v1.10|v0.2.0|Beta
v1.11|v0.3.0|Beta
v1.13|v0.3.0, v1.0.0|GA

## CSI标准接口

### CSI插件实现必须实现的3部分:  **Identity Service** **Controller Service** **Node Service**.

**Identity Service:** Both the Node Plugin and the Controller Plugin MUST implement this sets of RPCs.

**Controller Service:** The Controller Plugin MUST implement this sets of RPCs.

**Node Service:** The Node Plugin MUST implement this sets of RPCs.

### Identity Service

```golang
service Identity {
    //获取plugin信息
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}
   //获取plugin所能提供的服务
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}
   //探测插件
  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```

### Controller Service

```golang
service Controller {
    //创建volume存储资源,即从总的存储空间中分配指定大小的存储空间
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}
    //删除存储资源,即释放存储空间
  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}
    //发布一个volume,类似公布一个volume资源,并记录
  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}
    //回收一个volume,撤销volume资源的公布,删除该记录
  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}
    //验证volume capability,即验证volume 信息是否合法
  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}
    //列举volume实例
  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}
    //获取存储空间的存储空间情况
  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}
    //获取ControllerService所提供的服务能力,即那些功能
  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}
    //创建快照
  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}
    //删除快照
  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}
    //列举快照
  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}
    //扩展volume的空间大小
  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}
}
```

### Node Service

```golang
service Node {
    //在node节点上挂载该volume,在controller NodePublishVolume成功之后被调用,在node NodePublishVolume之前被执行
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}
    //卸载node上挂载的volume,在controller NodeUnpublishVolume之前调用,在 node NodeUnpublishVolume之后被调用
  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}
    //在node节点上挂载volume,在NodeStageVolume之后被调用
  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}
    //在node节点上卸载volume,在NodeStageVolume之前被调用
  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}
    //获取node上volume 容量使用状态信息
  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}
    //扩展node上的volume
  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}
    //获取plugin node所提供的服务,即提供的功能
  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}
    //获取node上volumes挂载的信息
  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

### CSI RPCServer

* Identity RPCServer
* Controller RPCServer
* Node RPCServer

## CSI Plugin Deployment

* **Controller RPCServer**和**Node RPCServer** 都必须提供**Identify RPCServer**服务;

* **Identify RPCServer** 和 **Controller RPCServer**可以部署在任意节点上,由k8s的支持CSI的外部组件基于Unix Socket 通过RPC协议调用;

* **Identify RPCServer** 和 **Node RPCServer**的部署取决于volume的在哪里使用,即使用的volume的任何node上,都必须部署,kubelet基于Unix Socket 通过RPC协议与node RPCServer交互;

[**Link:** CSI Protocol](https://github.com/container-storage-interface/spec/blob/master/spec.md)

## K8S与插件的交互架构图

---

![K8s <-> CSI plugin](pictures/k8s-csi-logical.png#pic_center "K8s <-> CSI plugin")  

Extentnal Components(Driver Registrar External Provisioner Exteernal Attacher)是实现存储out-tree关键.

* **Driver Registrar** : 注册插件信息到k8s，需要请求 CSI 插件的 Identity 服务来获取插件信息; 
 注: 驱动注册服务服务建议使用**node-driver-registrar**(k8s release 1.13), cluster-driver-registrar和driver-registrar已经淘汰.
* **External Provisioner** : 的生命周期管理Watch  APIServer 的 PVC 对象，PVC 被创建时，就会调用 CSI Controller 的 CreateVolume 方法，创建对应 PV;
* **External Attacher** : Watch APIServer 的 VolumeAttachment 对象，就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段

## 认证授权

1. 管理存储资源的权限,例如获取存储资源信息/创建volume/删除volume/清空volume等.

2. 使用已分配的volume,挂载,卸载,数据读写等.

## 插件的部署 TODO

## TODO

1. CSI插件RPC服务及插件协议接口:
    * Identity Service:
      * GetPluginInfo
      * GetPluginCapabilities
      * Probe
    * Controller Service:
      * CreateVolume
      * DeleteVolume
      * ControllerPublishVolume
      * ControllerUnpublishVolume
      * ValidateVolumeCapabilities
      * ListVolumes
      * GetCapacity
      * ControllerGetCapabilities
      * ControllerExpandVolume
    * Node Service
      * NodeStageVolume
      * NodeUnstageVolume
      * NodePublishVolume
      * NodeUnpublishVolume
      * NodeGetVolumeStats
      * NodeExpandVolume
      * NodeGetCapabilities
      * NodeGetInfo

2. 提供文件系统管理服务:
   * 创建文件系统空间,即创建目录;
   * 删除文件系统空间,即删除目录;
   * 列举已分配的共享空间;
   * 清除文件系统空间中的文件,即清空指定目录下的内容;
   * 给文件系统空间产生有权限的key, 挂着该文件系统的用户才有该目录的权限;

3. 认证授权,k8s要持有共享文件系统的管理权限,可行使管理员的对共享共享文件系统管理权限:
   * k8s需要有共享文件系统的创建/删除/清理/赋权限的功能.
   * 插件所在的node上要持有能挂着共享文件系统的权限(RO/RW).

4. 挂载客户端需要支持, fuse挂载时灵活指定参数选项:
   * 例如只读挂载(RO),该功能通过挂载共享目录时,传参灵活支持.

5. 需要支持quota
   * 资源创建时指定了带创建资源的大小;

**注:** k8s需要能够持有共享存储的管理权限,及共享文件系统的使用权限.

## CSI Design

[CSI Desgine](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)

## CSI Documentation

[CSI documentation](https://kubernetes-csi.github.io/docs/introduction.html)

## External Components

* external-attacher
* external-provisioner
* node-driver-registrar
* cluster-driver-registrar
* external-resizer
* external-snapshotter
* livenessprobe

## TODO2

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
