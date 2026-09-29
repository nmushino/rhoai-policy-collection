# rhoai-policy-collection

> **Fork notice**: this is a fork of [eformat/rhoai-policy-collection](https://github.com/eformat/rhoai-policy-collection), branch `rhoai-3.5-servicemesh3`. It migrates the RHOAI install from 2.25.11 + classic Service Mesh 2 to **RHOAI 3.5.1 + Service Mesh 3**, because classic Service Mesh 2 (Maistra `ServiceMeshControlPlane`) is incompatible with OCP 4.21 — the SMCP admission webhook rejects `spec.version` even for the values it lists as valid. RHOAI 2.x has no OLM upgrade path to 3.x (no `replaces`/`olm.skipRange` between the channels), so this fork performs a clean uninstall+reinstall onto the 3.5.1 channel instead. See [Changes vs upstream](#changes-vs-upstream) below for the full list.

Showcases OLM v0,1 support for RHOAI Clusters using policy-as-code and Advanced Cluster Manager.

Showcases BYO [ModelCar Catalog](https://github.com/eformat/modelcar-catalog) deployment of popular open-weight models (deepseek-r1-qwen-distillation, granite, granite-vision, llama) using [vLLM](https://github.com/vllm-project/vllm) and [LLama.cpp](https://github.com/ggml-org/llama.cpp) runtimes.

This repo is deployed using [SNO on SPOT](https://github.com/eformat/sno-for-100) in AWS with a g6 NVIDIA instance as an example accelerated infrastructure

> How-to run your GenAI infrastructure - (model as a service, inference as a service) - on the smell of an oily RAG.

## Prerequisite

OpenShift Cluster with cluster-admin access. See [SNO on SPOT](https://github.com/eformat/sno-for-100) using:

```bash
export INSTANCE_TYPE=g6.8xlarge
export ROOT_VOLUME_SIZE=400
export OPENSHIFT_VERSION=stable-4.21
```

## Bootstrap

Installs ArgoCD and ACM

```bash
kustomize build --enable-helm gitops/bootstrap | oc apply -f-
```

Create CR's

```bash
oc apply -f gitops/bootstrap/setup-cr.yaml
```

We keep Auth, PKI, Storage separate for now as these are Infra specific.

Create htpasswd admin user

```bash
./gitops/bootstrap/users.sh
```

Install LE Certs

```bash
./gitops/bootstrap/certficates.sh
```

Install Extra AWS Storage

```bash
./gitops/bootstrap/storage.sh
```

## Setup app-of-apps storage

With only `storage.yaml` in the app-of-apps folder:

```bash
oc apply -f gitops/app-of-apps/sno-app-of-apps.yaml
```

And set SC default

```bash
oc annotate sc/lvms-vgsno storageclass.kubernetes.io/is-default-class=true
oc annotate sc/gp3-csi storageclass.kubernetes.io/is-default-class-
```

This uses the default storage we setup for LVM

## Vault Secrets

Install `vault.yaml` in the app-of-apps folder.

Setup Vault Auth for ArgoCD

```bash
./gitops/bootstrap/vault-setup.sh
```

## Installs Policy Collection for RHOAI

WIP - base `rhoai` DSC currently.

## Changes vs upstream

Compared to `eformat/rhoai-policy-collection@main`, this branch (`rhoai-3.5-servicemesh3`):

- `gitops/applications/rhoai/base/values-v0.yaml`: `rhods-operator` channel/version bumped to `stable-3.5` / `3.5.1`; `servicemeshoperator` (classic v2) disabled; `servicemeshoperator3` (Sail-based, Service Mesh 3) added, channel `stable`, version `3.4.2`.
- `gitops/applications/rhoai/base/dsc-cr.yaml` and `gitops/applications/rhoai/overlay/sno/dsc-cr.yaml`: migrated to the `datasciencecluster.opendatahub.io/v2` API (component names/shape changed in 3.x, e.g. `datasciencepipelines`→`aipipelines`, `trainingoperator`→`trainer`; `codeflare`/`modelmeshserving` removed; `aigateway`/`feastoperator`/`mlflowoperator`/`ogx`/`sparkoperator`/`llamastackoperator`/`mcplifecycleoperator` added and set `Removed`).
  - `kueue` and `trainer` are set to `Removed`: in 3.5.1, `kueue: Managed` is rejected outright (`managementState Managed is not supported, please use Removed or Unmanaged`), and `trainer: Managed` requires the JobSet operator, which this cluster doesn't install. Neither is needed for kserve-based model serving.
- `gitops/app-of-apps/roadshow/rhoai.yaml`: `source.repoURL`/`targetRevision` point at this fork/branch instead of upstream `main`, so the ArgoCD `ApplicationSet` → child `Application` chain pulls the above changes (required because the live cluster's ACM Policies enforce `selfHeal`, which reverts any direct in-cluster patch back to whatever the source repo says).

RHOAI 2.x → 3.x is not an in-place OLM upgrade (no `replaces`/`olm.skipRange` chain exists between the channels). Migrating an existing 2.25.11 cluster onto this branch requires manually deleting the old `DataScienceCluster`, the `rhods-operator` CSV, and the `Subscription` object first so the ACM Policy recreates it fresh against the `stable-3.5` channel.
