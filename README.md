# Helm Charts

Each helm chart that you can use has the following keys and you need to set them. The `cluster.provider` is used as a key for the various cloud features enabled. Also you only need to specify one cloud provider, **not** both if deploying to cloud. As of writing this doc, AWS and Azure are fully supported.

```bash
# dict with what features and the env you're deploying to
cluster:
  provider: local  # choose from: local | aws | azure
  cloudNativeServices: false # set to true to use Cloud Native Services (SecretsManager and IAM for AWS; KeyVault & Managed Identities for Azure), synced via External Secrets Operator

aws:
  # the aws cli commands uses the name 'besu-sa' so only change this if you altered the name
  serviceAccountName: besu-sa
  # the region you are deploying to
  region: ap-southeast-2

azure:
  serviceAccountName: besu-sa
  # the clientId of the user assigned managed identity (used by workload identity for the hooks and the External Secrets SecretStore)
  identityClientId: azure-clientId
  keyvaultName: azure-keyvault
  # the tenant ID of the key vault
  tenantId: azure-tenantId
  # the subscription ID to use - this needs to be set explicitly when using multi tenancy
  subscriptionId: azure-subscriptionId

```

Setting the `cluster.cloudNativeServices: true` will:

- Keys are stored in KeyVault or Secrets Manager
- We make use of Managed Identities or IAMs for access
- Keys are synced into each node's `<fullname>-keys` Secret by [External Secrets Operator](https://external-secrets.io) (ESO), which must be installed on the cluster (see below)

You are encouraged to pull these charts apart and experiment with options to learn how things work.

## Cluster prerequisites: CSI drivers and External Secrets

The `besu-node` chart creates a StorageClass for each node, and, when `cloudNativeServices: true`, an ExternalSecret and SecretStore. **It does not install the drivers or operators behind them.** If they're missing, `helm install` fails with `no matches for kind "ExternalSecret"`, or it succeeds but the pod stays `Pending` / `ContainerCreating`. The PVC then shows `waiting for a volume to be created, either by external provisioner "<name>" or manually created`, or the pod reports `secret "<fullname>-keys" not found`. Install the drivers for your settings **once per cluster, before installing any besu-node release**.

### Which driver you need

| `cluster.provider` | Setting | Driver / provisioner | Installed by default? |
|---|---|---|---|
| `azure` | `storage.azure.diskType: azure-file` | `file.csi.azure.com` | Yes (AKS built-in) |
| `azure` | `premium-ssd-v1` / `premium-ssd-v2` / `ultra-disk` | `disk.csi.azure.com` | Yes (AKS built-in). Ultra Disk also needs `--enable-ultra-ssd` on the nodepool |
| `azure` | `local-nvme` | `localdisk.csi.acstor.io` (Azure Container Storage v2) | **No** |
| `aws` | `storage.aws.diskType: gp3` / `io2` | `ebs.csi.aws.com` (Amazon EBS CSI driver) | **No** |
| `aws` | `local-nvme` | none (static `hostPath` PV) | n/a. The instance store must be formatted and mounted on the node (see below) |
| `azure` / `aws` | `cluster.cloudNativeServices: true` | External Secrets Operator (`external-secrets.io/v1` CRDs) | **No** |
| `local` | — | the cluster's default provisioner (e.g. minikube `hostpath`) | Yes |

Check what's already installed:

```bash
kubectl get csidrivers
kubectl get crd externalsecrets.external-secrets.io   # External Secrets Operator
```

### Azure

**Local NVMe (`local-nvme`): Azure Container Storage v2.** The nodepool must use a VM size with local NVMe (e.g. Lsv3 / Lasv4) and a managed OS disk, so an ephemeral OS disk doesn't take one of the NVMe devices. You need Azure CLI 2.83.0 or later.

```bash
az extension add --upgrade --name k8s-extension
az aks update -n <cluster> -g <resource-group> --enable-azure-container-storage

# verify (the NVMe driver pods appear once the first besu-node StorageClass exists)
kubectl get deploy -n kube-system | grep acstor
kubectl get pod -n kube-system | grep acstor
```

If Azure Container Storage v1 is installed, remove it first. Data on local NVMe is node-local: it's lost if the node is deleted, reimaged or deallocated. See [Install Azure Container Storage](https://learn.microsoft.com/en-us/azure/storage/container-storage/install-container-storage-aks) and [Use with local NVMe](https://learn.microsoft.com/en-us/azure/storage/container-storage/use-container-storage-with-local-disk).

### AWS

**EBS volumes (`gp3` / `io2`): Amazon EBS CSI driver.** EKS doesn't install this driver by default. It needs an IAM role with `AmazonEBSCSIDriverPolicyV2`, and is best installed as the EKS add-on:

```bash
# IAM role for the driver's service account (requires an OIDC provider on the cluster)
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa --namespace kube-system --cluster <cluster> \
  --role-name AmazonEKS_EBS_CSI_DriverRole --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicyV2 \
  --approve

aws eks create-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<account-id>:role/AmazonEKS_EBS_CSI_DriverRole

kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

> **EKS Auto Mode:** Auto Mode has its own built-in EBS provisioner, `ebs.csi.eks.amazonaws.com`, and doesn't use `ebs.csi.aws.com`. The chart's StorageClass uses `ebs.csi.aws.com`, so on an Auto Mode cluster you need either the add-on above or a StorageClass changed to the Auto Mode provisioner.

See [Use Kubernetes volume storage with Amazon EBS](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

**Local NVMe (`local-nvme`).** There's no driver to install. The chart creates a static `hostPath` PV at `storage.aws.nvmePath/<release>`, so the instance store (i3 / i3en / i4i etc.) must already be formatted and mounted at `storage.aws.nvmePath` when the node boots. On AL2023 EKS nodes you can have `nodeadm` do this by setting `spec.instance.localStorage.strategy` in the node's `NodeConfig` (`RAID0` stripes all instance-store disks; the default mount path is `/mnt/k8s-disks/`). Run `findmnt` on a node to confirm the path, then set `storage.aws.nvmePath` to match. Pin pods to that nodegroup with `affinity`, because the data lives on the node.

### External Secrets Operator (`cloudNativeServices: true`)

With `cloudNativeServices: true`, the pre-install hook (or the genesis job, for validators) writes each node's keys to Key Vault / Secrets Manager. The chart then creates an `ExternalSecret` that syncs three of them into the `<fullname>-keys` Secret the pod mounts at `/keys`:

| Vault secret | Key in `<fullname>-keys` |
|---|---|
| `<fullname>-nodekey` | `nodekey` |
| `<fullname>-nodekeypub` | `nodekey.pub` |
| `<fullname>-enode` | `enode` |

Anything else in the vault for that node is ignored. All three must exist, because ESO won't sync the Secret if any referenced key is missing.

**Install ESO** (once per cluster, provider-agnostic). Tested against chart `2.9.0`; if you manage it with Argo CD, use `chart: external-secrets`, `repoURL: https://charts.external-secrets.io`, `targetRevision: 2.9.0`:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --version 2.9.0 \
  --namespace external-secrets --create-namespace --wait

kubectl get pods -n external-secrets
kubectl get crd externalsecrets.external-secrets.io
```

**Authentication.** By default the chart creates a namespaced `SecretStore` per release. It authenticates as the same service account the besu pods and hooks already use (`azure.serviceAccountName` / `aws.serviceAccountName`):

| Provider | SecretStore auth | Service account / identity requirements |
|---|---|---|
| Azure | `azurekv`, `authType: WorkloadIdentity`, vault `https://<azure.keyvaultName>.vault.azure.net` | AKS with `--enable-oidc-issuer --enable-workload-identity`. SA annotated `azure.workload.identity/client-id: <identityClientId>`, with a federated credential on that managed identity. The identity needs to read secrets (e.g. `Key Vault Secrets User`), plus write access for the hooks (e.g. `Key Vault Secrets Officer`) |
| AWS | `aws` / `SecretsManager` in `aws.region`, `auth.jwt.serviceAccountRef` (IRSA) | SA annotated `eks.amazonaws.com/role-arn: <role>`. The role needs `secretsmanager:GetSecretValue` and `secretsmanager:DescribeSecret`, plus create/delete for the hooks |

To use a store you already manage instead, for example a `ClusterSecretStore` that uses EKS Pod Identity or the ESO controller's own identity, point the chart at it:

```yaml
externalSecrets:
  secretStoreRef:
    name: my-cluster-store
    kind: ClusterSecretStore
```

**Verify** after installing a node:

```bash
kubectl get secretstore,externalsecret -n <namespace>     # STATUS Valid / SecretSynced
kubectl get secret <fullname>-keys -n <namespace>
kubectl describe externalsecret <fullname>-keys -n <namespace>   # auth / missing-key errors show here
```

## Local Development:

Minikube defaults to 2 CPU's and 2GB of memory, unless configured otherwise. We recommend you starting with at least 16GB, depending on the amount of nodes you are spinning up - the recommended requirements for each besu node are 4GB

```bash
minikube start --memory 16384 --cpus 2
# or with RBAC
minikube start --memory 16384 --cpus 2 --extra-config=apiserver.Authorization.Mode=RBAC
# optionally start the dashboard
minikube dashboard &
```

Verify kubectl is connected to Minikube with: (please use the latest version of kubectl)

```bash
$ kubectl version
Client Version: v1.36.0
Kustomize Version: v5.8.1
Server Version: v1.35.1
```

## Usage

### _Spin up prometheus-stack for metrics: (Optional but recommended)_

**NOTE:** this uses charts from prometheus-community - please configure this as per your requirements and policies

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
# NOTE: please refer to values/monitoring.yml to configure the alerts per your requirements ie slack, email etc
helm install monitoring prometheus-community/kube-prometheus-stack --version 88.5.2 --namespace=monitoring --create-namespace --values ./values/monitoring.yml --wait
```

Locally you can open grafana on port 3000
```bash
kubectl port-forward -n monitoring $(kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o name) 3000:3000
```
### _For Besu:_

```bash
# The following step creates config maps required for subsequent steps. Allow Kubernetes jobs to complete the process before proceeding to next steps. 
# It has been noted that genesis tool take a little time to complete. Wait for the pod to complete its work.
helm install genesis ./charts/besu-genesis --namespace besu --create-namespace --values ./values/genesis-predefined.yml
# Now that the genesis in created with keys, create the validators
helm install validator-1 ./charts/besu-node --namespace besu --values ./values/validator.yml
helm install validator-2 ./charts/besu-node --namespace besu --values ./values/validator.yml
helm install validator-3 ./charts/besu-node --namespace besu --values ./values/validator.yml
helm install validator-4 ./charts/besu-node --namespace besu --values ./values/validator.yml
# Now an RPC node
helm install rpc-1 ./charts/besu-node --namespace besu --values ./values/rpc.yml

# Logs when all 4 validators have not yet started
2024-09-12 05:05:47.566+00:00 | EthScheduler-Timer-0 | INFO  | FullSyncTargetManager | Unable to find sync target. Currently checking 3 peers for usefulness
2024-09-12 05:05:52.567+00:00 | EthScheduler-Timer-0 | INFO  | FullSyncTargetManager | Unable to find sync target. Currently checking 3 peers for usefulness
2024-09-12 05:05:57.045+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | RoundChangeManager | BFT round summary (quorum = 3)
2024-09-12 05:05:57.045+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | RoundChangeManager | Address: 0x4d27048b7f2bd1ca29d96a0c28e881179ee4d6bc  Round: 2 (Local node)
2024-09-12 05:05:57.046+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | RoundChangeManager | Address: 0x5caaded557eaa2b403147a09debe48fa73477b72  Round: 2
2024

# Logs when all 4 validators started and connected
2024-09-12 05:12:04.715+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | QbftRound | Importing proposed block to chain. round=ConsensusRoundIdentifier{Sequence=1, Round=4}, hash=0x916003b5f8468e09416c9d35d803225b56d693a1b6b88401c2262b3aa5588a8a
2024-09-12 05:12:04.732+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | QbftBesuControllerBuilder | Imported #1 / 0 tx / 0 pending / 0 (0.0%) gas / (0x916003b5f8468e09416c9d35d803225b56d693a1b6b88401c2262b3aa5588a8a)
2024-09-12 05:12:06.101+00:00 | EthScheduler-Timer-0 | INFO  | FullSyncTargetManager | Unable to find sync target. Currently checking 5 peers for usefulness
2024-09-12 05:12:09.037+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | QbftBesuControllerBuilder | Produced #2 / 0 tx / 0 pending / 0 (0.0%) gas / (0xb04b1dfeca0605fd4e2cb2bb91085d3dc14d0c89da9a5b238204892201c571a4)
2024-09-12 05:12:14.044+00:00 | BftProcessorExecutor-QBFT-0 | INFO  | QbftBesuControllerBuilder | Imported #3 / 0 tx / 0 pending / 0 (0.0%) gas / (0xb86626f296bdcf08d5e794cc8153d716ce0ac740a11109472655cff8abfe183a)
```
