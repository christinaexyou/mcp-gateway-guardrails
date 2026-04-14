# Guardrailing the MCP Gateway
## Deployment Topology

```
Namespace: gateway-system
  ├── Gateway/mcp-gateway (Istio-managed Envoy proxy)
  └── Deployment/body-based-router

Namespace: istio-system
  └── EnvoyFilter (injects MCP router ext_proc into Gateway)

Namespace: mcp-system
  ├── Deployment/mcp-broker-router (Broker :8080, Router ext_proc :50051)
  ├── Deployment/mcp-controller
  ├── ConfigMap/mcp-gateway-config
  └── Secret/mcp-aggregated-credentials

Namespace: trustyai-guardrails
  └── NemoGuardrails CR → Operator creates:
      ├── Deployment/<cr-name>         (NeMo server on :8000)
      ├── Service/<cr-name>            (ClusterIP, port 80 → 8000)
      ├── ConfigMap/<cr-name>-ca-bundle
      └── Route/<cr-name>              (OpenShift only)
```


## Deploy the MCP Gateway

Run the following script to:
* Install the required operators (Service Mesh, Connectivity Link, and MCP Gateway Controller)
* Install MCPGateway CRDs (MCPGatewayExtensions, MCPServerRegistrations)
* Deploy an MCPGateway instance and create an external route

```
curl -sL https://raw.githubusercontent.com/Kuadrant/mcp-gateway/main/config/openshift/deploy_openshift.sh | bash
```

The MCP Gateway is accesible via:
```
echo https://$(oc get routes -n gateway-system -o jsonpath='{ .items[0].spec.host }')/mcp
```

## Deploy the NeMo Guardrails Server

Create a namespace for the TrustyAI Guardrails Operator:

```
oc new-project trustyai-guardrails-operator-system
```

Run the following command to install the operator and CRDs:
```
oc apply -f https://raw.githubusercontent.com/trustyai-explainability/trustyai-guardrails-operator/main/release/trustyai_guardrails_bundle.yaml \
  -n trustyai-guardrails-operator-system

oc wait --for=condition=ready pod \
  -l control-plane=controller-manager \
  -n trustyai-guardrails-operator-system \
  --timeout=300s
```

Deploy a NeMo Guardrails instance which is configured with the following rails:
* Personally Identifable Information (PII) detector
* Prompt Injection/Jailbreak detector
* Hate and Profanity (HAP) detector

```
oc new-project trustyai-guardrails || oc project trustyai-guardrails

oc apply -f https://raw.githubusercontent.com/trustyai-explainability/trustyai-guardrails-operator/main/config/samples/nemoguardrails_sample.yaml \
  -n trustyai-guardrails

oc wait --for=condition=ready pod \
  -l app=example-nemoguardrails \
  -n trustyai-guardrails \
  --timeout=300s
```

## Install Payload

Install the ExternalModel CRD

```
oc apply -f https://raw.githubusercontent.com/opendatahub-io/models-as-a-service/refs/heads/main/deployment/base/maas-controller/crd/bases/maas.opendatahub.io_externalmodels.yaml
```

Set `GATEWAY_NAME`,`GATEWAY_NAMESPACE`, and `NEMO_GUARDRAILS_URL` environment variables

```
export GATEWAY_NAME=<>
export GATEWAY_NAMESPACE=<>
export NEMO_GUARDRAILS_URL=https://$(oc get route example-nemoguardrails -o jsonpath='{.spec.host}')
```

Replace the environment variables in `values.yaml`:

```
envsubst < manifests/values.yaml > /tmp/values.yaml
```

Install `payload-processing` via Helm:

```
helm install payload-processing ./deploy/payload-processing \
  --namespace ${GATEWAY_NAMESPACE} \
  --dependency-update \
  --set upstreamBbr.provider.istio.envoyFilter.anchorSubFilter=extensions.istio.io/wasmplugin/${GATEWAY_NAMESPACE}.kuadrant-${GATEWAY_NAME} \
  -f /tmp/values.yaml
```

Send a test request through the MCP Gateway:

```
export MODEL_NAME=<model-name>
export GATEWAY_URL=http://${GATEWAY_NAME}-istio.${GATEWAY_NAMESPACE}.svc:80

curl -si --max-time 30 \
  ${GATEWAY_URL}/${MODEL_NAME}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Connection: close" \
  -d '{
    "model": "'"${MODEL_NAME}"'",
    "messages": [{"role": "user", "content": "hello from '"${MODEL_NAME}"'"}]
  }'
```


## References
1. https://github.com/Kuadrant/mcp-gateway/tree/main/config/openshift
2. https://github.com/trustyai-explainability/trustyai-guardrails-operator/blob/main/docs/nemo_guardrails_quickstart.md
3. https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/85
