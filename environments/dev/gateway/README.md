# Development Gateway

Development deployment configuration for the platform gateway and ingress-facing services.

kubectl -n llm-serving create secret generic model-gateway-secrets \
  --from-literal=GATEWAY_API_KEY='YOUR_GATEWAY_API_KEY' --context k3d-fde-gpu-v2