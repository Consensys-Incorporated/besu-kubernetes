# besu-node Helm Chart

Deploys a single Hyperledger Besu node as a StatefulSet. Intended to be installed multiple times — once per validator and once per RPC node.

## Storage Configuration

Storage is controlled by `storage.azure.diskType` (Azure) or `storage.aws.diskType` (AWS). Set the value in your node's values file.

### Azure

| `diskType` | Provisioner | SKU | Caching | Use case |
|---|---|---|---|---|
| `azure-file` | `file.csi.azure.com` | Standard_LRS | — | Default, dev/test |
| `premium-ssd-v1` | `disk.csi.azure.com` | Premium_LRS | None / ReadOnly / ReadWrite | P-series disks (P40/P50 by PVC size); use when caching is required |
| `premium-ssd-v2` | `disk.csi.azure.com` | PremiumV2_LRS | None (forced) | Higher IOPS/throughput ceiling than v1; no caching support |
| `ultra-disk` | `disk.csi.azure.com` | UltraSSD_LRS | None (forced) | Lowest latency, highest IOPS; requires `--enable-ultra-ssd` on nodepool |
| `local-nvme` | `localdisk.csi.acstor.io` (Azure Container Storage v2) | — | — | Raw NVMe speed, striped across all local NVMe disks; requires Lsv3/Lasv4 VM and the Azure Container Storage extension |

**Key:** Premium SSD v2 and Ultra Disk do not support host caching. If caching is needed, use `premium-ssd-v1` with `cachingMode: ReadOnly`.

```yaml
storage:
  sizeLimit: "20Gi"
  pvcSizeLimit: "20Gi"
  azure:
    diskType: premium-ssd-v1   # azure-file | premium-ssd-v1 | premium-ssd-v2 | ultra-disk | local-nvme
    cachingMode: ReadOnly       # premium-ssd-v1 only — None | ReadOnly | ReadWrite
```

#### Azure prerequisites by type

| Type | Requirement |
|---|---|
| `ultra-disk` | Nodepool created with `--enable-ultra-ssd`; node must be in a zone that supports Ultra Disk |
| `local-nvme` | Nodepool VM size with local NVMe (e.g. Lsv3/Lasv4); Azure Container Storage v2 enabled on the cluster (`az aks update --enable-azure-container-storage`). Data is node-local and lost if the node is deleted/deallocated |
| `premium-ssd-v2` | Node zone must support PremiumV2_LRS |

#### How `local-nvme` works on Azure

The chart creates a StorageClass with provisioner `localdisk.csi.acstor.io` and `volumeBindingMode: WaitForFirstConsumer`, and adds the `localdisk.csi.acstor.io/accept-ephemeral-storage: "true"` annotation to the StatefulSet's PVC template. The driver then:

- stripes the volume across every local NVMe disk on the VM (no manual formatting or mounting);
- creates the volume on the node the pod is scheduled to, and pins the PV to that node, so a restarted pod always comes back to its data.

Because the data lives on the VM, it is lost if the node is deleted, reimaged or deallocated. The pod then stays `Pending` until its PVC is deleted, after which it gets a fresh volume and resyncs.

---

### AWS

| `diskType` | Provisioner | Type | IOPS | Throughput | Use case |
|---|---|---|---|---|---|
| `gp3` | `ebs.csi.aws.com` | gp3 | Up to 16,000 (set via `iops`) | Up to 1,000 MB/s (set via `throughput`) | Default, cost-effective baseline |
| `io2` | `ebs.csi.aws.com` | io2 | Up to 256,000 on Nitro (set via `iops`) | — | High-throughput workloads; io2 Block Express on supported instances |
| `local-nvme` | static hostPath PV | — | Max NVMe | Max NVMe | Raw instance store speed; i3/i3en/i4i family, device pre-mounted |

```yaml
storage:
  sizeLimit: "20Gi"
  pvcSizeLimit: "20Gi"
  aws:
    diskType: gp3         # gp3 | io2 | local-nvme
    iops: "6000"          # gp3 (up to 16,000) or io2 (up to 256,000)
    throughput: "250"     # gp3 only, MB/s (up to 1,000)
    nvmePath: /mnt/nvme   # local-nvme only
```

#### AWS prerequisites by type

| Type | Requirement |
|---|---|
| `io2` | Nitro-based instance for io2 Block Express (e.g. m5, c5, r5 or newer) |
| `local-nvme` | i3, i3en, or i4i instance family; instance store formatted and mounted at `nvmePath` |

---

## Quick reference — predefined values files

| File | Storage type | Provider |
|---|---|---|
| `validator-predefined-ultra.yml` | Ultra Disk | Azure |
| `validator-predefined-nvme.yml` | Local NVMe | Azure |
| `validator-predefined-premium-cached.yml` | Premium SSD v1, ReadOnly cache | Azure |
| `rpc-predefined-ultra.yml` | Ultra Disk | Azure |
| `rpc-predefined-nvme.yml` | Local NVMe | Azure |
| `rpc-predefined-premium-cached.yml` | Premium SSD v1, ReadOnly cache | Azure |
