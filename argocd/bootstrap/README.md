# Argo CD Bootstrap

Bootstrap manifests and configuration for installing and initializing Argo CD.


We added root-app.yaml to implement the Argo CD App-of-Apps pattern.

Without root-app.yaml: You must manually apply changes to vllm-dev.yaml using kubectl apply.

With root-app.yaml: Argo CD automatically detects changes to Application definitions in your ai-gitops repository and updates them.
How it works
ai-gitops (GitHub)
       ↓
root-app.yaml (Root Application)
       ↓
vllm-dev.yaml (Child Application)
       ↓
Helm Chart
       ↓
vLLM (GPU Cluster)


Key difference:
- Root Application: Manages Argo CD Applications.
- Child Application: Manages Kubernetes workloads (Deployment, Service).

FDE interview point: The App-of-Apps pattern enables declarative, centralized management of multiple applications and environments through GitOps.

You can later add rag-service, model-gateway, and redis as child Applications without manually registering each one.



-------------------

Yes, correct. If auto-sync is enabled in vllm-dev, Argo CD automatically deploys Helm chart or values changes to your GPU cluster.

Why do we need root-app.yaml then?
- vllm-dev auto-sync: Synchronizes Kubernetes workloads (Deployments, Services).
- ai-platform-root auto-sync: Synchronizes Argo CD Application configurations (vllm-dev.yaml, future applications).
Example: If you change vllm-dev.yaml to point to a different Git repository or Helm chart revision, vllm-dev cannot automatically pick up that configuration change. The root Application can.

Bottom line: Root App is optional for a single application, but valuable when managing multiple applications through GitOps.