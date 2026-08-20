# Helm Charts

Each helm chart that you can use has the following keys and you need to set them. The `cluster.provider` is used as a key for the various cloud features enabled. Also you only need to specify one cloud provider, **not** both if deploying to cloud. As of writing this doc, AWS and Azure are fully supported.

```bash
# dict with what features and the env you're deploying to
cluster:
  provider: local  # choose from: local | aws | azure
  cloudNativeServices: false # set to true to use Cloud Native Services (SecretsManager and IAM for AWS; KeyVault & Managed Identities for Azure)

aws:
  # the aws cli commands uses the name 'besu-sa' so only change this if you altered the name
  serviceAccountName: besu-sa
  # the region you are deploying to
  region: ap-southeast-2

azure:
  serviceAccountName: besu-sa
  # the clientId of the user assigned managed identity (used by workload identity and CSI secret provider)
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

You are encouraged to pull these charts apart and experiment with options to learn how things work.

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
helm install monitoring prometheus-community/kube-prometheus-stack --version 88.5.2 --namespace=besu --create-namespace --wait
```
### _For Besu:_

```bash
# Following step creates config maps required for subsequent steps. Allow Kubernetes jobs to complete the
# process before proceeding to next steps. 
# It has been noted that genesis tool take a little time to complete. Wait for the pod to complete its work
helm install genesis ./charts/besu-genesis --namespace besu --create-namespace --values ./values/genesis.yml
helm install genesis ./charts/besu-genesis --namespace besu --create-namespace --values ./values/genesis-predefined.yml

# bootnodes - optional but recommended
helm install bootnode-1 ./charts/besu-node --namespace besu --values ./values/bootnode.yml
helm install bootnode-2 ./charts/besu-node --namespace besu --values ./values/bootnode.yml

# !! IMPORTANT !! - If you use bootnodes, please set `pvtNetworkFlags.bootnodes: true` in the override yaml files
# for validator.yml, txnode.yml, reader.yml
# All 4 validators must be started for the blocks to be produced.
helm install validator-1 ./charts/besu-node --namespace besu --values ./values/validator.yml
helm install validator-2 ./charts/besu-node --namespace besu --values ./values/validator.yml
helm install validator-3 ./charts/besu-node --namespace besu --values ./values/validator.yml
helm install validator-4 ./charts/besu-node --namespace besu --values ./values/validator.yml

# Logs when not all 4 validators not yet started
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

# spin up a besu rpc node
helm install rpc-1 ./charts/besu-node --namespace besu --values ./values/rpc.yml
```
