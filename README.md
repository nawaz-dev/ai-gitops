# ai-gitops
GitOps for Enterprise AI Platform Engineering

enterprise-ai-gitops/
├── argocd/
│   ├── bootstrap/
│   ├── applications/
│   └── projects/
├── environments/
│   ├── dev/
│   │   ├── vllm/
│   │   └── gateway/
│   ├── staging/
│   └── production/
├── clusters/
│   ├── k3d/
│   └── openshift/
└── policies/

## vLLM on the local GPU cluster

[Application example](applications/vllm/application.yaml) and
[handover guide](applications/vllm/README.md) for the existing Helm-managed vLLM
workload. The chart is pinned to an immutable commit in
`nawaz-dev/ai-platform-engineering`; the first sync is manual.
