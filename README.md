## Gloo Gateway + Agentgateway Installation

1. Export the Gloo Gateway license key and agentgateway license key
```
export GLOO_GATEWAY_LICENSE_KEY=
export AGENTGATEWAY_LICENSE_KEY=
```

2. Install the Kubernetes Gateway CRDs
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

3. Install Gloo Gateway CRDs
```
helm upgrade -i --create-namespace --namespace gloo-system --version 2.0.1 gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds
```

4. Install Gloo Gateway with agentgateway enabled

```
helm upgrade -i -n gloo-system gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
   --set gateway.aiExtension.enabled=true \
   --set agentgateway.enabled=true \
   --set licensing.glooGatewayLicenseKey=$GLOO_GATEWAY_LICENSE_KEY \
   --set licensing.agentgatewayLicenseKey=$AGENTGATEWAY_LICENSE_KEY \
   --set --version=2.0.1
```

## Set Up A Gateway For Testing

1. Export your Anthropic API key
```
export ANTHROPIC_API_KEY=
```

2. Create a Gateway using the agentgateway enterprise gatewayclass to ensure the proper data plane/proxy is configured for AI traffic
```
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: agentgateway
  namespace: gloo-system
  labels:
    app: agentgateway
spec:
  gatewayClassName: agentgateway-enterprise
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

3. Add the ALB IP into an environment variable
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n gloo-system agentgateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

4. Create a Kubernetes Secret for the Anthropic key
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: anthropic-secret
  namespace: gloo-system
  labels:
    app: agentgateway
type: Opaque
stringData:
  Authorization: $CLAUDE_API_KEY
EOF
```

5. Create a backend for Gloo Gateway to know where to route to. In this case, it's a Claude Model
```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata:
  labels:
    app: agentgateway
  name: anthropic
  namespace: gloo-system
spec:
  type: AI
  ai:
    llm:
        anthropic:
          authToken:
            kind: SecretRef
            secretRef:
              name: anthropic-secret
          model: "claude-3-5-haiku-latest"
EOF
```

6. Ensure the Backend was accepted.
```
kubectl get backend -n gloo-system
```

7. Create an HTTP route for to route traffic to the LLM
```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: claude
  namespace: gloo-system
  labels:
    app: agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: gloo-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anthropic
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /v1/chat/completions
    backendRefs:
    - name: anthropic
      namespace: gloo-system
      group: gateway.kgateway.dev
      kind: Backend
EOF
```

8. Test the Model connectivity
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -H content-type:application/json -H x-api-key:$ANTHROPIC_API_KEY -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "Write me a paragraph containing the best way to think about Istio Ambient Mesh"
    }
  ]
}' | jq
```

## MCP Server Security Connectivity

1. Create a traffic policy that realize on a JWT key for authentication
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: jwt
  namespace: gloo-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway
  glooJWT:
    beforeExtAuth:
      providers:
        selfminted:
          issuer: solo.io
          jwks:
            local:
              key: '{"keys":[{"kty":"RSA","kid":"solo-public-key-001","use":"sig","alg":"RS256","n":"AOfIaJMUm7564sWWNHaXt_hS8H0O1Ew59-nRqruMQosfQqa7tWne5lL3m9sMAkfa3Twx0LMN_7QqRDoztvV3Wa_JwbMzb9afWE-IfKIuDqkvog6s-xGIFNhtDGBTuL8YAQYtwCF7l49SMv-GqyLe-nO9yJW-6wIGoOqImZrCxjxXFzF6mTMOBpIODFj0LUZ54QQuDcD1Nue2LMLsUvGa7V1ZHsYuGvUqzvXFBXMmMS2OzGir9ckpUhrUeHDCGFpEM4IQnu-9U8TbAJxKE5Zp8Nikefr2ISIG2Hk1K2rBAc_HwoPeWAcAWUAR5tWHAxx-UXClSZQ9TMFK850gQGenUp8","e":"AQAB"}]}'
EOF
```

2. Try curling the Gateway
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -H content-type:application/json -H x-api-key:$ANTHROPIC_API_KEY -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "Write me a paragraph containing the best way to think about Istio Ambient Mesh"
    }
  ]
}' | jq
```

You should see a failure in the agentgateway Pod logs
```
kubectl logs deploy/agentgateway -n gloo-system
```

3. Open MCP Inspector:
```

```

4. Add the following for the **Bearer Token** for authentication:
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6ImJvYiIsInRlYW0iOiJvcHMiLCJleHAiOjIwNzQyNzQ5NTQsImxsbXMiOnsibWlzdHJhbGFpIjpbIm1pc3RyYWwtbGFyZ2UtbGF0ZXN0Il19fQ.GF_uyLpZSTT1DIvJeO_eish1WDjMaS4BQSifGQhqPRLjzu3nXtPkaBRjceAmJi9gKZYAzkT25MIrT42ZIe3bHilrd1yqittTPWrrM4sWDDeldnGsfU07DWJHyboNapYR-KZGImSmOYshJlzm1tT_Bjt3-RK3OBzYi90_wl0dyAl9D7wwDCzOD4MRGFpoMrws_OgVrcZQKcadvIsH8figPwN4mK1U_1mxuL08RWTu92xBcezEO4CdBaFTUbkYN66Y2vKSTyPCxg3fLtg1mvlzU1-Wgm2xZIiPiarQHt6Uq7v9ftgzwdUBQM1AYLvUVhCN6XkkR9OU3p0OXiqEDjAxcg
```

## Agentgateway Metrics

1. Add the Prometheus Helm Chart

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

2. Install Kube-Prometheus
```
helm install kube-prometheus -n monitoring prometheus-community/kube-prometheus-stack --create-namespace
```

3. Deploy a Pod Monitor

The `PodMonitor` selector matches `gateway.networking.k8s.io/gateway-class-name: agentgateway`, which will match any Gateway that uses the agentgateway GatewayClass
```
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: agentgateway
  namespace: gloo-system
  labels:
    app: agentgateway
    release: kube-prometheus
spec:
  selector:
    matchLabels:
      gateway.networking.k8s.io/gateway-class-name: agentgateway
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
EOF
```

To look at the metrics in Prometheus
```
kubectl --namespace monitoring port-forward svc/kube-prometheus-kube-prome-prometheus 9090
```

To build a dashboard with the metrics
```
kubectl --namespace monitoring port-forward svc/kube-prometheus-grafana 3000:80
```

To log into the Grafana UI:

1. Username: admin
2. Password: prom-operator

4. Generate some LLM traffic
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -H content-type:application/json -H x-api-key:$ANTHROPIC_API_KEY -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "Write me a paragraph containing the best way to think about Istio Ambient Mesh"
    }
  ]
}' | jq
```

## Agentgateway Traces

This section includes the creation and checking of traces within agentgateway

### Tracing Setup

1. Install the OTel collector
```
helm upgrade --install opentelemetry-collector-traces opentelemetry-collector \
--repo https://open-telemetry.github.io/opentelemetry-helm-charts \
--version 0.127.2 \
--set mode=deployment \
--set image.repository="otel/opentelemetry-collector-contrib" \
--set command.name="otelcol-contrib" \
--namespace=telemetry \
--create-namespace \
-f -<<EOF
config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
  exporters:
    otlp/tempo:
      endpoint: http://tempo.telemetry.svc.cluster.local:4317
      tls:
        insecure: true
    debug:
      verbosity: detailed
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [debug, otlp/tempo]
EOF
```

2. Ensure that its running
```
kubectl get pods -n telemetry
```

3. The configmap below specifies the tracing endpoint and a few extra metrics that you want to be collected within your trace
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-gateway-config
  namespace: gloo-system
data:
  config.yaml: |-
    config:
      tracing:
        otlpEndpoint: http://opentelemetry-collector-traces.telemetry.svc.cluster.local:4317
        otlpProtocol: grpc
        randomSampling: true
        fields:
          add:
            gen_ai.operation.name: '"chat"'
            gen_ai.system: "llm.provider"
            gen_ai.request.model: "llm.requestModel"
            gen_ai.response.model: "llm.responseModel"
            gen_ai.usage.completion_tokens: "llm.outputTokens"
            gen_ai.usage.prompt_tokens: "llm.inputTokens"
EOF
```

4. Set up a parameters object to reference the configmap above
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooGatewayParameters
metadata:
  name: tracing
  namespace: gloo-system
spec:
  kube:
    agentgateway:
      customConfigMapName: agent-gateway-config
EOF
```

### Checking Traces

To check the traces:

1. Get the logs for agentgateway
```
kubectl logs deploy/agentgateway -n gloo-system
```

2. Look for the `trace.id=`

3. Look at the logs for the OTel tracing collector and you'll be able to see the trace ID
```
kubectl logs deploy/opentelemetry-collector-traces -n telemetry
```

## Bonus: Rate Limiting

1. Create env variable for Anthropic key

```
export ANTHROPIC_API_KEY=
```

2. Create a secret to store the Claude API key
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: anthropic-secret
  namespace: kgateway-system
  labels:
    app: agentgateway
type: Opaque
stringData:
  Authorization: $ANTHROPIC_API_KEY
EOF
```

3. Create a Gateway for Anthropic

A `Gateway` resource is used to trigger kgateway to deploy agentgateway data plane Pods

The Agentgateway data plane Pod is the Pod that gets created when a Gateway object is created in a Kubernetes environment where Agentgateway is deployed as the Gateway API implementation.
```
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: agentgateway
  namespace: gloo-system
  labels:
    app: agentgateway
spec:
  gatewayClassName: agentgateway-enterprise
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

4. Create a `Backend` object 

A Backend resource to define a backing destination that you want Gloo Gateway to route to. In this case, it's Claude.
```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata:
  labels:
    app: agentgateway
  name: anthropic
  namespace: gloo-system
spec:
  type: AI
  ai:
    llm:
        anthropic:
          authToken:
            kind: SecretRef
            secretRef:
              name: anthropic-secret
          model: "claude-3-5-haiku-latest"
EOF
```

5. Ensure everything is running as expected
```
kubectl get backend -n gloo-system
```

6. Create a rate limit rule that targets the `HTTPRoute` you just created
```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: anthropic-ratelimit
  namespace: gloo-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: claude
  rateLimit:
    local:
      tokenBucket:
        maxTokens: 1
        tokensPerFill: 1
        fillInterval: 100s
EOF
```

7. Apply the Route so you can reach the LLM
```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: claude
  namespace: gloo-system
  labels:
    app: agentgateway
spec:
  parentRefs:
    - name: agentgateway
      namespace: gloo-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anthropic
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /v1/chat/completions
    backendRefs:
    - name: anthropic
      namespace: gloo-system
      group: gateway.kgateway.dev
      kind: Backend
EOF
```

8. Capture the LB IP of the service. This will be used later to send a request to the LLM.
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n gloo-system agentgateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

9. Test the LLM connectivity
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -v \ -H content-type:application/json -H x-api-key:$ANTHROPIC_API_KEY -H "anthropic-version: 2023-06-01" -d '{
  "model": "claude-sonnet-4-5",
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "Write me a paragraph containing the best way to think about Istio Ambient Mesh"
    }
  ]
}' | jq
```

10. Run the `curl` again

You'll see a `curl` error that looks something like this:

```
< x-ratelimit-limit: 1
< x-ratelimit-remaining: 0
< x-ratelimit-reset: 76
< content-length: 19
< date: Tue, 18 Nov 2025 15:35:45 GMT
```

And if you check the agentgateway Pod logs, you'll see the rate limit error.

```
kubectl logs -n gloo-system agentgateway-pod-name --tail=50 | grep -i "request\|error\|anthropic"
```

```
2025-10-20T16:08:59.886579Z     info    request gateway=gloo-system/agentgateway listener=http route=gloo-system/claude src.addr=10.142.0.25:42187 http.method=POST http.host=34.148.15.158 http.path=/anthropic http.version=HTTP/1.1 http.status=429 error="rate limit exceeded" duration=0ms
```