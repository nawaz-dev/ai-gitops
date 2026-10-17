# vLLM Argo CD handover

This example lives in `ai-gitops` and renders `helm/vllm` from
`https://github.com/nawaz-dev/ai-platform-engineering.git` at commit
`41f34f1a35f534260e83c743d4ac7b732c6f9e87`. It deliberately supplies no Helm
value overrides. The Application and Helm release name are both `vllm`.
The Application belongs in the `argocd` namespace of the cluster running
Argo CD (`k3d-dev-cluster`); its workload destination is `llm-serving` on
the separately registered GPU cluster `k3d-fde-gpu-v2`.

## Preserved chart behavior

| Area | Pinned chart defaults |
| --- | --- |
| Identity/network | Deployment and Service `vllm`; selector `app: vllm`; ClusterIP TCP 8000 → 8000; ingress disabled |
| Scheduling | One replica; `Recreate`; autoscaling disabled; runtime class `nvidia`; node `k3d-fde-gpu-v2-agent-0` |
| Resources | GPU request/limit both 1; CPU request 1 / limit 4; memory request 3Gi / limit 8Gi |
| Inference | `vllm/vllm-openai:v0.10.2`; `Qwen/Qwen2.5-0.5B-Instruct` served as `qwen-small`; half precision; model length 1024; GPU memory utilization 0.60; max sequences 2 |
| Environment | `enableServiceLinks: false`; `HF_HOME=/models/huggingface`; `NVIDIA_DRIVER_CAPABILITIES=compute,utility` |
| Storage | Existing `vllm-model-cache` mounted read/write at `/models`; `pvc.create=false`; no PVC rendered or managed by this Application |
| WSL2 | Read-only `/usr/lib64/libdxcore.so` node hostPath (`File`) mounted at `/usr/lib/x86_64-linux-gnu/libdxcore.so` |
| Probes | Startup `/health` on `http`, every 10s, failure threshold 90; readiness `/health`, initial delay 0, every 10s, timeout 2s, failure threshold 3, success threshold 1; liveness disabled |

The DXCore mount is specific to this WSL2 lab. The source file must exist inside
the selected k3d node, not just on the Windows/WSL host. Do not reuse this
Application for native Linux GPU nodes without a separately reviewed values
change. The runtime class, NVIDIA device plugin, GPU allocation, and existing
Bound cache PVC must already work; this Application does not install them.

## One writer for the Deployment

After handover, **Argo CD is the only deployment writer** for these resources.

- Tekton may build, test, scan, and push images and propose Git changes. Promotion
  means reviewing a chart/values commit in `ai-platform-engineering`, then updating
  this Application's `targetRevision` through a PR in `ai-gitops`.
- Remove or disable Tekton tasks/triggers that run `kubectl apply`, `set image`,
  `patch`, `scale`, `rollout restart`, or `helm upgrade/rollback/uninstall` against
  this Deployment. Stop or let active PipelineRuns finish before the handover;
  check their actual service accounts, kubeconfigs, and remote-cluster credentials.
- Remove those CI identities' write access to the serving resources, including
  indirect broad RoleBindings/ClusterRoleBindings. Retain only necessary read
  access. Kubernetes RBAC is additive: a narrow role does not cancel broad grants.
- Do not run direct Helm upgrades, rollbacks, or uninstalls after adoption.
  Argo CD uses Helm to render manifests; it does not maintain Helm release history.
  Existing Helm release Secrets can remain as historical records, but Helm status
  is no longer the deployment source of truth. Do not uninstall the old release
  to transfer ownership: that could delete the running Deployment and Service.
- Do not add `ignoreDifferences` to hide Tekton changes. Manual sync only delays
  reconciliation; it does not make concurrent writers safe. `FailOnSharedResource`
  catches ownership by another Argo CD Application, **not Tekton or Helm writes**.

No Tekton definitions were present in the inspected source repositories. These
steps must be checked against the actual pipelines before the first sync.

## Preflight and first sync

Run these commands from your workstation with access to both clusters and an
authenticated Argo CD CLI. No commands in this guide have been run against the
live clusters as part of creating the example.

1. Confirm the destination registration:

   ```sh
   argocd cluster list
   kubectl --context k3d-fde-gpu-v2 -n llm-serving get deploy/vllm svc/vllm pvc/vllm-model-cache
   kubectl --context k3d-fde-gpu-v2 get runtimeclass nvidia
   kubectl --context k3d-fde-gpu-v2 get node k3d-fde-gpu-v2-agent-0 -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
   ```

   `destination.name` must exactly match the Argo CD registered name. Earlier lab
   output used `fde-gpu-v2`; a kubectl context named `k3d-fde-gpu-v2` alone is not
   sufficient. If still registered under the old name, intentionally update that
   registration to `k3d-fde-gpu-v2` after checking any existing Applications that
   reference it. Do not create a duplicate registration blindly. Alternatively,
   adapt this example's destination name to the actual registration. Confirm Argo
   CD can reach the registered API endpoint; do not substitute workstation loopback.
   The `default` AppProject must permit the source repo and target namespace/cluster.

2. Stop other writers as described above and capture the current state locally:

   ```sh
   helm --kube-context k3d-fde-gpu-v2 -n llm-serving get values vllm --all > /tmp/vllm-live-values.yaml
   helm --kube-context k3d-fde-gpu-v2 -n llm-serving get manifest vllm > /tmp/vllm-helm-before.yaml
   kubectl --context k3d-fde-gpu-v2 -n llm-serving get deploy/vllm svc/vllm -o yaml > /tmp/vllm-live-before.yaml
   ```

   Compare live values with the pinned chart defaults. The example preserves the
   committed chart, not unobserved live overrides. If they differ, record and
   review the required overrides before syncing. Keep exports out of Git.

3. Render the exact pin (using a fresh checkout directory):

   ```sh
   git clone https://github.com/nawaz-dev/ai-platform-engineering.git /tmp/vllm-chart-review
   git -C /tmp/vllm-chart-review checkout --detach 41f34f1a35f534260e83c743d4ac7b732c6f9e87
   helm lint /tmp/vllm-chart-review/helm/vllm
   helm template vllm /tmp/vllm-chart-review/helm/vllm --namespace llm-serving > /tmp/vllm-desired.yaml
   kubectl --context k3d-fde-gpu-v2 -n llm-serving diff -f /tmp/vllm-desired.yaml
   ```

   `kubectl diff` exits 1 for differences. Review those differences, especially
   selectors, Service fields, pod specification, and volumes; do not use force or
   replace to bypass an immutable-field error. The expected render contains only
   a Service and Deployment, no PVC. A `Recreate` rollout can interrupt serving.

4. From the `ai-gitops` checkout, create the Application in the management cluster:

   ```sh
   kubectl --context k3d-dev-cluster apply -f applications/vllm/application.yaml
   argocd app get vllm --refresh
   argocd app diff vllm
   ```

   Applying the Application does not sync workloads automatically. Verify the
   source pin, destination, and diff in Argo CD, then explicitly sync without pruning:

   ```sh
   argocd app sync vllm
   argocd app wait vllm --sync --health --timeout 1200
   kubectl --context k3d-fde-gpu-v2 -n llm-serving get deploy/vllm svc/vllm pvc/vllm-model-cache
   kubectl --context k3d-fde-gpu-v2 -n llm-serving port-forward svc/vllm 8000:8000
   ```

   In a second terminal, check `curl -fsS http://127.0.0.1:8000/health` and
   `curl -fsS http://127.0.0.1:8000/v1/models`; expect the served name `qwen-small`.
   If GPU scheduling, cache access, or startup fails, inspect pod events/logs before
   changing the preserved configuration.

## Subsequent changes and rollback

Commit Application changes in this repository and explicitly re-apply this file
to the management cluster, then review/sync the app. This standalone example does
not install a parent Application to watch `ai-gitops`; merging a PR here alone
does not update the live Application. Chart changes require a new reviewed SHA.

After successful adoption and removal of other writers, automated sync/self-heal
can be enabled in a separate change. Keep pruning disabled until resource ownership
has been reviewed. No cascading-deletion finalizer is included. The cache claim
is external and should remain independently managed and backed up.

For rollback after a later promotion, revert `targetRevision` (and any reviewed
overrides) to the last known-good Git configuration, re-apply the Application,
review the diff, and sync. Do not combine this with a Helm rollback. If abandoning
Argo CD ownership entirely, stop its reconciliation and remove the Application
non-cascading before re-enabling another writer; verify serving resources remain.

References: [Argo CD Helm integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/),
[Application specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/),
[sync options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/).
